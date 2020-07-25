+++
title = "Linux内核在RISC-V架构下的架构相关实现"
date = 2020-07-25
draft = true

[taxonomies]
tags = ["kernel", "riscv"]

+++

在分析完Linux内核在RISC-V架构下的启动流程后，我们分析Linux下与RISC-V相关的架构相关实现。很明显，这类知识都是非常零散的，这里使用的入手点为`setup_arch`的实现。

在开始分析之前，一定要对Linux内核的架构相关代码有一定的认识，这里做一个原理性的说明。Linux内核的codebase可以简单分成两个部分：架构相关部分和架构无关部分。为了支持多个架构，且最大限度地公用代码，又保留架构相关实现的灵活性，Linux内核的实现经过了精细的设计。内核底层对一些架构相关的操作进行了抽象，向内核通用代码提供了公共的接口。每一个内核支持的架构都对应一个`arch/`下的文件夹，里面存放着本架构相关的代码。

接下来简要介绍一下内核实现多架构支持的手段：

* 条件编译。这个方法主要用在一些极端特殊场合，多用于驱动对特定平台的区别操作。内核提供了一些宏，用于检测当前架构。
* weak函数。这个方法利用了ELF object文件中的`weak symbol`，具有这个属性的`symbol`在链接时，如果链接器可以在所有进行链接的object文件中找到同名`symbol`，则会用这个`symbol`把`weak symbol`顶替掉。内核使用`__weak`（本质就是一个GCC扩展）标记weak函数。内核可以对所有架构实现一个通用的weak函数，如果有架构需要一个自己的版本，则可以直接定义，并将其顶替。
* 平台相关函数。这类函数为强平台相关，内核一般定义一个共同函数原型及函数语意，由各架构自行实现该函数。这里其实也包括一部分宏。

## setup_arch

阅读过任何一本内核书的人对这个函数一定不陌生。这个函数主要做架构相关的初始化操作。函数首先设置`init_mm`上的四个变量，前面提到过，`_stext`等变量是通过链接脚本放到内核二进制文件特定地址的，通过他们可以获取内核内存布局的范围。注意这里已经开启MMU了，且RISC-V内核使用的是PC相对寻址，因此这时获取的是对应的虚拟地址。

```c
	init_mm.start_code = (unsigned long) _stext;
	init_mm.end_code   = (unsigned long) _etext;
	init_mm.end_data   = (unsigned long) _edata;
	init_mm.brk        = (unsigned long) _end;
```

### 设备树与参数解析

在这个阶段，设备树并没有完全进行解析，内核只需要提取一小部分重要信息即可。回顾原先的分析，`_start_kernel`在调用`start_kernel`前调用了`parse_dtb`函数，如下：

```c
void __init parse_dtb(void)
{
	if (early_init_dt_scan(dtb_early_va))
		return;

	pr_err("No DTB passed to the kernel\n");
#ifdef CONFIG_CMDLINE_FORCE
	strlcpy(boot_command_line, CONFIG_CMDLINE, COMMAND_LINE_SIZE);
	pr_info("Forcing kernel command line to: %s\n", boot_command_line);
#endif
}
```

这个函数简单调用了设备树（OF）模块的通用函数`early_init_dt_scan`，从设备树中解析并设置一些基本的信息。这里简单分析一下获取了哪些信息。在简单检查设备树的合法性之后，可以看到进行了如下操作：

```c
	/* Retrieve various information from the /chosen node */
	rc = of_scan_flat_dt(early_init_dt_scan_chosen, boot_command_line);
	if (!rc)
		pr_warn("No chosen node found, continuing without\n");

	/* Initialize {size,address}-cells info */
	of_scan_flat_dt(early_init_dt_scan_root, NULL);

	/* Setup memory, calling early_init_dt_add_memory_arch */
	of_scan_flat_dt(early_init_dt_scan_memory, NULL);
```

总结如下：

* 扫描设备树的`chosen`节点，获取内核命令行，initrd等关键信息，其中内核并命令行被保存在`boot_command_line`字符数组中
* 扫描`/`节点下的`#address-cells`，`#size-cells`等信息并记录
* 扫描`memory`节点信息，获取设备树中关于内存的描述，并调用`early_init_dt_add_memory_arch`

`early_init_dt_add_memory_arch`函数RISC-V使用了内核默认的实现，即将内核区域添加到`memblock`中。也就是说，RISC-V架构下的启动内存管理器是memblock，memblock的实现比较独立，另开文档进行分析。

