+++
title = "PCI驱动框架分析"
date = 2020-02-19


tags = ["kernel", "pci"]
+++

# PCI

分析流程：

* 硬件文档
* PCI框架核心层
* PCI框架热插拔
* vfio
* iommu
* vfio接口层与用户态使用cloud-hypervisor

一共十一万行代码。



从ACPI的codepath开始看吧，比较熟悉一点。首先是找到初始化函数，`acpi_pci_init`。可以看到该函数首先从FADT中的boot_flags标志里检查两个标志：

| flags             | desc                   |
| ----------------- | ---------------------- |
| ACPI_FADT_NO_MSI  | 表示不支持MSI          |
| ACPI_FADT_NO_ASPM | 表示不支持高级电源管理 |



## ACPI下对PCI总线的枚举操作

ACPI的Definition Block中使用`PNP0A03`表示一个PCI Host Bridge。有关ACPI的知识就不提了，这里只总结机制，剩下的交给ACPI的Spec。在内核代码简单搜索即可发现对于PCI Host Bridge的处理在`drivers/acpi/pci_root.c`文件中。

```c
static const struct acpi_device_id root_device_ids[] = {
	{"PNP0A03", 0},
	{"", 0},
};

static struct acpi_scan_handler pci_root_handler = {
	.ids = root_device_ids,
	.attach = acpi_pci_root_add,
	.detach = acpi_pci_root_remove,
	.hotplug = {
		.enabled = true,
		.scan_dependent = acpi_pci_root_scan_dependent,
	},
};
```

TODO： 这里留存有疑点，即内核中是如何处理`_HID`和`_CID`的关系的。PCIE Host Bridge一般将`_CID`设置成`PNP0A03`。

因此可以发现这里进行设备枚举的代码即为`acpi_pci_root_add`函数。函数首先通过`_SEG`确定该PCI Host bridge的segment group。随后通过`_CRS`里的BusRange类型资源取得该Host Bridge的Secondary总线范围，保存在`root->secondary`这个resource中。然后设置基本属性：

```c
	root->device = device;
	root->segment = segment & 0xFFFF;
	strcpy(acpi_device_name(device), ACPI_PCI_ROOT_DEVICE_NAME);
	strcpy(acpi_device_class(device), ACPI_PCI_ROOT_CLASS);
	device->driver_data = root;
```

`_CBA`对象中可以保存这个PCI Host Bridge的用于MMCONFIG枚举的基址：

```c
	root->mcfg_addr = acpi_pci_root_get_mcfg_addr(handle)
```

TODO _OSC

```
	negotiate_os_control(root, &no_aspm);
```

随后，调用`pci_acpi_scan_root`函数枚举这个Host Bridge上的设备。该函数是一个平台相关的函数，即各个平台独立实现了该函数，下面分析ARM64平台下的实现。

```c
struct pci_ecam_ops pci_generic_ecam_ops = {
	.bus_shift	= 20,
	.pci_ops	= {
		.map_bus	= pci_ecam_map_bus,
		.read		= pci_generic_config_read,
		.write		= pci_generic_config_write,
	}
};
```



## Host Bridge管理



## 通用控制器驱动分析

这里分析内核中实现的通用控制器驱动，该驱动通过ECAM实现对设备的枚举，在ARM中比较常用。该驱动一般通过设备树进行配置，其设备树bingding位于`bindings/pci/host-generic-pci.txt`文件中。

```c
struct pci_ecam_ops {
	unsigned int			bus_shift;
	struct pci_ops			pci_ops;
	int				(*init)(struct pci_config_window *);
};

struct pci_ops {
	int (*add_bus)(struct pci_bus *bus);
	void (*remove_bus)(struct pci_bus *bus);
	void __iomem *(*map_bus)(struct pci_bus *bus, unsigned int devfn, int where);
	int (*read)(struct pci_bus *bus, unsigned int devfn, int where, int size, u32 *val);
	int (*write)(struct pci_bus *bus, unsigned int devfn, int where, int size, u32 val);
};
```

从名字可以看出pci_ecam_ops是用于操作ECAM空间的一组函数指针。驱动通过填充这组函数指针，可以向通用代码提供一个统一的操作入口。对于`pci-host-ecam-generic`来说，其操作定义如下：

```c
struct pci_ecam_ops pci_generic_ecam_ops = {
	.bus_shift	= 20,
	.pci_ops	= {
		.map_bus	= pci_ecam_map_bus,
		.read		= pci_generic_config_read,
		.write		= pci_generic_config_write,
	}
};
```

下面来看进行probe操作的通用函数`pci_host_common_probe`。事实上这个驱动对于PCI总线一般会进行两个操作：配置与枚举。配置操作即为配置整个PCI总线，分配总线和设备号，而枚举操作即为常规的枚举总线上的设备。在PCI总线已经配置好（如固件配置，或者虚拟机等）的情况下，可以通过chosen节点下配置`linux,pci-probe-only`让内核跳过配置操作。
