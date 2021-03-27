+++
title = "KVM实现分析"
date = 2020-05-28
draft = true


tags = ["kernel", "kvm"]

+++

## 模块入口

本来这个东西不应该写的，但是KVM的模块入口比较特殊，与常规的模块不在一个位置。从内核的角度来看，KVM不是一个独立体系结构之外的模块，即使他的代码单独放在了一个独立的文件夹`virt/kvm`中。事实上，那个文件夹放的是通用的代码，每一个需要实现KVM的体系结构会将这些通用代码一同编译到自己的kvm模块中。

简单来说，对于ARM64架构，其kvm模块的代码位于`arch/arm64/kvm`文件夹下，可以看到其Makefile中存在如下结构：

```makefile
KVM=../../../virt/kvm

obj-$(CONFIG_KVM_ARM_HOST) += kvm.o
obj-$(CONFIG_KVM_ARM_HOST) += hyp/

kvm-$(CONFIG_KVM_ARM_HOST) += $(KVM)/kvm_main.o $(KVM)/coalesced_mmio.o $(KVM)/eventfd.o $(KVM)/vfio.o
kvm-$(CONFIG_KVM_ARM_HOST) += $(KVM)/arm/arm.o $(KVM)/arm/mmu.o $(KVM)/arm/mmio.o
```

而其kvm模块的模块初始化代码是在`virt/kvm/arm/arm.c`中实现的。注意这里由于ARM需要同时支持32位和64位的KVM，因此其通用代码是放到`virt/kvm/arm`中的。

```c
static int arm_init(void)
{
	int rc = kvm_init(NULL, sizeof(struct kvm_vcpu), 0, THIS_MODULE);
	return rc;
}
```

## 初始化

前面看到`arm_init`直接调用了`kvm_init`函数，这个函数是KVM的通用初始化函数。与内核中其他地方一样，KVM分为架构相关和架构无关的代码，这两个类代码之间会相互调用。`kvm_init`函数开头即调用了体系结构相关的初始化函数`kvm_arch_init`，下面来看这个函数。

```c
	if (!is_hyp_mode_available()) {
		kvm_info("HYP mode not available\n");
		return -ENODEV;
	}
```

函数首先检查当前CPU是否支持hypervisor模式，其原理为内核启动时会记录CPU0和CPU1的异常级别，如果他们都为EL2，则认为当前CPU支持hypervisor模式。

```c
	if (!in_hyp_mode && kvm_arch_requires_vhe()) {
		kvm_pr_unimpl("CPU unsupported in non-VHE mode, not initializing\n");
		return -ENODEV;
	}
```

接下来的检查需要解释一下：如果CPU当前不在EL2，说明内核没有运行在VHE模式下，说明CPU不支持VHE，但是如果`kvm_arch_requires_vhe`返回true的话，说明内核中的某些特性需要VHE，此时KVM模块认为CPU设计有失误，会拒绝初始化。

接下来内核通过特定手段向所有CPU发送请求，让其执行`check_kvm_target_cpu`函数并返回结果，这个手段是架构相关的，一般为IPI中断。

```c
static void check_kvm_target_cpu(void *ret)
{
	*(int *)ret = kvm_target_cpu();
}
```

`kvm_target_cpu`函数根据CPU ID直接返回对应的KVM Target，默认fallback到Generic Target，也就是说这个检查其实永远不会失败。而这些检查结束之后就是真正的初始化操作了。

### IPA (TODO)

### SVE (TODO)

### HYP (TODO)

这段可以放在分子系统的文档中。

## 中断虚拟化

所有KVM支持的中断控制器驱动在初始化设备的时候会填写一个`struct gic_kvm_info`结构体，并保存在一个静态变量中，这使得KVM可以在初始化中断控制器的时候获取中断控制器的信息。注意ARM64下的KVM是不支持编译成模块的，这使得对应的函数不需要导出符号。

