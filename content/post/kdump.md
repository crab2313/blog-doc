+++
title = "kdump工作机制分析"
date = 2019-11-22


tags = ["kernel", "kdump"]
+++

## kdumpctl

kdumpctl是一个shell脚本，用于查看当前kdump的状态和进行kdump相关操作。kdumpctl的入口是main函数，从中可以看到下列命令行参数：

| 命令      | 说明                                                     |
| --------- | -------------------------------------------------------- |
| start     | 执行capture内核装入                                      |
| stop      | 卸载capture内核                                          |
| status    | 显示kdump状态                                            |
| restart   | stop + start                                             |
| propagate | 将配置文件中设置的ssh-key通过ssh-copy-id拷贝到目标服务器 |
| showmem   | 显示内核为dump预留的内存大小                             |

首先明确kdumpctl可以工作在两种模式下：kdump和fadump，这里只提kdump模式。

### status命令

status命令直接显示当前是否已经装载了capture内核。具体操作为读取`/sys/kernel/kexec_crash_loaded`文件，如果文件内容为1则认为capture内核已经装载。

### showmem命令

即内核为kdump保留的内存空间。从`/sys/kernel/kexec_crash_size`中读取。

### start命令

这里应该是开启kdump服务的操作。但是如果内核的proc中存在`/proc/vmcore`，即当前内核为capture内核，则会进行保存core文件的操作。在primary内核中，会进行启动操作，在操作前会进行一系列检查：

* 检查内核当前是否支持kdump，操作与status命令一致
* 检查内核是否为kdump保留了内存空间，操作与showmem命令一致
* 检查配置文件`/etc/kdump.conf`是否合法
* 检查raw选项配置的硬盘分区中是否保存有上次生成的内核dump，如果有则将其取出保存在默认目录，并清空该分区
* 如果设置了通过ssh远程保存内核dump，则需要检查ssh服务器是否可用
* 检查是否需要重新生成capture kernel使用的initrd，如果需要则进行重新生成

最后则进行kdump内核装载操作。该操作实际上就是调用了kexec工具，如下：

```shell
$KEXEC $KEXEC_ARGS $standard_kexec_args \
		--commandl-line="$KDUMP_COMMANDLINE" \
		--initrd=$TARGET_INITRD $dump_kernel
```

注意到一点是准备内核命令行时加入了`disable_cpu_apicid`参数，其他没有什么值的注意的了。

### stop命令

这个命令就是简单的调用`kexec -p -u`将装入的内核卸载。

### propagate命令

调用ssh-copy-id将`kdump.conf`文件中配置的ssh-key上传到目标服务器中。

### capture kernel initrd生成

该过程在rebuild_kdump_initrd中进行，主要调用如下：

```shell
$MKDUMPRD $TARGET_INITRD $dump_kver
```

其中$MKDUMPRD为`/sbin/mkdumprd -f`。

## kexec-tools

### kexec工具代码组织

首先明确该工具有两种典型的应用场景：

* 普通场景，即kexec的设计初衷：从现在运行的内核加载新内核
* crash模式，即通过kexec获取当前运行内核的coredump信息

在该工具中的代码中，用运行kexec_load系统调用时的flags参数里的`KEXEC_ON_CRASH`标志区分两种情况：

```c
if (info->kexec_flags & KEXEC_ON_CRASH) {
	...
}
```

在本文中只分析crash模式下kexec工具的行为。除此之外，该工具高度与运行平台的架构相关，需要注意区分平台相关与平台无关的代码。

```c
struct file_type file_type[] = {
	{"vmlinux", elf_arm64_probe, elf_arm64_load, elf_arm64_usage},
	{"Image", image_arm64_probe, image_arm64_load, image_arm64_usage},
	{"uImage", uImage_arm64_probe, uImage_arm64_load, uImage_arm64_usage},
};
```

`struct file_type`描述当前体系结构下支持的内核文件格式。probe函数用于验证内核文件合法性，load用于装载内核。后续只分析ELF格式的内核装载。

