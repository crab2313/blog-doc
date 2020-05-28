+++
title = "TTM内存分配器分析"
date = 2020-04-26

[taxonomies]
tags = ["kernel", "drm"]
+++

# TTM

本文结合QXL内的实现分析内核DRM框架中提供的TTM内存管理器。

## BO

BO是Buffer Object的缩写，与Buffer是有区别的。个人理解BO和Buffer最大的区别就是BO比Buffer多了好几个操作，且BO的后端Buffer位置是会变化的。这是因为在GPU工作时，同一个BO可能被管理器移动到三个位置上：

* GPU的专有内存中，也称显存、VRAM
* 系统内存中
* 系统内存在磁盘中的缓存中

因此，BO提供了这些操作的抽象。注意BO有可能被多个对象访问，如GPU和CPU，且GPU和CPU只能访问特定位置上的BO。如，GPU只能访问VRAM和（部分）系统内存中的BO。这其中的拷贝和管理就需要TTM进行。

## TTM的局限性

TTM有一大堆问题，但作为目前最成熟的codebase，基本上所有开发者都是边骂边用。目前的基本操作是使用GEM当作用户态前端，但是后端使用TTM进行。目前TTM的codebase经过多年clean up，目前还剩1w行左右。

## 初始化

TTM的初始化由`ttm_bo_device_init`函数进行，该函数会会让驱动得到一个`ttm_bo_device`。

`ttm_bo_init_mm`负责初始化一个memory_manager。

```c
struct ttm_bo_device {
	struct list_head device_list;
	struct ttm_bo_driver *driver;
	struct ttm_mem_type_manager man[TTM_NUM_MEM_TYPES];
	struct drm_vma_offset_manager *vma_manager;
	struct list_head ddestroy;
	struct address_space *dev_mapping;
	struct delayed_work wq;

	bool need_dma32;
	bool no_retry;
};
```

所有的`ttm_bo_device`都会放到一个`ttm_bo_global`上的`device_list`中。

## ttm_mem_type_manager

前面看到`ttm_bo_device`中保存了一个`ttm_mem_type_manager`数组，即用于管理多种内存类型的管理器。注意到其内部保存了一组`func`函数指针，该指针通常由使用TTM的驱动进行设置，一般为`ttm_bo_manager_func`。

```c
const struct ttm_mem_type_manager_func ttm_bo_manager_func = {
	.init = ttm_bo_man_init,
	.takedown = ttm_bo_man_takedown,
	.get_node = ttm_bo_man_get_node,
	.put_node = ttm_bo_man_put_node,
	.debug = ttm_bo_man_debug
};
```