```c
struct gic_kvm_info {
	/* GIC type */
	enum gic_type	type;
	/* Virtual CPU interface */
	struct resource vcpu;
	/* Interrupt number */
	unsigned int	maint_irq;
	/* Virtual control interface */
	struct resource vctrl;
	/* vlpi support */
	bool		has_v4;
	/* rvpeid support */
	bool		has_v4_1;
};

enum gic_type {
	GIC_V2,
	GIC_V3,
};
```

如果阅读过GIC的Datasheet的话上面的定义是非常直观的。VGIC的初始由`init_subsystem`中的`kvm_vgic_hyp_init`函数完成。该函数的核心操作如下：

1. 填充`kvm_vgic_global_state`
2. 注册对应的VGIC设备到KVM模块中，使用户态可以使用`CREATE_DEVICE`创建对应的设备
3. 注册GIC中定义的maintenance中断

```c
extern struct vgic_global kvm_vgic_global_state;
```

`kvm_vgic_global_state`实际上是一个`vgic_global`的全局变量，其内部主要记载GIC的一些基本信息，包括支持的特性和寄存器基质。填充和注册过程极其无聊，不再赘述。

## KVM_CREATE_VM

这个IOCTL是设备级别，用于创建一个虚拟机，是最常用的API之一。这个IOCTL对应`kvm_dev_ioctl_create_vm`函数，该函数最开始直接调用`kvm_create_vm`，下面来分析这个函数。

从函数中可以看到，实际上一个KVM机由`struct kvm`表示，可以想象出这个结构体是比较复杂的。`kvm_create_vm`的大部分工作就是初始化这个结构体。注意这个函数调用了两个体系结构相关的hook：

* kvm_arch_init_vm
* kvm_arch_post_init_vm

且将初始化好的`struct kvm`放到了一个全局列表`vm_list`上。其他字段的初始化忽略不提，在用到他们的时候再看。

如果架构支持`CONFIG_KVM_MMIO`，则`kvm_dev_ioctl_create_vm`接下来会调用`kvm_coalesced_mmio_init`初始化虚拟机。随后函数会为当前进程申请一个新的文件描述符，并创建一个匿名文件，然后注册该文件描述符到当前进程中。

```c
	r = get_unused_fd_flags(O_CLOEXEC);
	file = anon_inode_getfile("kvm-vm", &kvm_vm_fops, kvm, O_RDWR);
	fd_install(r, file);
```

这个文件描述符实际就是VM文件描述符，后续大多数VM级别的IOCTL都是直接操作于这个文件描述符的，其对应回调函数就由`kvm_vm_fops`定义。最后注意这里还会有一个uevent事件生成。

## KVM_CREATE_VCPU

这个IOCTL用于为虚拟机中注册一个VCPU，注意ARM平台下的VCPU是无法在中断控制器初始化之后继续注册的。这个ioctl由`kvm_vm_ioctl_create_vcpu`实现，可以看到，一个VCPU由`struct kvm_vcpu`表示。

一个虚拟机创建的VCPU个数会保存在`kvm->created_vcpus`字段中。ARMv8允许的VCPU个数最大为VGICv3支持的512个。由于VCPU与架构极其相关，所有创建的VCPU的大部分工作是由架构相关的函数实现的：

* kvm_arch_vcpu_precreate
* kvm_arch_vcpu_create
* kvm_arch_vcpu_postcreate

这三个函数在ARM64下的实现下文会分析，这里先看体系结构无关的部分。我们知道VCPU在用户态由VCPU的文件描述符进行表示，且该文件描述符会向用户态提供一个MMAP内存区域，用于与内核态快速共享数据。这个内存区域由`struct kvm_run`表示，并保存在`kvm_vcpu->run`指针中，实际上就是一个page大小。

```c
	page = alloc_page(GFP_KERNEL | __GFP_ZERO);
	if (!page) {
		r = -ENOMEM;
		goto vcpu_free;
	}
	vcpu->run = page_address(page);
```