内核对于kexec特性一共提供了两个系统调用：kexec_load与kexec_file_load，本文重点分析kexec_load。由于kexec_file_load的接口较为简单，其功能主要实现在内核态，该系统调用会在内核代码分析部分提到。kexec_file_load系统调用可以由`-s`命令行参数强行启用。

`kexec_load`系统调用原型如下：

```c
static inline long kexec_load(void *entry, unsigned long nr_segments,
			struct kexec_segment *segments, unsigned long flags);
```

其中entry为指向跳转地址的指针，`segments`为一个kexec_segment类型的数组，flags在crash模式下一定要设置`KEXEC_ON_CRASH`标志。

```c
struct kexec_segment {
	const void *buf;
	size_t bufsz;
	const void *mem;
	size_t memsz;
};
```

`kexec_segment`类型数组的目的是向内核描述一串内存区域，其元素的组织为：

* buf && bufsz： 用户态缓冲区及其长度
* mem && memsz： 物理内存起始地址及其长度

这个参数的目的是让内核将用户态缓冲区的内容拷贝到物理地址中，因此kexec工具的主要功能就如同一个bootloader，通过kexec_load系统调用将特定内存装载到特定物理地址，然后跳转到entry指针指向的内存执行。

### 必要信息的采集

`crash_get_memory_ranges`函数实际上描述如何parse当前系统物理内存layout。函数实质上会输出三个信息：

* system_memory_rgns
* usablemem_rgns
* elf_info

system_memory_rgns和usablemem_rgns是`struct memory_ranges`类型的数据，其结构简单易懂不再赘述。

```c
struct memory_ranges {
        unsigned int size;
        unsigned int max_size;
        struct memory_range *ranges;
};

struct memory_range {
	unsigned long long start, end;
	unsigned type;
#define RANGE_RAM	0
#define RANGE_RESERVED	1
#define RANGE_ACPI	2
#define RANGE_ACPI_NVS	3
#define RANGE_UNCACHED	4
#define RANGE_PMEM		6
#define RANGE_PRAM		11
};
```

函数简单读取`/proc/iomem`文件内容，对其进行解析：

```c
	if (!usablemem_rgns.size)
		kexec_iomem_for_each_line(NULL, iomem_range_callback, NULL);
```

对该文件每一个条目，调用`iomem_range_callback`进行处理。该函数将遇到的`System RAM`区域保存在`system_memory_rgns`，将`Crash kernel`区域保存在`usablemem_rgns`里。函数中可以通过计算得到elf_info的一些字段，elf_info最后用于创建elf文件的header。

```c
struct crash_elf_info {
	unsigned long class;
	unsigned long data;
	unsigned long machine;

	unsigned long long page_offset;
	unsigned long long kern_vaddr_start;
	unsigned long long kern_paddr_start;
	unsigned long kern_size;
	unsigned long lowmem_limit;

	int (*get_note_info)(int cpu, uint64_t *addr, uint64_t *len);
};
```

`kern_paddr_start`即内核的物理起始地址设置为`Kernel code`内存区域的起始地址。内核的长度即`kern_size`可以通过`Kernel data`的结尾地址减去`Kernel code`的起始地址获得。`kern_vaddr_start`为内核的起始虚拟地址，可以通过读取`/proc/kallsym`文件中的`_text`符号地址获取。理解这些参数的获取方法本质上需要理解内核在内存中的layout。

### core信息segment装载

首先明确core信息是什么。在capture内核执行时，并没有直接的方式获取原先crash掉的内核的内存布局，因此开发者设计了针对此情况的辅助机制。capture内核启动时，可以通过`elfcorehdr`命令行参数或者`/chosen/linux,elfcorehdr`设备树节点向其传递一个core类型的ELF文件header。该ELF文件header由kexec放置在crashkernel保留的内存区域中，并通过物理内存地址的方式传递给capture内核。该header以program header的形式记录了原内核运行时的内存布局以及内核crash时用于保存crash信息时的内存区域物理地址。

