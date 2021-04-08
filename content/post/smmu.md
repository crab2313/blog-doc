+++
title = "SMMU内核驱动分析"
date = 2020-02-20


tags = ["kernel", "iommu", "arm"]
+++

# IOMMU核心框架层

IOMMU核心框架是管理IOMMU设备的一个通用框架，IOMMU设备通过实现特定的回调函数并将自身注册到IOMMU核心框架中，以此通过IOMMU核心框架提供的API向整个内核提供IOMMU功能。所有的IOMMU设备都嵌入了一个`struct iommu_device`，iommu的核心代码只会操作这个结构体。可以看到，我们唯一需要关心的就是ops，这是iommu驱动注册到core中的回调函数。:

```c
struct iommu_device {
	struct list_head list;
	const struct iommu_ops *ops;
	struct fwnode_handle *fwnode;
	struct device *dev;
};

/**
 * struct iommu_ops - iommu ops and capabilities
 * @capable: check capability
 * @domain_alloc: allocate iommu domain
 * @domain_free: free iommu domain
 * @attach_dev: attach device to an iommu domain
 * @detach_dev: detach device from an iommu domain
 * @map: map a physically contiguous memory region to an iommu domain
 * @unmap: unmap a physically contiguous memory region from an iommu domain
 * @flush_tlb_all: Synchronously flush all hardware TLBs for this domain
 * @tlb_range_add: Add a given iova range to the flush queue for this domain
 * @tlb_sync: Flush all queued ranges from the hardware TLBs and empty flush
 *            queue
 * @iova_to_phys: translate iova to physical address
 * @add_device: add device to iommu grouping
 * @remove_device: remove device from iommu grouping
 * @device_group: find iommu group for a particular device
 * @domain_get_attr: Query domain attributes
 * @domain_set_attr: Change domain attributes
 * @get_resv_regions: Request list of reserved regions for a device
 * @put_resv_regions: Free list of reserved regions for a device
 * @apply_resv_region: Temporary helper call-back for iova reserved ranges
 * @domain_window_enable: Configure and enable a particular window for a domain
 * @domain_window_disable: Disable a particular window for a domain
 * @domain_set_windows: Set the number of windows for a domain
 * @domain_get_windows: Return the number of windows for a domain
 * @of_xlate: add OF master IDs to iommu grouping
 * @pgsize_bitmap: bitmap of all possible supported page sizes
 */
```

iommu中的核心代码在`drivers/iommu/iommu.c`中实现，下面从一些基本的方面分析核心层提供的功能。由于一个运行的系统中只会同时存在几个iommu设备，因此设备管理实现比较简单，是由链表实现的。

```c
static LIST_HEAD(iommu_device_list);
static DEFINE_SPINLOCK(iommu_device_lock);
```

注册和注销设备实质上就是操作这个链表。iommu向外提供的API不多，后面主要分析：

* iommu_map && iommu_unmap
* iommu_domain_alloc && iommu_domain_free
* iommu_attach_device && iommu_detach_device

## iommu_domain_alloc

在我的理解中，domain这个词是从intel的VT-d文档中继承下来的，其他平台有各自的叫法，比如ARM下叫context。一个domain应该是指一个独立的iommu映射上下文。处于同一个domain中的设备使用同一套映射做地址转换（对于mmio来说就是独立的页表）。core层中使用`struct iommu_domain`表示一个domain：

```c
struct iommu_domain {
	unsigned type;
	const struct iommu_ops *ops;
	unsigned long pgsize_bitmap;	/* Bitmap of page sizes in use */
	iommu_fault_handler_t handler;
	void *handler_token;
	struct iommu_domain_geometry geometry;
	void *iova_cookie;
};
```

后面分析各个字段的含义。注释中提到了内核支持的domain类型：