函数会调用`kvm_vcpu_init`初始化体系结构无关的字段。增加完`kvm`结构体的引用计数之后，创建好的`kvm_vcpu`会被保存在`kvm->vcpus`中。下面来看ARM64相关部分。

首先是`kvm_arch_vcpu_precreate`，该函数只做两件事情：

* 如果使用内核态的中断控制器且该中断控制器已经被初始化了，那么返回-EBUSY阻止其次VCPU创建。实际上就是VGIC的初始化意味着虚拟机的VCPU结构不能进行变动了。
* 检查VCPU的id是否超过虚拟机定义的最大值，该最大值保存在`kvm->arch.max_vcpus`中。

`kvm_arch_vcpu_postcreate`则没有做任何事情。

(TODO) 

`kvm_arch_vcpu_create`函数需要结合应用分析，它作的第一件事如下：

```c
	/* Force users to call KVM_ARM_VCPU_INIT */
	vcpu->arch.target = -1;
	bitmap_zero(vcpu->arch.features, KVM_VCPU_MAX_FEATURES)
```

这里首先明白一件事：ARM64下要求所有的VCPU都使用`KVM_ARM_VCPU_INIT`进行初始化，该IOCTL会设置VCPU的型号（如cortex-a53），支持的特性，以及一些特定的行为。将目标CPU型号设置为-1可以强制要求所有用户调用这一IOCTL。

## KVM_ARM_VCPU_INIT

前面提到这个IOCTL是ARM64平台下初始化VCPU必须调用的API。它的参数非常简单：

```c
struct kvm_vcpu_init {
	__u32 target;
	__u32 features[7];
};
```

其中`target`是一个表示CPU目标平台的整数，而`features`是一个bitmap，存有许多flags，可以改变VCPU的行为，详见内核KVM-API文档。事实上，这个target只能是当前物理CPU的类型，详见`kvm_vcpu_set_target`函数：

```c
	int phys_target = kvm_target_cpu();

	if (init->target != phys_target)
		return -EINVAL;
```

ARM64下提供了一个VM级别的`KVM_ARM_PREFERRED_TARGET`调用，用于获取当前需要进行设置的`target`。

(TODO) 寄存器设置

## KVM_SET_USER_MEMORY_REGION

VM级别的IOCTL，用于设置虚拟机的内存。首先明白内存设置的机制：

* 对于设备内存，用户态无需进行额外的设置，虚拟机访问MMIO时，VCPU线程退出到用户态，并返回推出原因（MMIO及对应的地址），由用户态自行处理
* 对于普通内存，需要用户态初始化虚拟机时传入多段用户态申请好的内存

这个IOCTL即为配置KVM虚拟机普通内存的调用。调用的参数比较简单：

```c
struct kvm_userspace_memory_region {
	__u32 slot;
	__u32 flags;
	__u64 guest_phys_addr;
	__u64 memory_size; /* bytes */
	__u64 userspace_addr; /* start of the userspace allocated memory */
};
```

基本上就是内存区域在用户态的地址，大小，已经想要对应虚拟机的物理地址。一个注册的内存区域被称作一个memory slot，在`struct kvm`中由`kvm->slots_lock`保护，而该IOCTL真正由`__kvm_set_memory_region`进行实现，下面分析它的代码。

```c
	as_id = mem->slot >> 16;
	id = (u16)mem->slot;
```

首先可以看到`slot`被分成的两段：低16位和高位。从内核文档中可以明白，低16位为slot_id，而高位为地址空间ID。接下来就是一些基础的检查，可以忽略。内核中使用`struct kvm_memory_slot`表示一个这样的slot：

```c
struct kvm_memory_slot {
	gfn_t base_gfn;
	unsigned long npages;
	unsigned long *dirty_bitmap;
	struct kvm_arch_memory_slot arch;
	unsigned long userspace_addr;
	u32 flags;
	short id;
};
```