该操作在kexec中由`load_crashdump_segments`函数实现，其中的核心操作即生成ELF header由`crash_create_elf64_headers`函数实现。对于每一个处于运行状态的逻辑CPU都有与之对应的一片内存区域，内核crash时，会将对应于该CPU的信息（寄存器，内核栈等）保存于此。这片内存区域的物理地址与大小由：

```
/sys/devices/system/cpu/cpu[N]/crash_notes
/sys/devices/system/cpu/cpu[N]/crash_notes_size
```

这两个文件导出到了用户态供kexec读取。与之类似的还有vmcoreinfo：

```
/sys/kernel/vmcoreinfo
```

函数会将这些信息以PT_NOTE的形式记录为ELF header的Program header。除此之外，还需要以PT_LOAD的形式记录内核的`System RAM`内存区域，这些区域可以通过枚举`/proc/iomem`文件获取。对于relocatable的内核或者实际放置地址存在偏移量的内核，仍需重复记录一遍内核占用内存区域的内存空间，即需要新增一个PT_LOAD类型的Program header，然后将其pt_vaddr设置为内核起始虚拟地址。

这个ET_CORE类型的ELF header保存在crashkernel保留的内存区域中，并创建对应的kexec_segment。后续会通过修改DTB并加入`/chosen/linux,elfcorehdr`的方式传递给capture内核。

### 内核segment装载

首先调用`arm64_locate_kernel_segment`函数获取内核装载的起始地址。

```c
		hole = (crash_reserved_mem.start < mem_min ?
				mem_min : crash_reserved_mem.start);
		hole = _ALIGN_UP(hole, MiB(2));
```

注意ARM64内核boot协议中明确要求内核放置地址应该2M对齐。然后调用`fixup_elf_header`函数，目的是根据实际情况修正ELF格式内核的entry和program headers，使后续处理ELF格式文件的通用代码正确执行。

```c
	/* load the kernel */
	if (info->kexec_flags & KEXEC_ON_CRASH)
		/*
		 * offset addresses in elf header in order to load
		 * vmlinux (elf_exec) into crash kernel's memory
		 */
		fixup_elf_addrs(&ehdr);
```

我们实际上要将位于`arm64_mem.phys_offset`物理地址上的内核搬运到`crash_reserved_mem.start`上，搬运偏移量为`crash_reserved_mem.start - arm64_mem.phys_offset`。对于ELF header里的entry地址和所有PT_LOAD类型的program header对应的虚拟地址，要加上这个偏移量，才能使后续的`elf_exec_load`函数将其装载到正确的物理地址上。

对于`elf_exec_load`函数，我们可以忽略其对`ET_DYN`类型ELF文件的处理，因为ELF格式的内核不是该类型的ELF文件。函数中，对于每一个PT_LOAD类型的program header，都执行如下操作：

```c
		add_segment(info,
			phdr->p_data, size,
			phdr->p_paddr + base, phdr->p_memsz);
```

该函数定义如下：

```c
void add_segment(struct kexec_info *info, const void *buf, size_t bufsz,
	unsigned long base, size_t memsz)
{
	add_segment_phys_virt(info, buf, bufsz, base, memsz, 1);
}
```

注意`add_segment_phys_virt`的最后一个参数为1，这表明应该将传入的地址当作虚拟地址对待，通过`virt_to_phys`函数将其转换为物理地址。

### 内核命令行与initrd的传入

ARM64内核启动时，要求用x0寄存器传递DTB在内存中的地址。kexec工具支持从命令行传入被装载内核的DTB文件，因此kexec可以从三个地方获取装载内核的DTB文件：

* 命令行传入文件
* /sys/firmware/fdt
* /proc/device-tree

优先级由高到低。获取到DTB之后，需要分别修正三个地方：

* /chosen/bootargs          # 内核命令行
* /chosen/linux,elfcorehdr           # 前面提到的ELF header地址
* /chosen/linux,usable-memory-range     # 可用的内存区域，即crashkernel保留的内存区域

然后将initrd装载到一个新的kexec_segment中，然后修正DTB，加入initrd相关的结点：

* /chosen/linux,initrd-start
* /chosen/linux,initrd-end