```c
/*
 * This are the possible domain-types
 *
 *	IOMMU_DOMAIN_BLOCKED	- All DMA is blocked, can be used to isolate
 *				  devices
 *	IOMMU_DOMAIN_IDENTITY	- DMA addresses are system physical addresses
 *	IOMMU_DOMAIN_UNMANAGED	- DMA mappings managed by IOMMU-API user, used
 *				  for VMs
 *	IOMMU_DOMAIN_DMA	- Internally used for DMA-API implementations.
 *				  This flag allows IOMMU drivers to implement
 *				  certain optimizations for these domain
 */
```

这个函数仅仅就是调用ops中驱动注册的`domain_alloc`回调函数分配一个iommu_domain，从这里看书每个驱动也是要提供自己的domain类型并把`struct iommu_domain`嵌入进取的。

## iommu_attach_device

```c
int iommu_attach_device(struct iommu_domain *domain, struct device *dev);
```

从函数原型中可以看出该函数的操作对象是一个domain和一个设备，联想函数名称可以认为该函数是将一个设备添加到一个domain中。但事实上还是有些偏差的，该函数实际上将设备所在的Group与该domain绑定。值得一提的是，如果函数发现设备存在的Group中存在多个设备，则不进行绑定操作。总结下来，该操作针对独立设备（即所在Group里只有自己），将设备所在Group与domain进行绑定。

```c
	if (iommu_group_device_count(group) != 1)
		goto out_unlock;

	ret = __iommu_attach_group(domain, group);
```

`__iommu_attach_group`遍历Group中所有的设备，并调用`__iommu_attach_device`。该函数首先通过`domain->ops`中的is_attach_deffered检查是否延后进行attach操作。然后调用ops中的attach_dev函数将设备绑定到该domain中去。这里需要注意区分Group中default_domain和domain的概念：domain指group当前所在的domain，而default_domain指Group默认应该在的domain。进行attach操作时，会检查default_domain是否与domain相同，以此判断该Group是否已经attach到别的domain上了，在该情况下返回`-EBUSY`。

## iommu_detach_device

该函数与上面的`iommu_attach_device`几乎完全相反，并且该函数也是操作独立设备。这里注意如果Group有自己的default_domain，那么该函数在detach完成之后会重新attach到default_domain上。

## iommu_map

```c
int iommu_map(struct iommu_domain *domain, unsigned long iova,
	      phys_addr_t paddr, size_t size, int prot);
```

函数原型上可以看出来是用于映射domain内的iova，将长度为`size`以`iova`为起始地址的iova区域映射到以paddr为起始地址的物理地址。该函数只能用于`UNMANAGED`类型和`DMA`类型的domain。`domain->pgsize_bitmap`是一个bitmap，用于记录domain支持的最小page大小。iommu_map函数进行操作时，是以page为单位的，page大小不固定可以根据需要使用不同大小的page，在同一次iommu_map操作中也不要求page大小一致。最终一个page的映射是调用iommu->ops中的map回调函数实现的。

## iommu_iova_to_phys

该函数调用`domain->ops`中提供的`iova_to_phys`回调函数将iova转换成物理地址。

TODO dma integration

## IOMMU Group

啃了两天PCIE协议，对IOMMU的Group概念也有了一定的理解。从内核角度来看Group是一组设备，是IOMMU可以辨别的最小单位，即IOMMU无法区分出一个Group中的设备。区分标准是什么呢，IO地址空间。以PCIE总线来举一个例子，我们知道PCIE是一个点对点的协议，如果一个多function设备挂到了一个不支持ACS的bridge下，那么这两个function可以通过该bridge进行通信。这样的通信直接由bridge进行转发而无需通过Root Complex，自然也就无需通过IOMMU。这种情况下，这两个function的IOVA无法完全通过IOMMU隔离开，所以他们需要分到同一个Group中。同一个Group的设备应该是公用一个domain的。

