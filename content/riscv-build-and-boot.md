+++
title = "Linux内核在RISC-V架构下的构建与启动"
date = 2020-07-25

[taxonomies]
tags = ["kernel", "risc-v"]

+++

本文分析RISC-V的linux移植是如何完成的，并给出具体的入手方法，希望对后来者有所启发。RISC-V是一个比较新的体系结构，截至目前已经完成了非特权级Spec和特权级Spec（不包含hypervisor）的修订，由于后发优势，加上设计得当，RISC-V的设计相当简洁易懂，是学习体系结构以及Linux内核的良好素材。借助RISC-V，我们可以通读与RISC-V体系结构相关的Linux内核代码，这在其他体系结构上对于初学者是很难做到的。

首先我们需要通读RISC-V的手册，两本加起来不到三百页，对于稍有基础的人来说可能只需要两天就能够读完。RISC-V的汇编语言也非常简洁，设计独到，简化了程序开发人员的许多工作，推荐目前正在施工的[官方教程](https://github.com/riscv/riscv-asm-manual/blob/master/riscv-asm.md)。本文尽量假定读者不熟悉内核的一些tricks，会做适当说明，至少会给出对应关键字，用以查找学习。

体系结构相关代码一般放置于`arch/${arch}/`文件夹下，主要涉及内核中与体系结构相关的部分，每个文件夹下的内容只在特定的体系结构中使用。我们主要研究其中的如下部分：

* 内核的构建
* 内核的启动
* 内核体系结构相关函数的实现
* 内存一致性模型的映射

## 内核的构建

很多与内核相关的书中都没有具体提到内核是如何构建的，最多介绍一下内核的大致内存布局。这里我主要介绍一下内核构建的大致原理。这里涉及的大部分内容实质上为ELF格式、编译器、链接器以及装载器的原理。Linux内核实质上为一个巨大的二进制可执行文件，这个可执行文件的格式一般是不确定的，与体系结构有关。这个格式实质上是与bootloader的约定，bootloader可以检测并识别特定的内核格式，然后将其装载到内存中并执行。这个约定一般被明确写在`Documentation/${arch}/`下的某个文件中，被称作`Boot Protocol`，即启动协议。特化到RISC-V体系结构，这个格式为与ARM64体系结构相同的PE格式，该格式的一个显著优势是可以作为UEFI Executable由UEFI直接执行，即我们熟知的UEFI stub启动方式。

熟悉了上面的概念后，我们来看内核的`Makefile`是如何生成RISC-V下的内核的。首先找到最显眼的文件`arch/riscv/Makefile`，可以看到如下定义：

```makefile
# Default target when executing plain make
boot            := arch/riscv/boot
KBUILD_IMAGE    := $(boot)/Image.gz

...

ifeq ($(CONFIG_RISCV_M_MODE)$(CONFIG_SOC_KENDRYTE),yy)
KBUILD_IMAGE := $(boot)/loader.bin
else
KBUILD_IMAGE := $(boot)/Image.gz
endif
BOOT_TARGETS := Image Image.gz loader loader.bin

all:    $(notdir $(KBUILD_IMAGE))

$(BOOT_TARGETS): vmlinux
        $(Q)$(MAKE) $(build)=$(boot) $(boot)/$@
        @$(kecho) '  Kernel: $(boot)/$@ is ready'
```

不难得出如下结论：

* 在非勘智SOC（K210）下，默认执行make后生成的文件为`Image.gz`
* 对于由`BOOT_TARGETS`中定义的几种镜像名称，可以在`arch/riscv/boot`下的`Makefile`中找到生成方式

我们现在就来看`Image`是如何生成的：

```makefile
OBJCOPYFLAGS_Image :=-O binary -R .note -R .note.gnu.build-id -R .comment -S

$(obj)/Image: vmlinux FORCE
        $(call if_changed,objcopy)

$(obj)/Image.gz: $(obj)/Image FORCE
        $(call if_changed,gzip)
```

可以看到`Image`实质上就是由vmlinux通过`objcopy -O binary`生成的binary格式。具有嵌入式开发经验的人对这个操作会比较熟悉，binary格式实质上就是ELF装载到内存后的内存dump，即以ELF起始装载地址指向的物理地址为起始地址的整片内存区域转储。而这样就比较有趣了，看到这里你一定能猜出`vmlinux`用了什么特殊手段将其内存布局安排的明明白白，使其通过objcopy后，可以得到一个PE格式的可执行程序。这个手段就是**链接脚本**。

关于链接脚本的解读就不赘述了，建议通读`GNU ld`的文档。我们可以在`arch`文件夹下找到生成链接脚本的模板文件：`arch/riscv/kernel/vmlinux.ld.S`。之所以称其为模板文件，是因为这个文件通过C语言的预处理后，可以生成最终的链接脚本`vmlinux.ld`。脚本的内容比较简单，我们先来看前几行：

```asm
OUTPUT_ARCH(riscv)
ENTRY(_start)

jiffies = jiffies_64;
```

这几行比较常规，可以知道：

* 输出的ELF为riscv体系结构
* 输出的ELF的entry point为`_start`符号的地址
* 符号`jiffies`与`jiffies_64`共用一个地址。这里可以参考其它文档中对于时间子系统的描述，这个trick可以让我们在内核中通过`jiffies`变量访问`jiffies_64`变量的低32位。

接下来就是最关键的部分：`SECTIONS`定义，它能精确控制链接器的行为，按照你的需求合并section，并控制内存布局。

```c
#define LOAD_OFFSET PAGE_OFFSET
#include <asm/vmlinux.lds.h>

...

SECTIONS
{
        /* Beginning of code and text segment */
        . = LOAD_OFFSET;
        _start = .;
        HEAD_TEXT_SECTION
        . = ALIGN(PAGE_SIZE);
```

可以看到，整个内存的布局由`LOAD_OFFSET`，即`PAGE_OFFSET`地址开始，而`PAGE_OFFSET`大部分书中都有所涉及，这里不再提它是什么。我们可以看到`_start`符号的地址被定义为`PAGE_OFFSET`，即vmlinux虚拟地址空间的最开头，这也是其名字的由来，在内核中可以通过`&_start`获取其地址。

如果对ELF格式有一定理解，这一段可以跳过不看。这里`SECTIONS`中定义的是虚拟内存的布局，事实上ELF格式中是严格区分物理地址（在链接器的定义中称为装载地址`LMA`）和虚拟地址的，这一点要铭记。在大多数情况下，这两个地址是相同的，但由于内核会将其位于的物理地址通过页表映射到PAGE_OFFSET上位置（不是绝对，有偏移量随机化实现），所以需要严格区分这两种情况。

```c
/* Section used for early init (in .S files) */
#define HEAD_TEXT  KEEP(*(.head.text))

#define HEAD_TEXT_SECTION                                                       \
        .head.text : AT(ADDR(.head.text) - LOAD_OFFSET) {               \
                HEAD_TEXT                                               \
        }
```

`HEAD_TEXT_SECTION`定义如上，很简单，即将所有`.o`文件名为`.head.text`的section合并成vmlinux名为`.head.text`的section。这里注意两个trick：

* 链接脚本中可以通过AT属性指定section的装载地址，这里可以看到`.head.text`的装载地址是0x0
* 链接器需要对最终生成的ELF进行优化，有可能会删除section没有被应用的符号，这里使用KEEP阻止链接器进行这个操作

很明显这个名为`.head.text`的section就是objcopy后得到的`Image`镜像的开头部分。我们已经知道了`Image`是一个PE兼容的镜像，所以这里一定定义了PE header。这里稍微有一点经验的人即可猜到`.head.text`是定义在`head.S`文件中的，这也是其名字的由来。对于内核内的汇编代码文件，`linux/init.h`中定义了多个helper，用以简化section的定义：

```c
/* For assembly routines */
#define __HEAD          .section        ".head.text","ax"
#define __INIT          .section        ".init.text","ax"
#define __FINIT         .previous

#define __INITDATA      .section        ".init.data","aw",%progbits
#define __INITRODATA    .section        ".init.rodata","a",%progbits
#define __FINITDATA     .previous

#define __MEMINIT        .section       ".meminit.text", "ax"
#define __MEMINITDATA    .section       ".meminit.data", "aw"
#define __MEMINITRODATA  .section       ".meminit.rodata", "a"
```

接着我们在`arch/riscv/kernel/head.S`文件中看到这个section的定义：

```asm
__HEAD
ENTRY(_start)
        /*
         * Image header expected by Linux boot-loaders. The image header data
         * structure is described in asm/image.h.
         * Do not modify it without modifying the structure and all bootloaders
         * that expects this header format!!
         */
        /* jump to start kernel */
        j _start_kernel
        /* reserved */
        .word 0
        .balign 8
#if __riscv_xlen == 64
        /* Image load offset(2MB) from start of RAM */
        .dword 0x200000
#else
        /* Image load offset(4MB) from start of RAM */
        .dword 0x400000
#endif
        /* Effective size of kernel image */
        .dword _end - _start
        .dword __HEAD_FLAGS
        .word RISCV_HEADER_VERSION
        .word 0
        .dword 0
        .ascii RISCV_IMAGE_MAGIC
        .balign 4
        .ascii RISCV_IMAGE_MAGIC2
        .word 0
```

可以看到该section开头即为精心构建的PE header。至此，内核的构建方式已经比较明了，即利用链接脚本，精心设置整个vmlinux文件的布局，将`.head.text`放置到最前，并在其开头填充PE header，最后用objcopy导出这个带有PE header的二进制`Image`镜像。原理非常简单，但是其中涉及的技术比较非常规，对应用层开发人员来说并不常见。总的来说，其构建过程**与ARM64基本一致**。

## 内核启动

前面的分析其实已经起了个头，按照前面提到的，我们可以找到RISC-V架构的启动协议`Documentation/riscv/boot-image-header.txt`。事实上这个文件只是简单介绍了一下`Image`文件header的结构，更详细的启动协议还是处于TODO状态，我们需要从代码进行分析。前面提到RISC-V内核比较类似于ARM64内核的格式，如下：

```
        u32 code0;                /* Executable code */
        u32 code1;                /* Executable code */
        u64 text_offset;          /* Image load offset, little endian */
        u64 image_size;           /* Effective Image size, little endian */
```

从bootloader的角度，装载并执行`Image`类型的内核只需要做两件事情：

* 将整个`Image`文件放置到内存起始处向后偏移`text_offset`的内存地址
* 跳转到`code0`的地址进行执行

后续的事情，内核自理。我们看到`head.S`中有如下代码：

```asm
ENTRY(_start)
        /*
         * Image header expected by Linux boot-loaders. The image header data
         * structure is described in asm/image.h.
         * Do not modify it without modifying the structure and all bootloaders
         * that expects this header format!!
         */
        /* jump to start kernel */
        j _start_kernel
        /* reserved */
```

很明显`code0`和`code1`放置的就是`j _start_kernel`生成的指令。那么我们就需要从`_start_kernel`开始看起。

### _start_kernel

`_start_kernel`可以看到，一开始主要做了三件事：

1. 关闭所有的中断
2. 设置gp寄存器指向对应的地址（该寄存器为ABI相关，用于存放`__global_pointer$`的地址，即GOT的位置，可以参考[RISCV调用协定](https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md)）
3. 关闭FPU，内核中是不用任何浮点指令的

我们可以看到一个名为`CONFIG_RISCV_M_MODE`的内核配置，该选项启用时，内核默认自己从`Machine`特权级启动。该选项用于内核支持K210 SoC，我们默认该选项没有开启。这里注意，bootloader将控制权交给内核时，`a0`寄存器保存的值即为当前CPU执行单元的ID（RISC-V中称为HART ID）。

`_start_kernel`接下来就开始执行一个简单的`Boot Protocol`，选出一个用于启动内核的CPU，其他CPU进入等待状态。RISC-V处理器在reset之后，所有的处理单元（HART）都会一起开始执行，而Linux内核启动时为主从模型，因此需要挑选出其中一个完成部分内核启动工作，之后告知其他的处理器继续执行开始处理任务。首先确定CPU ID是否合法，即有没有超出内核编译时选择的最大支持CPU数，如果超过则非法：

```asm
#ifdef CONFIG_SMP
        li t0, CONFIG_NR_CPUS
        blt a0, t0, .Lgood_cores
        tail .Lsecondary_park
.Lgood_cores:
#endif
```

`.Lsecondary_park`分支实质上是循环调用`wfi`指令。接下来内核使用一个简单的策略选出用于启动的主CPU：先到先得。`setup.c`中定义了一个原子变量：

```c
atomic_t hart_lottery __section(.sdata);
```

经过codepath的所有CPU都会试图通过原子操作将这个变量加1。RISC-V的原子操作指令会将原子变量的原有值保存在原子变量的目标寄存器中，也就是说，如果操作后目标寄存器的值为0的CPU为第一对其进行操作的CPU。

```asm
        /* Pick one hart to run the main boot sequence */
        la a3, hart_lottery
        li a2, 1
        amoadd.w a3, a2, (a3)
        bnez a3, .Lsecondary_start
```

对于所有竞争失败的CPU，我们在后面进行分析，目前还是顺着主CPU进行分析。随后主CPU的操作基本如下：

* 清空bss段，写为0
* 设置临时内核栈与`task_struct`
* 依次调用`setup_vm`以及`relocate`设置内核的虚拟内存

最后调用C语言通用代码启动内核：

```asm
        /* Start the kernel */
        call soc_early_init
        call parse_dtb
        tail start_kernel
```

### setup_vm与relocate

`setup_vm`函数位于`arch/riscv/mm/init.c`中，其目的是设置一个最小的页表，让内核开启`MMU`并工作在虚拟内存之下，为后续的内存初始化做准备。注意`setup_vm`在被调用时很明显是没有设置好页表的，也就是说`setup_vm`函数生成汇编时的引用必须为`PC-relative`的，由于目前内核在RISC-V体系结构下全局使用`-cmodel=medany`进行编译，因此这一点是满足的。

`setup_vm`函数执行的操作在各个体系结构中没有本质区别，都为设置一个最简单的页表，将内核所在的物理地址映射到PAGE_OFFSET所在虚拟地址区域（即将内核二进制所在的物理地址加上一个PAGE_OFFSET减去\_start所的的偏移量）。因此，这里的任务实质上是建立一个临时页表，即内核临时的虚拟地址空间映射，：

```c
#define MAX_EARLY_MAPPING_SIZE	SZ_128M

pgd_t early_pg_dir[PTRS_PER_PGD] __initdata __aligned(PAGE_SIZE);
```

来挖细节，确定这个临时页表里初始化了哪些东西。首先明确一点，RISC-V支持多种页表结构，我们只分析64位的Sv39，目前内核在RISC-V 64位下就支持这一种模式。随后应该想到，setup_vm的运行环境里并没有初始化任何内存，因此必须静态定义一些变量，预留出一些内存供我们使用。先看`setup_vm`里用到的两个helper函数：`create_pgd_mapping` && `create_pmd_mapping`。

当前在RISC-V下的内核并不支持四级页表，那么自然没有PUD。`early_pmd`数组用于满足在MMU还没有开启时存放PMD表的需求：

```c
#if MAX_EARLY_MAPPING_SIZE < PGDIR_SIZE
#define NUM_EARLY_PMDS		1UL
#else
#define NUM_EARLY_PMDS		(1UL + MAX_EARLY_MAPPING_SIZE / PGDIR_SIZE)
#endif
pmd_t early_pmd[PTRS_PER_PMD * NUM_EARLY_PMDS] __initdata __aligned(PAGE_SIZE);
```

在64位下，`NUM_EARLY_PMDS`始终为1。可以通过`alloc_pmd`函数申请一个PMD表：

```c
static phys_addr_t __init alloc_pmd(uintptr_t va)
{
	uintptr_t pmd_num;

	if (mmu_enabled)
		return memblock_phys_alloc(PAGE_SIZE, PAGE_SIZE);

	pmd_num = (va - PAGE_OFFSET) >> PGDIR_SHIFT;
	BUG_ON(pmd_num >= NUM_EARLY_PMDS);
	return (uintptr_t)&early_pmd[pmd_num * PTRS_PER_PMD];
}
```

这里能很明显看到，在MMU使能之前使用`early_pmd`的空间，而使能之后则使用memblock中分配的内存。`create_pgd_mapping`函数原型如下：

```c
static void __init create_pgd_mapping(pgd_t *pgdp,
				      uintptr_t va, phys_addr_t pa,
				      phys_addr_t sz, pgprot_t prot);
```

这里仔细看实现，不要误解了`va`和`pa`的意思。va是你想进行映射的**虚拟地址**，`pa`是你要往PGD中给这个虚拟地址对应的表项存放的**下一级页表**的**物理地址**。`sz`是你要映射的内存区域的大小，必须小于`PGDIR_SIZE`，可以为下面一级表项可以映射内存区域的长度。如果`create_pgd_mapping`函数发现`sz`小于`PGDIR_SIZE`，则会递归的创建更下一级的页表。

`early_pg_dir`页表内包含两类映射：内核和FIXMAP。其中内核映射是将内核自身处于的连续物理内存区域映射到位于`PAGE_OFFSET`的虚拟内存地址上。而FIXMAP映射并没有映射全部的FIXMAP，而是仅仅影射了FIX_FDT部分，让后续的代码可以访问设备树。

`setup_vm`并不是仅仅只创建了一个`early_pg_dir`页表，还创建了`trampoline_pg_dir`页表。这个页表将`PAGE_OFFSET`后长为`PMD_SIZE`的区域映射到`load_pa`，即内核起始被装载后的起始物理内存地址。`trampoline_pg_dir`页表的作用我们随后就会看到。

分析完`setup_vm`函数后，又回到汇编代码中，这次来看relocate函数。函数只有一个参数`a0`寄存器，用于传递一个页表的物理地址。这个`relocate`函数原理听着很简单，但是真到自己写的时侯则是满满的细节。首先我们知道我们要在该函数中开启MMU，这就说明调用函数时保存的`ra`寄存器，即函数返回地址中的地址已经失效，需要将其修改为对应的虚拟地址。这个原理比较简单，首先通过PC相对寻址得到`_start`的物理地址，然后将其与`PAGE_OFFSET`相减，即得到物理地址与虚拟地址的偏移量。

接下来就到前面设置的`trampoline_pg_dir`上场的时间了，我们需要借助`trampoline`页表通过中断向量的方式从物理地址跳转到虚拟地址，原理如下：

* 首先将中断向量设置到`relocate`函数的后半段需要跳转的地方
* 然后将`trampoline_pg_dir`的地址设置到STVEC寄存器，设置完成MMU被启用，这使得访问原有物理地址时触发异常并跳转到中断向量

事实上，这个`trampoline`是多余的，我不知道上游为什么不删除它，也许是因为CPU热插拔支持的缘故。`relocate`最终会装载`early_pg_dir`页表，此时内核已经位于正确的虚拟地址空间上。

### 临时内核栈与task_struct

回到`_start_kernel`，当`relocate`执行完毕之后，内核需要初始化一个内核线程的运行环境。

```asm
        /* Initialize page tables and relocate to virtual addresses */
        la sp, init_thread_union + THREAD_SIZE
```

这里init_thread_union是通过链接脚本留出的一个PAGE（riscv平台上为4KB）大小的区域，用以充当临时内核栈。随后程序按照riscv的C程序调用协定保存两个Caller需要保存的寄存器，即：

* a0：当前hart（Hardware Thread，即RISCV术语中的最小执行单元，即一个逻辑CPU）的id
* a1：指向设备树的指针

这两个参数都是bootloader传进来的。到达这一步之后，内核调用setup_vm()函数，设置一个基本的页表，进而为后续打开分页机制做准备。

进入内核C语言环境的最后一个步骤是设置`tp`寄存器。我们知道内核初始化时，自身运行的上下文为init_task，即内核的第一个任务（`init/init_task.c`）。该task_struct是静态初始化的，因此我们唯一需要做得就是修改当前CPU的运行上下文，使其认为他当前是在运行init_task任务。内核的进程上下文切换是一个及其架构相关的操作，我们来看看riscv平台是如果进行这个操作的。

```asm
        la tp, init_task
        sw s0, TASK_TI_CPU(tp) # 将当前CPU ID表存在task_struct中
```

如果读过riscv的C语言调用约定一定会知道tp寄存器的存在，事实上文档中并没有写的较为详细。但是现在我们在这里看到在Supervisor模式下的用途：保存当前CPU运行的进程上下文，即`task_struct`结构体。查看`asm/current.h`也可以发现这一事实：

```c
create_pte_mappingstatic __always_inline struct task_struct *get_current(void)
{
        register struct task_struct *tp __asm__("tp");
        return tp;
}
```