事实上DRM子系统提供了自己的内存分配器，称为`drm_mm`，具体信息详见[内核文档](https://www.kernel.org/doc/html/latest/gpu/drm-mm.html#drm-mm-range-allocator)。`ttm_mem_type_manager_func`中的回调函数使用`drm_mm`管理各类型的存储空间。有关于初始化的`init`和与之相反的`takedown`可以不谈，主要需要分析`get_node`函数，该函数为内存分配的入口函数。函数事实上的实现也很简单，仅仅是调用`drm_mm`的接口获取`drm_mm_node`并将其写入传入的`ttm_mem_reg`中：

```c
		mem->mm_node = node;
		mem->start = node->start;
```

注意这个`start`字段，后面会看到可以以它计算出BO对应的物理地址。

除了`TTM_PL_SYSTEM`类型外的`ttm_mem_type_manager`一般由驱动程序自行初始化。初始化`ttm_mem_type_manager`的入口函数为`ttm_bo_init_mm`，该函数会调用驱动程序注册给`ttm_bo_device`的`ttm_bo_driver`上的`init_mem_type`函数指针初始化对应类型的内存管理器。

## ttm_buffer_object

这个对象应该是TTM管理的BO的基类，对BO的管理应该都是围绕它进行的。从这里看到TTM和GEM其是并不是独立的两个组件，`ttm_buffer_object`是以`drm_gem_object`为基类的。

先来看明白一个结构体：`ttm_placement`，如下：

```c
struct ttm_placement {
	unsigned		num_placement;
	const struct ttm_place	*placement;
	unsigned		num_busy_placement;
	const struct ttm_place	*busy_placement;
};

struct ttm_place {
	unsigned	fpfn;
	unsigned	lpfn;
	uint32_t	flags;
};
```

每个`ttm_buffer_object`都有与之关联的`ttmp_placement`，而`ttm_placement`的意义也很简单，即BO可以放置的位置和当空间紧张时，BO可以放置的位置。`ttm_place`实际上描述了一段内存区域（由起始和结束PFN描述）和一个flags。这两个结构体结合即可描述清楚BO对于存储空间的偏好。

除此之外BO还有类型属性，即：

```c
enum ttm_bo_type {
	ttm_bo_type_device,
	ttm_bo_type_kernel,
	ttm_bo_type_sg
};
```

这里只需要明白device和kernel的最大区别是kernel类型的BO无法被用户态进行访问。

`ttm_bo_init`函数负责初始化一个`ttm_buffer_object`，而其内部由`ttm_bo_init_reserved`实现。

## ttm_mem_reg

`ttm_bo_init_reserved`使用`ttm_bo_validate`函数验证`ttm_buffer_object`是否在正确的位置上，如果不是，则调用move操作将其放置在正确的位置上。一个BO当前处于的存储位置由`ttm_mem_reg`描述：

```c
struct ttm_mem_reg {
	void *mm_node;
	unsigned long start;
	unsigned long size;
	unsigned long num_pages;
	uint32_t page_alignment;
	uint32_t mem_type;
	uint32_t placement;
	struct ttm_bus_placement bus;
};

struct ttm_bus_placement {
	void		*addr;
	phys_addr_t	base;
	unsigned long	size;
	unsigned long	offset;
	bool		is_iomem;
	bool		io_reserved_vm;
	uint64_t        io_reserved_count;
};
```

## 地址空间映射

该操作主要指将一个BO映射到CPU的虚拟地址，该操作是BO最主要的操作之一。只有映射到虚拟地址空间，一个BO后端的Buffer才能被CPU访问，才能被应用程序操作（读写）。TTM提供了好几组工具函数，用于实现各种各样的映射，首先来看`ttm_bo_kmap/kunmap`。

提到映射，那么我们首先要明白TTM允许使用它的驱动程序注册一组callback，这组callback在映射/取消映射时被调用，可以让驱动程序hook自己的操作。这组callback为`ttm_bo_driver`上的`io_mem_reserve`和`io_mem_free`。

`ttm_bo_kmap`函数的主要工作为将BO映射到内核虚拟地址空间中，注意它的最后一个参数`map`，为一个`ttm_bo_kmap_obj`结构体。该结构体用于表示一个kmap映射，如下：

```c
#define TTM_BO_MAP_IOMEM_MASK 0x80
struct ttm_bo_kmap_obj {
	void *virtual;
	struct page *page;
	enum {
		ttm_bo_map_iomap        = 1 | TTM_BO_MAP_IOMEM_MASK,
		ttm_bo_map_vmap         = 2,
		ttm_bo_map_kmap         = 3,
		ttm_bo_map_premapped    = 4 | TTM_BO_MAP_IOMEM_MASK,
	} bo_kmap_type;
	struct ttm_buffer_object *bo;
};
```

很明显`virtual`为映射好之后的起始虚拟地址，而从后面可以看到`page`为kmap下映射用的`struct page`指针（注意kmap模式只有在映射一个page，且允许缓存时使用）。简单分析了这个参数后，来看正主。除去参数合法性检查等操作后，可以看到函数调用了`ttm_mem_io_reserve`，这本质就是调用了上面提到的`io_mem_reserve`回调函数。随后，函数根据BO是否为IO内存调用`ttm_bo_kmap_ttm`或者`ttm_bo_ioremap`。

`ttm_bo_kmap_ttm`函数像前面提到的一样，在只映射一页且允许缓存的情况下使用kmap，在其他情况下使用vmap进行映射，这在`ttm_tt`的分析里详细看。而`ttm_bo_ioremap`的实现更加直观，分为两种情况：已经映射和没有映射。对于已经映射的BO，其`bo->mem.bus.addr`不为0，则将`ttm_bo_kmap_obj`的类型设置为`ttm_bo_map_premapped`，并以该虚拟地址作为映射的虚拟地址。对于没有进行映射的BO，则根据`ttm_buffer_object`中的`ttm_mem_reg`中的`placement`标志中是否存在`TTM_PL_FLAG_WC`对物理地址调用ioremap_wc或者ioremap函数进行映射。最后一提，映射使用的物理地址通过`ttm_mem_reg.bus`上的`base+offset`计算而来。

## ttm_mem_glob

`ttm_mem_glob`是一个`ttm_mem_global`结构体，是一个单例对象，没错是全局的。这意味着无论内核中无论同时跑了多少个使用TTM的显卡驱动实例，它们都使用的是同一个`ttm_mem_glob`对象。首先明确`TTM`并没有直接调用内存子系统进行page的分配回收操作，而是增加了自己的抽象。

```c
#define TTM_MEM_MAX_ZONES 2
struct ttm_mem_zone;
extern struct ttm_mem_global {
	struct kobject kobj;
	struct workqueue_struct *swap_queue;
	struct work_struct work;
	spinlock_t lock;
	uint64_t lower_mem_limit;
	struct ttm_mem_zone *zones[TTM_MEM_MAX_ZONES];
	unsigned int num_zones;
	struct ttm_mem_zone *zone_kernel;
#ifdef CONFIG_HIGHMEM
	struct ttm_mem_zone *zone_highmem;
#else
	struct ttm_mem_zone *zone_dma32;
#endif
} ttm_mem_glob;
```

该全局对象由`ttm_mem_global_init`初始化，我们只看它初始化的page pool，也就是page缓存池。这个page缓冲区池由`ttm_pool_manager`描述，也是单例对象，由`ttm_page_alloc_init`函数初始化。

```c
struct ttm_pool_manager {
	struct kobject		kobj;
	struct shrinker		mm_shrink;
	struct ttm_pool_opts	options;

	union {
		struct ttm_page_pool	pools[NUM_POOLS];
		struct {
			struct ttm_page_pool	wc_pool;
			struct ttm_page_pool	uc_pool;
			struct ttm_page_pool	wc_pool_dma32;
			struct ttm_page_pool	uc_pool_dma32;
			struct ttm_page_pool	wc_pool_huge;
			struct ttm_page_pool	uc_pool_huge;
		} ;
	};
};
```

这里注意一个逻辑，有cache的page是不需要缓存起来的，可以直接通过内存子系统提供的接口进行申请和回收。其于的情况下都会为其维护一个page池，由`struct ttm_page_pool`描述。

```c
struct ttm_page_pool {
	spinlock_t		lock;
	bool			fill_lock;
	struct list_head	list;
	gfp_t			gfp_flags;
	unsigned		npages;
	char			*name;
	unsigned long		nfrees;
	unsigned long		nrefills;
	unsigned int		order;
};
```

## ttm_tt

TTM的全称是`Translation Table Manager`，那么`ttm_tt`中的`tt`就应该是`Translation Table`的缩写。这个结构体在BO的操作中时有出现，很显然它用管理内存页表，应用于存在于非VRAM空间中的BO。从名字上来看`ttm_tt`是TTM的核心对象之一。

```c
struct ttm_tt {
	struct ttm_bo_device *bdev;
	struct ttm_backend_func *func;
	struct page **pages;
	uint32_t page_flags;
	unsigned long num_pages;
	struct sg_table *sg; /* for SG objects via dma-buf */
	struct file *swap_storage;
	enum ttm_caching_state caching_state;
	enum {
		tt_bound,
		tt_unbound,
		tt_unpopulated,
	} state;
};
```

`ttm_tt`实质上管理BO的存储后端，所以它是嵌入到`ttm_buffer_object`中的。事实上`ttm_buffer_object->ttm`的创建是通过驱动程序注册的`ttm_bo_driver->ttm_tt_create`回调函数实现的，下面是`ttm_ttm_create`函数的片段：

```c
	bo->ttm = bdev->driver->ttm_tt_create(bo, page_flags);
```

这个回调函数内部一般会调用`ttm_tt_init`初始化`ttm_tt`，本质上是初始化`ttm_tt`各字段并申请一个用于存放`struct page`的数组。

`ttm_tt`的`state`中保存了当前的状态，即：

* unpopulated，内存没有分配
* unbound，内存已经分配，但是没有bind
* bind，内存已经分配，且已经bind