最后将DTB装入自己的kexec_segment中。后面会提到，DTB的地址由purgatory通过x0寄存器传递给capture内核。

### purgatory机制

前面可能已经注意到了，最终传递给`kexec_load`系统调用的entry参数并不是直接指向内核的。这说明kexec实现了自己的微型引导程序。kexec中这段微型引导程序被称为purgatory，其作用主要为校验内核镜像的哈希值与跳转到内核执行。该引导程序唯一需要准备的就是内核执行时将x0置为DTB的物理地址，其他ARM64内核boot协议中要求的条件都将由primary内核在crash时准备妥当。

purgatory的实现比较简单，由一小段汇编与一个用于计算内核镜像哈希值的C语言函数组成。程序被编译链接成relocatable的ELF文件镜像，并由kexec工程的编译系统将这个EFL文件填充至一个字节数组中备用。

```c
const char purgatory[] = {
0x7f, 0x45, 0x4c, 0x46, 0x02, 0x01, 0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
0x01, 0x00, 0x3e, 0x00, 0x01, 0x00, 0x00, 0x00, 0x40, 0x07, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00,
    ...
```

```asm
.globl purgatory_start
purgatory_start:

	adr	x19, .Lstack
	mov	sp, x19

	bl	purgatory

	/* Start new image. */
	ldr	x17, arm64_kernel_entry
	ldr	x0, arm64_dtb_addr
	mov	x1, xzr
	mov	x2, xzr
	mov	x3, xzr
	br	x17

size purgatory_start
```

arm64_kernel_entry和arm64_dtb_addr都是64位的指针，由kexec装载purgatory到内存时填充。注意purgatory是relocatable的镜像，在装入内存时需要进行relocate操作，kexec完整了relocatable ELF镜像的loader。ELF文件格式及其装载操作可以参考ELF Spec文档，这里不再赘述。

```c
	// 读取purgatory_start在内存中relocate后的位置
	info->entry = (void *)elf_rel_get_addr(&info->rhdr, "purgatory_start");
	// image_base = kernel_segment + arm64_mem.text_offset
	elf_rel_set_symbol(&info->rhdr, "arm64_kernel_entry", &image_base,
		sizeof(image_base));
    // 设置装载到内存中的arm64_dtb_addr变量的值
	elf_rel_set_symbol(&info->rhdr, "arm64_dtb_addr", &dtb_base,
		sizeof(dtb_base));
```

### capture内核如何获取vmcore

这个问题的前置问题是：capture内核如何确定自己是capture内核？内核启动时会检测`elfcorehdr`命令行参数（或者`/chosen/linux,elfcorehdr`设备树节点），如果存在则认为自己是capture内核。内核parse完该参数后，会将`elfcorehdr`的地址保存在`elfcorehdr_addr`变量中，后续也以该变量的值确认当前的状态。

```c
static inline bool is_kdump_kernel(void)
{
	return elfcorehdr_addr != ELFCORE_ADDR_MAX;
}
```

用户态判断当前的内核是否为capture内核的直接方式是确认`/proc/vmcore`是否存在。那么分析内核代码时，也需要从这里入手。前面elfcorehdr参数实际上是一个earlyparam，因此在调用`__init`函数之前就已经完成的处理。内核中注册了位于`fs/proc/vmcore.c`中的`vmcore_init`函数，用于处理从primary内核传入的elfcorehdr。

```c
fs_initcall(vmcore_init);
```

该函数在检测到elfcorehdr后会进行处理，然后注册`/proc/vmcore`文件。处理过程较为简单，下面主要提及主要步骤：

* 将所有的PT_NOTE类型的Program header合并成一个
* 重新计算所有Program Header类型的大小及偏移量
* 将所有需要dump内存区域的信息整理成一个链表备用
* 计算vmcore文件的大小

最后用户态读取`/proc/vmcore`文件时，对于vmcore的ELF header和PT_NOTE可以直接读取，对于需要读取内存dump时则从原先primary内核的地址空间读取，省去了大片内存的拷贝。

### PT_NOTE如何生成