## KVM_IRQFD

这个IOCTL实质上提供了从用户态向虚拟机中注入中断的手段。可以通过该IOCTL向KVM的GSI绑定一个eventfd文件描述符，用户态可以通过写入这个文件描述符通知内核对该中断的注入。该IOCTL的参数比较简单：

```c
struct kvm_irqfd {
	__u32 fd;
	__u32 gsi;
	__u32 flags;
	__u32 resamplefd;
	__u8  pad[16];
};
```

通过`flags`中是否有`KVM_IRQFD_FLAG_DEASSIGN`标志位可以确定此次操作是绑定还是解除绑定，我们只看绑定操作，即`kvm_irqfd_assign`函数。

```c
struct kvm_kernel_irqfd {
	/* Used for MSI fast-path */
	struct kvm *kvm;
	wait_queue_entry_t wait;
	/* Update side is protected by irqfds.lock */
	struct kvm_kernel_irq_routing_entry irq_entry;
	seqcount_t irq_entry_sc;
	/* Used for level IRQ fast-path */
	int gsi;
	struct work_struct inject;
	/* The resampler used by this irqfd (resampler-only) */
	struct kvm_kernel_irqfd_resampler *resampler;
	/* Eventfd notified on resample (resampler-only) */
	struct eventfd_ctx *resamplefd;
	/* Entry in list of irqfds for a resampler (resampler-only) */
	struct list_head resampler_link;
	/* Used for setup/shutdown */
	struct eventfd_ctx *eventfd;
	struct list_head list;
	poll_table pt;
	struct work_struct shutdown;
	struct irq_bypass_consumer consumer;
	struct irq_bypass_producer *producer;
};
```

可以注意到`kvm_kernel_irqfd->inject`字段，是一个`work_struct`，用于真正向虚拟机注入中断：

```c
	INIT_WORK(&irqfd->inject, irqfd_inject);
```

```c
	if (!irqfd->resampler) {
		kvm_set_irq(kvm, KVM_USERSPACE_IRQ_SOURCE_ID, irqfd->gsi, 1,
				false);
		kvm_set_irq(kvm, KVM_USERSPACE_IRQ_SOURCE_ID, irqfd->gsi, 0,
				false);
	} else
		kvm_set_irq(kvm, KVM_IRQFD_RESAMPLE_IRQ_SOURCE_ID,
			    irqfd->gsi, 1, false);
```

当需要注入中断时，会调度起这个`work_struct`。这里要注意到这个IOCTL可以支持两种工作模式，普通模式和RESAMPLED模式，分别对应边沿触发中断和电平触发中断。对于电平触发中断，需要虚拟机系统进行一个解除触发（de-assert）操作，这时KVM可以通过`kvm_irqfd->resampled_fd`传入的eventfd收到该信号。

## KVM_IOEVENTFD

(TODO)

## KVM_SET_GSI_ROUTING

(TODO)

## 中断控制器虚拟化

这里只分析GICv3，算是实现比较典型的功能了，GICv4只是多了中断穿透功能，可以在VFIO代码分析时再看。首先明确中断虚拟化是如何实现的：硬件辅助实现。中断控制器提供VGIC功能，可以看作一个虚拟的中断控制器，管理着自己的虚拟中断，系统软件可以通过编程，将物理中断绑定到虚拟中断上，实现特定的功能。下面就来分析虚拟中断控制器是如何抽象和管理的。

由于KVM最早是在x86平台上开发的，因此中断控制器的抽象比较统一，没有考虑多种中断控制器的情形。在ARM下，原先x86所用的IRQCHIP接口只能创建GICv2的中断控制器，如果我们想要创建GICv3控制器，则需要使用DEVICE接口。