```c
struct iommu_group {
	struct kobject kobj;
	struct kobject *devices_kobj;
	struct list_head devices;
	struct mutex mutex;
	struct blocking_notifier_head notifier;
	void *iommu_data;
	void (*iommu_data_release)(void *iommu_data);
	char *name;
	int id;
	struct iommu_domain *default_domain;
	struct iommu_domain *domain;
};
```

从iommu_group的结构中可以发现，devices列表保存group中设备。一个group需要关联两个iommu_domain，除此之外支持内核中其他组件向该group中注册listener。default_domain保存的是默认该设备应该位于的domain，而domain字段保存设备当前所在的domain。

### iommu_group_add_device

该函数将一个设备加入一个Group，函数的主要操作如下：

* 处理sysfs相关的事项，如建立iommu_group符号链接
* 将设备的iommu_group字段设置为这个Group
* 调用iommu_group_create_direct_mappings建立设备的iova映射
* 将设备加入到Group内的list里
* 通知所有注册到Group里的listener有设备加入

TODO: iommu_group_create_direct_mappings

### iommu_group_get_for_dev

该函数获取一个设备所在的Group，如果设备不属于任何一个Group，则调用IOMMU驱动提供的`device_group`回调函数尝试进行获取。

```c
	group = iommu_group_get(dev);
	if (group)
		return group;

	if (!ops)
		return ERR_PTR(-EINVAL);

	group = ops->device_group(dev);
```

随后，为获取的Group设置domain，最后将设备加入Group。

```c
	if (!group->default_domain) {
		struct iommu_domain *dom;

		dom = __iommu_domain_alloc(dev->bus, iommu_def_domain_type);
		if (!dom && iommu_def_domain_type != IOMMU_DOMAIN_DMA) {
			dev_warn(dev,
				 "failed to allocate default IOMMU domain of type %u; falling back to IOMMU_DOMAIN_DMA",
				 iommu_def_domain_type);
			dom = __iommu_domain_alloc(dev->bus, IOMMU_DOMAIN_DMA);
		}

		group->default_domain = dom;
		if (!group->domain)
			group->domain = dom;
	}
```

## Bus integration

每一个`struct device`中保存了一个`struct iommu_group`的指针，用以获取该设备处于的group。除此之外，内核需要其他方式将iommu功能集成到总线中。很明显，一个iommu设备是作用于一个或者多个总线上的，那么就需要一个自然的方式管理与iommu相关的功能。首先明确`struct bus`中存在一个iommu_ops用于保存当前bus上生效的iommu驱动注册的iommu_ops。同时可以根据这个指针是否为NULL确认这个bus中是否支持iommu功能。

iommu核心框架中提供了`bus_set_iommu`函数，该函数可以被iommu驱动调用，用以将自身挂入到 对应总线中。函数中除了设置iommu_ops指针之外，还进行了两个工作：

* 向bus中注册一个listener：对于bus上设备的插入与移除的设备，调用iommu_ops中对应的add_device和remove_device回调函数。对于bus接收到的其他设备事件（如bind，unbind等），则将其传播给该设备所处于的group中。
* 对于bus中已经存在的设备，则挨个调用`add_device`将其纳入iommu的管辖，并设置其group

### iommu_fwspec

```c
struct iommu_fwspec {
	const struct iommu_ops	*ops;
	struct fwnode_handle	*iommu_fwnode;
	void			*iommu_priv;
	unsigned int		num_ids;
	u32			ids[1];
};
```

## IOMMU Domain

每一个domain即代表一个iommu映射地址空间，即一个page table。一个Group逻辑上是需要与domain进行绑定的，即一个Group中的所有设备都位于一个domain中。

```c
struct iommu_domain {
	unsigned type;
	const struct iommu_ops *ops;
	unsigned long pgsize_bitmap;	/* Bitmap of page sizes in use */
	iommu_fault_handler_t handler;
	void *handler_token;
	struct iommu_domain_geometry geometry;
	void *iova_cookie;
};
```

# SMMU硬件及驱动分析

看代码要点：