前面看到PT_NOTE类型的Program header一共有两类，分别由：

```
/sys/devices/system/cpu/cpu[N]/crash_notes{,_size}
/sys/kernel/vmcoreinfo
```

给出其物理地址。内核crash时应该会向其中写入对应的PT_NOTE类型的Program headers。这种类型的Program header的内容是由不定长的条目组成的，每个条目的header如下：

```c
/* Note header in a PT_NOTE section */
typedef struct elf64_note {
  Elf64_Word n_namesz;	/* Name size */
  Elf64_Word n_descsz;	/* Content size */
  Elf64_Word n_type;	/* Content type */
} Elf64_Nhdr
```

在header之后紧随name和desc的字符串，字符串的首尾必须4字节对齐。name与type表示该条目的名称与类型，是由core文件生成方与读取方协定好的。条目的内容存放在desc中，格式也由core文件生成方自行定义，ELF文件spec中没有过多定义。

先看vmcoreinfo，该文件在内核编译时开启了`CONFIG_CRASH_CORE`时会存在：

```c
#ifdef CONFIG_CRASH_CORE

static ssize_t vmcoreinfo_show(struct kobject *kobj,
			       struct kobj_attribute *attr, char *buf)
{
	phys_addr_t vmcore_base = paddr_vmcoreinfo_note();
	return sprintf(buf, "%pa %x\n", &vmcore_base,
			(unsigned int)VMCOREINFO_NOTE_SIZE);
}
KERNEL_ATTR_RO(vmcoreinfo);

#endif /* CONFIG_CRASH_CORE */
```

这个PT_NOTE只有一个条目，其name与type为`VMCOREINFO`和0。desc格式如下：

```
{TYPE}({NAME})={VALUE}\n
{TYPE}({NAME})={VALUE}\n
{TYPE}({NAME})={VALUE}\n
```