DEVICE接口的设计理念与KVM其他部分是一致的，我们可以通过在VM文件描述符上调用`KVM_CREATE_DEVICE`等IOCTL创建一个特定类型的设备，并返回设备对应的文件描述符，后续对于该设备的操作都可以通过这个文件描述符进行。

```c
  struct kvm_create_device {
        __u32   type;   /* in: KVM_DEV_TYPE_xxx */
        __u32   fd;     /* out: device handle */
        __u32   flags;  /* in: KVM_CREATE_DEVICE_xxx */
  };
```

每一个设备都有多种属性，对于设备的控制可以通过修改对应的属性进行。事实上，我们这里要进行分析的VGICv3就是这样的一个设备，它在内核的定义如下：

```c
struct kvm_device_ops kvm_arm_vgic_v3_ops = {
	.name = "kvm-arm-vgic-v3",
	.create = vgic_create,
	.destroy = vgic_destroy,
	.set_attr = vgic_v3_set_attr,
	.get_attr = vgic_v3_get_attr,
	.has_attr = vgic_v3_has_attr,
};
```

先来看`create`操作，这个操作实际上是`kvm_vgic_create`，该函数向虚拟机中初始化一个中断控制器。函数首先做sanity check，从中我们知道在如下条件时，初始化无法进行：

* 虚拟机已经有中断控制器了
* 创建VGICv2时发现GICv3控制器不支持模拟GICv2
* 任一VCPU没有lock住
* 任一VCPU已经运行过，即调用过`KVM_RUN`
* 当前VCPU数超过最大支持的CPU数

接下来就是设置一些基本的属性：

```c
	kvm->arch.vgic.in_kernel = true;
	kvm->arch.vgic.vgic_model = type;

	kvm->arch.vgic.vgic_dist_base = VGIC_ADDR_UNDEF;

	if (type == KVM_DEV_TYPE_ARM_VGIC_V2)
		kvm->arch.vgic.vgic_cpu_base = VGIC_ADDR_UNDEF;
	else
		INIT_LIST_HEAD(&kvm->arch.vgic.rd_regions);
```

然后函数结束，简单的不可思议。事实上，单单创建一个VGICv3设备是远远不够的，用户态必须要设置一些基本的属性它才能正常工作。一个设备的属性可以在`Documentation/virt/kvm/devices`中找到，VGICv3也不例外。在GUEST系统中，VGICv3看起来就是一个真正的GICv3，可以直接使用GICv3的驱动进行操作，因此用户态实质上就是要设定这个VGICv3在虚拟机中的行为。

属性设置时使用如下结构：

```c
struct kvm_device_attr {
	__u32	flags;		/* no flags currently defined */
	__u32	group;		/* device-defined */
	__u64	attr;		/* group-defined */
	__u64	addr;		/* userspace address of attr data */
};
```

常见的设置项有：

* VGICv3的两组寄存器段（DIST，REDIST，详见硬件手册）在虚拟机内物理地址的MMIO映射
* VGICv3最大支持的中断个数
* VGICv3内部部分寄存器的值
* 初始化VGICv3的控制命令

接下来分析上面提到的初始化VGICv3控制命令，即`KVM_DEV_ARM_VGIC_INIT`，对应于`vgic_init`函数。函数首先初始化DIST部分（Distributor），该部分用于分发SPI中断到对应的VCPU上。可以看到，整个中断控制器由`struct vgic_dist`表示，且由`vgic_dist->spis`（struct vgic_irq类型数组）保存DIST部分的状态。所以DIST的初始化实质上就是初始化这个数组。随后初始化PPI，即位于`kvm_vcpu->arch.vgic_cpu`上的`struct vgic_cpu`结构体里的`private_irqs`数组，接着初始化LPI。

### 用户态中断注入路径

前面看到，用户态可以通过IRQFD向虚拟机中注入一个中断，那么现在就来结合VGIC代码分析这一整个调用路径。无论IRQFD中间怎样调用，最后都是调用`kvm_irq_set`函数设置中断状态。