在`setup_arch`中调用了`parse_early_param`通用函数，用于解析`early param`。内核中的`early param`都会特殊标记起来，保存在一个特殊的section里，在内核启动初期从内核命令行解析出来。

### memblock初始化 

前面看到，内核中设备树解析的通用代码将设备树中设定的可用内存区域传递给`memblock`进行管理。而在`setup_arch`中则调用`setup_bootmem`初始化memblock。

前面已经看到，从设备树中读取到的可用物理内存范围已经在内核默认的`early_init_dt_add_memory_arch`中加入到了memblock管理器中。此时，memblock还需要进行一些别的操作，使其可用。首当其冲的就是保留特定物理内存，这一点是比较容易想到的。一般情况下，设备树描述的是整个板子可用的物理内存区域，内核二进制，设备树二进制装载进内存时也会占用其中一部分空间，这就需要`setup_bootmem`函数手动保留这些区域。

如果读过设备树的标准，则应该知道，设备树中存在`reserved memory`的描述，因此`setup_bootmem`调用`early_init_fdt_scan_reserved_mem`函数进行扫描，然后将对应参数进行保留。最后函数将所有物理内存区域的NUMA节点设置为0。

这里还有一个点可以稍微提一下：在RISC-V体系结构下，如果开启两级页表，那么最大能够支持的物理内存大小是2GB，如果可用物理内存大小超过了2GB，页表是无法进行映射的。`setup_bootmem`函数因此进行了一项修复，即将超出最大支持范围的物理内存区域从bootmem中移除，避免对其的访问。对二进制比较敏感的话，可以想到，这一大小正好是`-PAGE_OFFSET`。

### paging_init

从名字上就可以看出这个函数是干什么的：初始化页表。前一篇文档看到内核在`setup_vm`中初始化了一个`early_pg_dir`页表，仅仅映射了内核所占用内存和一个FDT的fixmap，而`paging_init`中的`setup_vm_final`则是该操作的延续。首先明确使用两级初始化的原因：

* 在没有读取设备树之前，内核是不知道物理内存的大小的。如果非要缩成一步，那么只能从内核所在内存结尾处开始，猜一个大小然后进行映射。这种实现有巨大的不确定性，并不是一个好的选择
* 紧接上一条，这么做有可能需要映射一些不存在的内存区域，使得页表占用更多空间

所以`setup_vm_final`的操作本质上就是将memblock中管理的内存添加到到`swapper_pg_dir`页表中，然后启用该页表。

`paging_init`随后初始化`Sparse Memory`，当然我们这里没有开启其对应支持，不予分析。`setup_zero_page`函数将内核预先在BSS中预留的一个page进行初始化操作（全写0）。`zone_size_init`函数实现比较简单，但是原先分析时漏掉了一点，这里补上。

目前的内核已经把bootmem去掉了，但是它的影响依然在内核中。`setup_bootmem`函数中设置了几个变量，如下：

```c
	set_max_mapnr(PFN_DOWN(mem_size));
	max_pfn = PFN_DOWN(memblock_end_of_DRAM());
	max_low_pfn = max_pfn;
```

这几个静态变量的意义如下：

```
    min_low_pfn - the lowest PFN that is available in the system
    max_low_pfn - the highest PFN that may be addressed by low memory (ZONE_NORMAL)
    max_pfn - the last PFN available to the system.
```

`zone_size_init`如下：

```c
static void __init zone_sizes_init(void)
{
	unsigned long max_zone_pfns[MAX_NR_ZONES] = { 0, };

#ifdef CONFIG_ZONE_DMA32
	max_zone_pfns[ZONE_DMA32] = PFN_DOWN(min(4UL * SZ_1G,
			(unsigned long) PFN_PHYS(max_low_pfn)));
#endif
	max_zone_pfns[ZONE_NORMAL] = max_low_pfn;

	free_area_init_nodes(max_zone_pfns);
}
```

可以看到`ZONE_DMA32`的边界为4GB，这基本就是废话。记住这里初始化了`pg_data_t`就行了。

### unflatten_device_tree

该函数为OF模块的代码，目的是将设备树转换成更高效的内存中表示。

### sbi_init && setup_smp && riscv_fill_hwcap

这些非常独立的部分单独进行分析。