主要记录内核中一些关键符号的物理地址、关键数据类型的字段偏移量和关键常量的值，具体内容可以参见[内核文档](https://www.kernel.org/doc/html/latest/admin-guide/kdump/vmcoreinfo.html)。由于这些信息都是内核的固有信息，因此这些内容在内核初始化是就会自动生成。

```c
subsys_initcall(crash_save_vmcoreinfo_init);
```

下面来介绍crash_notes相关的信息。crash_notes应该是每个逻辑CPU对应一个，所以是一个percpu的变量，文件本身的定义在`driver/base/cpu.c`中可以找到。可以想象到crash_note是一个buffer的物理地址，这个buffer在`kernel/kexec_core.c`中定义。

```c
note_buf_t __percpu *crash_notes;

typedef u32 note_buf_t[CRASH_CORE_NOTE_BYTES/4];
```

该percpu变量由`crash_notes_memory_init`函数初始化。前面看到，ARM64内核下内核崩溃时会调用`machine_crash_shutdown`函数，进而调用到`crash_save_cpu`函数，该函数会填写`crash_notes`。从中可以发现该函数填写的是标准的PTSTATUS信息，即普通coredump通用的线程信息，其name与type分别为`CORE`和NT_PRSTATUS（1）。函数只拷贝了当前CPU的运行task的PID与当前CPU的通用寄存器。

crash_save_cpu可能从两条code path被调用。内核崩溃时，触发崩溃的CPU会调用crash_save_cpu，并通过IPI中断告知其他CPU内核已经崩溃。在其他CPU的IPI中断处理函数中也会调用一次crash_save_cpu用以保存其他CPU的现场。

### build_elf_exec_info

函数本质是通过搜集系统信息获取以下结构体的字段，然后对ELF文件进行合法性检查。传入的buf其实就是crash内核解压后的内容。

```c
struct mem_ehdr {
	unsigned ei_class;
	unsigned ei_data;
	unsigned e_type;
	unsigned e_machine;
	unsigned e_version;
	unsigned e_flags;
	unsigned e_phnum;
	unsigned e_shnum;
	unsigned e_shstrndx;
	unsigned long long e_entry;
	unsigned long long e_phoff;
	unsigned long long e_shoff;
	unsigned e_notenum;
	struct mem_phdr *e_phdr;
	struct mem_shdr *e_shdr;
	struct mem_note *e_note;
	unsigned long rel_addr, rel_size;
};
```

函数内部调用到了`build_elf_info`，继而调用到了`build_mem_ehdr`。`build_mem_ehdr函数`内部基本上就是填充ELF identity相关的字段，然后检查ELF header的合法性，最后根据ELF class的值（elf32或者elf64）将剩余工作委托给`build_mem_elf32_ehdr`函数或者`build_mem_elf64_ehdr`。这两个函数内部仅仅是拷贝对应字段，然后转换对应endian到host端endian。

### elf_arm64_load

前面可以看到，对于每一个架构，架构相关代码注册该架构能够处理的内核文件格式。该函数就是对应于ELF格式的内核装载函数。函数首先调用`build_elf_exec_info`将ELF内核相关的ELF headers拷贝出来，其实就是换个格式原样copy，改了下endian。

然后函数开始处理image header，这里的image header指ARM64的内核镜像的image header。可以从内核文档的ARM64 boot protocol中得到该header的信息。我们从内核的连接脚本可以知道ELF格式的内核有两个PT_LOAD类型的segment，分别为code和data。对于这两个segment，尝试从中读取image header。我们知道有个内核特性能够让内核的text段装载地址随机化，这里需要特殊处理一下，否则是找不到image header的。

```c
		header_offset = ehdr.e_entry - phdr->p_vaddr;

		header = (const struct arm64_image_header *)(
			kernel_buf + phdr->p_offset + header_offset);
```

在确认ARM64内核的magic无误之后，可以认为读取到了image header。接下来需要从image header中拿出两个值：text_offset和image_size，这两个值对于后面装载内核至关重要。在比较旧的内核（<3.17）中没有这两个值，需要使用默认值，这里需要注意。image_size即ARM64内核镜像的大小，text_offset为内核被放置的偏移量，后面详细说明。

函数后续调用`arm64_locate_kernel_segment`计算出内核放置的位置。对于crash模式，放置地址需要在crashkernel保留的内存空间中。

```c
	kernel_segment = arm64_locate_kernel_segment(info);
```

```c
		hole = (crash_reserved_mem.start < mem_min ?
				mem_min : crash_reserved_mem.start);
		hole = _ALIGN_UP(hole, MiB(2));
		hole_end = hole + arm64_mem.text_offset + arm64_mem.image_size;
```

这里第二行做这个对齐操作是因为ARM64启动协议要求内核被放置在一个2M对齐的内存区域上。

## 内核机制

flags中最后一位是`KEXEC_ON_CRASH`，有无这个flags会严格区分两个code path。由于我们分析kexec的目的是理解kdump的实现机制，因此从`KEXEC_ON_CRASH`入手进行分析。

```c
extern struct kimage *kexec_image;
extern struct kimage *kexec_crash_image;
```

内核使用kimage表示一个在内核中的镜像。使用`KEXEC_ON_CRASH`时，操作的是`kexec_crash_image`。当`kexec_load`系统调用传入的`nr_segments`参数为0时，内核会卸载掉对应的系统镜像。

### kexec_segment

```c
struct kexec_segment {
	/*
	 * This pointer can point to user memory if kexec_load() system
	 * call is used or will point to kernel memory if
	 * kexec_file_load() system call is used.
	 *
	 * Use ->buf when expecting to deal with user memory and use ->kbuf
	 * when expecting to deal with kernel memory.
	 */
	union {
		void __user *buf;
		void *kbuf;
	};
	size_t bufsz;
	unsigned long mem;
	size_t memsz;
};
```

这个结构体是kexec_load系统调用传入的主要参数之一，总共有两个形态。在`kexec_load`中，第一个字段为用户态地址，而在`kexec_file_load`中，其第一个字段为内核地址。

从`kimage_load_crash_segment`中可以看到，内核对这个结构体的主要处理是将由前两个参数指定的缓冲区的内容复制到由后两个参数指定的物理内存区域中。

### do_kexec_load

* 申请新的kimage
* 拷贝vmcore_info到新申请的page里(应该不是真的vmcore而是某种指针性质的东西)
* 调用`kimage_load_crash_segment`将kexec_segment中指定的buffer拷贝到对应物理内存区域中

image_terminate ? TODO

kimage_entry_t kimage->head

### kimage_entry_t

```c
struct kimage {
	kimage_entry_t head;
	kimage_entry_t *entry;
	kimage_entry_t *last_entry;
    ...
};
```

kimage结构体的开头三个字段是`kimage_entry_t`类型的数据。该类型为一个指针类型（unsigned long），但是利用了一个trick，使用指针的低位保存特定flags。该指针的高位应该是用来保存一个page的物理地址。

`kimage_entry_t`实际上在crash模式下的kimage中是没有被用到的，但是最后调用cpu_soft_reset时需要将`kimage->head`传入当作一个参数，后续读这段汇编代码时需要深入分析。先看初始化代码：

```c
	image->head = 0;
	image->entry = &image->head;
	image->last_entry = &image->head
```

上面为`do_kimage_alloc_init`函数中用于初始化这三个字段操作。首先明确这几个字段正常使用时分别保存了什么：

* head保存申请的内存区域第一个page的物理地址（物理地址）
* entry是指向当前保存的最后一个entry的指针（虚拟地址）
* last_entry指向申请的内存区域的最后一个entry位置（虚拟地址）

这里可以看到head设置为0，表示这块内存区域还没有申请。后续可以看到，在向kimage增加entry时（kimage_add_entry），会检查是否申请了内存区域，如果没有则进行申请：

```c
	if (image->entry == image->last_entry) {
		kimage_entry_t *ind_page;
		struct page *page;

		page = kimage_alloc_page(image, GFP_KERNEL, KIMAGE_NO_DEST);
		if (!page)
			return -ENOMEM;

		ind_page = page_address(page);
		*image->entry = virt_to_boot_phys(ind_page) | IND_INDIRECTION;
		image->entry = ind_page;
		image->last_entry = ind_page +
				      ((PAGE_SIZE/sizeof(kimage_entry_t)) - 1);
	}
```

这里尚没有分析这些entry的作用，但是可以确认在crash模式下的kexec并不会使用他们，而是直接使用`kimage_terminate`标记entry的结尾。后续需要分析普通模式下的kexec时，再做分析。

### __crash_kexec

`__crash_kexec`函数是kdump的入口函数，在内核调用panic函数时被调用。

```c
__crash_kexec(NULL);
```

函数看着比较简单：

```c
		if (kexec_crash_image) {
			struct pt_regs fixed_regs;

			crash_setup_regs(&fixed_regs, regs);
			crash_save_vmcoreinfo();
			machine_crash_shutdown(&fixed_regs);
			machine_kexec(kexec_crash_image);
		}
```

首先检查capture kernel是否已经装载，这是后续操作可以进行的前提。`crash_setup_regs`函数是一段简单的内联汇编，在其第二个参数为NULL时会dump下来当前寄存器的状态。`crash_save_vmcoreinfo`函数将前面的vmcoreinfo保存下来。`machine_crash_shutdown`函数会暂停所有CPU的执行，并dump下其寄存器状态。最后`machine_kexec`函数真正执行kexec的功能，启动crash内核。

### machine_kexec

前面可以看到，装载crash内核时，申请了一个`control_code_page`。内核在进行kexec操作时，需要尽量不要破坏原有内核的环境，这个control_code_page实质上就是在原先预留出的内存空间中申请出来的page，用于保存relocate内核操作的函数代码。我们可以看到如下操作：

```c
	reboot_code_buffer_phys = page_to_phys(kimage->control_code_page);
	reboot_code_buffer = phys_to_virt(reboot_code_buffer_phys);
	
	memcpy(reboot_code_buffer, arm64_relocate_new_kernel, arm64_relocate_new_kernel_size);
```

该函数实现在`./arch/arm64/kernel/relocate_kernel.S`文件中。在完成一些缓存与flags操作之后，重置cpu：

```c
	cpu_soft_restart(reboot_code_buffer_phys, kimage->head, kimage->start, 0);
```

### cpu_soft_restart

该函数工作本质上由`__cpu_soft_restart`完成。只不过需要将内核的page table还原成idmap（identity map）状态。可以明确entry参数为前面的`reboot_code_buffer_phys`即保存了`arm64_relocate_new_kernel`代码的一个page。剩余部分需要分析汇编代码，当然只看不写的话汇编代码也是比较简单的。

```asm
ENTRY(__cpu_soft_restart)
	/* Clear sctlr_el1 flags. */
	mrs	x12, sctlr_el1
	ldr	x13, =SCTLR_ELx_FLAGS
	bic	x12, x12, x13
	pre_disable_mmu_workaround
	msr	sctlr_el1, x12
	isb

	cbz	x0, 1f				// el2_switch?
	mov	x0, #HVC_SOFT_RESTART
	hvc	#0				// no return

1:	mov	x18, x1				// entry
	mov	x0, x2				// arg0
	mov	x1, x3				// arg1
	mov	x2, x4				// arg2
	br	x18
ENDPROC(__cpu_soft_restart)
```

这里首先注意`.pushsection    .idmap.text, "awx"`指令，他将这个函数放置名为`.idmap.text`的section中，并将其内存属性设置可分配，可写与可执行。函数开头将SCTRL_EL1上的M，A，C，SA，I位清除：

```c
#define SCTLR_ELx_FLAGS	(SCTLR_ELx_M | SCTLR_ELx_A | SCTLR_ELx_C | \
			 SCTLR_ELx_SA | SCTLR_ELx_I)
```

即关闭命令与数据缓存，关闭对齐检查与栈对齐检查，关闭MMU。然后用isb命令清空指令cache。后面这个调用hvc的code path这里不分析，后续需要结合异常处理代码分析。最后函数将参数保存至x0，x1，x2然后跳转到entry（即前面的arm64_relocate_new_kernel）执行。

### arm64_relocate_new_kernel

该函数由汇编写成。先总结一个结论，对于crash模式下的kexec，函数基本不执行任何操作，仅仅是清除`SCTLR_ELx_FLAGS`然后跳转到`kimage->start`然后开始执行，跳转时x0-x3参数的值都为0。为了更明确其行为，帮助理解普通kexec下进行的操作，有必要分析这个函数。前面提到函数开头会检测当前是否处于EL2，如果是则会清除对应的flags：

```asm
	/* Clear the sctlr_el2 flags. */
	mrs	x0, CurrentEL
	cmp	x0, #CurrentEL_EL2
	b.ne	1f
	mrs	x0, sctlr_el2
	ldr	x1, =SCTLR_ELx_FLAGS
	bic	x0, x0, x1
	pre_disable_mmu_workaround
	msr	sctlr_el2, x0
	isb
1:
```

函数之后处理kimage->head，看到这里明白`kimage->head`实际上是新kernel进行relocation所需的信息。当然对于crash模式下的kexec来说，`kimage->head`是IND_DONE，loop会直接跳出。

```asm
	/* Check if the new image needs relocation. */
	tbnz	x16, IND_DONE_BIT, .Ldone
```

loop在循环时会检查entry里flags的值：

* IND_DESTINATION：将entry里的page地址设置为拷贝时的目标地址dest
* IND_INDIRECTION：循环在遇到这个标志时，会从entry指向的page里读取新的entry。这样可以实现多buffer保存entry
* IND_SOURCE：循环遇到这个标志时，会把entry指向的page整个复制到dest指向的page，并将dest的值自增指向dest向后的下个page
* IND_DONE: 循环读到这个flag时，会退出执行

loop处理完毕时，新内核的relocation也完成了，函数在刷新完缓存后跳转到`kimage->start`进行执行。

```asm
	/* Start new image. */
	mov	x0, xzr
	mov	x1, xzr
	mov	x2, xzr
	mov	x3, xzr
	br	x17
```