* 一定要看文档，SMMU是一个比较简单的设备，他的Spec只有300页
* fault分为global和context，global基本上就是smmu本身的一些fault，而context是smmu在进行地址转换时出现的fault        

开始分析代码，我认为需要从中断处理入手，这也是错误信息的入口。先看context fault的处理函数，代码不贴了基本就是打出相关的寄存器信息，没有什么参考意义。我认为有意义的地方是这个中断是怎么注册的，即驱动是如何管理io domain的。这就涉及通用的IOMMU框架代码了。

可以发现`arm_smmu_attach_dev`函数中将一个设备添加到一个特定的iommu domain中。函数中调用`arm_smmu_init_domain_context`函数注册了这个中断。对于每一个`struct device`，其内部有一个`iommu_group`字段保存其所在的Group。

```c
static struct iommu_ops arm_smmu_ops = {
	.capable		= arm_smmu_capable,
	.domain_alloc		= arm_smmu_domain_alloc,
	.domain_free		= arm_smmu_domain_free,
	.attach_dev		= arm_smmu_attach_dev,
	.map			= arm_smmu_map,
	.unmap			= arm_smmu_unmap,
	.flush_iotlb_all	= arm_smmu_iotlb_sync,
	.iotlb_sync		= arm_smmu_iotlb_sync,
	.iova_to_phys		= arm_smmu_iova_to_phys,
	.add_device		= arm_smmu_add_device,
	.remove_device		= arm_smmu_remove_device,
	.device_group		= arm_smmu_device_group,
	.domain_get_attr	= arm_smmu_domain_get_attr,
	.domain_set_attr	= arm_smmu_domain_set_attr,
	.of_xlate		= arm_smmu_of_xlate,
	.get_resv_regions	= arm_smmu_get_resv_regions,
	.put_resv_regions	= arm_smmu_put_resv_regions,
	.pgsize_bitmap		= -1UL, /* Restricted during device attach */
};
```

由于前面已经分析了IOMMU核心框架，熟悉了IOMMU核心框架如何与IOMMU驱动如何互动。这里分析流程即为以一个设备的IOMMU操作周期为基准分析SMMU驱动向IOMMU核心框架注册回调函数。

## Stream Mapping管理

首先提及一些Spec中定义的名词：

* Steam翻译成中文是流的意思。在SMMU中特指Master设备向SMMU发起的请求流。StreamID即为SMMU用以辨别不同Stream用的编号，注意Stream和设备不是一一对应的关系。
* Stream Mapping在Spec中特指将StreamID映射到Stream Context（接近domain的概念）这一操作行为。

代码中SME应该是`Stream Mapping Entry`的缩写。Spec中提及到三种Stream Mapping的方式，这里主要提及Stream Indexing`和`Stream Matching`两种。

## arm_smmu_add_device

这个函数即为`add_device`回调函数。回忆前面的分析，IOMMU核心框架向bus中注册listener，每当bus中新增设备时，即会调用该函数。从这里看，该函数的主要功能就是将一个设备纳入到IOMMU驱动的管理中。该函数的核心参数就是被传入的`struct device`结构体中保存的`struct iommu_fwspec`。

```c
/**
 * struct iommu_fwspec - per-device IOMMU instance data
 * @ops: ops for this device's IOMMU
 * @iommu_fwnode: firmware handle for this device's IOMMU
 * @iommu_priv: IOMMU driver private data for this device
 * @num_ids: number of associated device IDs
 * @ids: IDs which this device may present to the IOMMU
 */
struct iommu_fwspec {
	const struct iommu_ops	*ops;
	struct fwnode_handle	*iommu_fwnode;
	void			*iommu_priv;
	unsigned int		num_ids;
	u32			ids[1];
};
```

该参数是从ACPI或者设备树中得到的，用以描述设备绑定的IOMMU及拓扑关系。这里需要注意该关系必须遵循硬件设计，不然很明显是无法正常工作的。函数为设备内分配了一个`arm_smmu_master_cfg`结构体，如下：

```c
struct arm_smmu_master_cfg {
	struct arm_smmu_device		*smmu;
	s16				smendx[];
};
```

并保存在iommu_fwspec中的`iommu_priv`指针中。该结构体从名字上就能看出是Master设备的配置，Master这个名词在Spec中是指Bus Master，即可以发起总线请求的设备。函数的核心操作由`arm_smmu_master_alloc_smes`完成，从名字可以看出是为设备分配Stream Mapping中的表项。对于每一个与设备关联的StreamID，都需要分配一个Stream Mapping中的表项：

```c
		ret = arm_smmu_find_sme(smmu, sid, mask);
		if (ret < 0)
			goto out_err;

		idx = ret;
		if (smrs && smmu->s2crs[idx].count == 0) {
			smrs[idx].id = sid;
			smrs[idx].mask = mask;
			smrs[idx].valid = true;
		}
		smmu->s2crs[idx].count++;
		cfg->smendx[i] = (s16)idx;
```

这里的操作简明易懂，唯一需要注意的就是表项的分配方式。首先搜索整个表中是否存在完全匹配（即集合意义上的包含）的表项，如果存在则使用该表项，否则使用表中第一个发现的空表项。到这里可以发现`arm_smmu_master_cfg`中的smendx即为保存该Master设备对应的Stream Mapping表项。表项申请完毕后，其实质上还是没有写入到SMMU的mmio空间去的，写入的话这个表项应该就生效了。但是软件上还是没有准备完毕的，这个设备没有加入任何Group或者domain。这里可以看到：

```c
	group = iommu_group_get_for_dev(dev);
```

这个函数是IOMMU核心框架提供的函数，函数最终还是会调用到`device_group`回调函数。这里我们只需要明确这个函数确定设备属于哪一个Group。最后，为了使表项立马生效，将其写入到S2CR寄存器中：

```c
	for_each_cfg_sme(fwspec, i, idx) {
		arm_smmu_write_sme(smmu, idx);
		smmu->s2crs[idx].group = group;
	}
```

## arm_smmu_device_group

该函数为device_group回调函数，目的是获取一个设备的Group。函数的操作也比较简单：

* sanity check：检查设备所有SME是否都位于同一个Group
* 如果设备已经有一个Group，那么返回该Group
* 设备没有Group的情况下，则需为设备分配一个Group。对于PCI设备调用`pci_device_group`，对于其他设备则为`generic_device_group`

## arm_smmu_domain_alloc

该函数为`domain_alloc`回调函数，其目的是申请一个domain。SMMU驱动只支持三种domain：

```c
	if (type != IOMMU_DOMAIN_UNMANAGED &&
	    type != IOMMU_DOMAIN_DMA &&
	    type != IOMMU_DOMAIN_IDENTITY)
		return NULL;
```

剩下的就是申请内存，初始化一些数据结构了，貌似有一些看着比较关键的字段是空着的。后面可以看到这个时候申请的domain仅仅是占位用的，没有什么实际意义。在调用attach_dev时，会初始化domain context。驱动通过`struct arm_smmu_domain`里的smmu字段判断context是否已经初始化。

## arm_smmu_attach_dev

这里的实现细节是，一个设备的绑定的SMMU与其Stream Mappings已经在`add_device`回调函数中确定好了。attach_dev的实际操作就是根据设备保存的这些信息初始化domain。初始化domain context由`arm_smmu_init_domain_context`函数完成，该函数满满的硬件细节，后续需要专门讨论。一个domain在SMMU硬件中实际对应的概念就是Context Bank，在使用Stream Matching的情况下，一共存在三层映射：StreamID && Mask -> S2CR寄存器 -> Context Bank。因此，初始化完domain（即Context Bank）后需要设置当前设备在Stream Mapping中对应的S2CR寄存器，使其指向该domain对应的Context Bank。
