+++
title = "Linux内核在RISC-V架构下的setup_arch与异常处理"
date = 2020-08-05


tags = ["kernel", "risc-v"]

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

## 进程管理

进程管理与体系结构相关的地方基本就是上下文切换了，在内核一般被称为switch。为了高速进行上下文切换，这个过程涉及的一部分数据结构和相关操作是与体系结构强相关的。**任务**上下文切换的基本执行操作由`__switch_to`完成，一般会再包装一层`switch_to`，现在以这个函数入手，分析RISC-V平台下对应的实现。

`__switch_to`在各个体系结构下有对应的实现，是体系结构强相关的。这是因为其基本行为就是保存当前CPU的执行状态，然后再装载一个其他的执行状态，这其中涉及到的寄存器保存等操作在每个体系结构都不一样。`__switch_to`函数在RISC-V下由汇编实现，位于`arch/riscv/kernel/entry.S`中，如下：

```asm
ENTRY(__switch_to)
        /* Save context into prev->thread */
        li    a4,  TASK_THREAD_RA
        add   a3, a0, a4
        add   a4, a1, a4
        REG_S ra,  TASK_THREAD_RA_RA(a3)
        REG_S sp,  TASK_THREAD_SP_RA(a3)
        REG_S s0,  TASK_THREAD_S0_RA(a3)
        REG_S s1,  TASK_THREAD_S1_RA(a3)
        REG_S s2,  TASK_THREAD_S2_RA(a3)
        ...
        REG_S s11, TASK_THREAD_S11_RA(a3)
        /* Restore context from next->thread */
        REG_L ra,  TASK_THREAD_RA_RA(a4)
        REG_L sp,  TASK_THREAD_SP_RA(a4)
        REG_L s0,  TASK_THREAD_S0_RA(a4)
        REG_L s1,  TASK_THREAD_S1_RA(a4)
        ...
        REG_L s11, TASK_THREAD_S11_RA(a4)
        /* Swap the CPU entry around. */
        lw a3, TASK_TI_CPU(a0)
        lw a4, TASK_TI_CPU(a1)
        sw a3, TASK_TI_CPU(a1)
        sw a4, TASK_TI_CPU(a0)
#if TASK_TI != 0
#error "TASK_TI != 0: tp will contain a 'struct thread_info', not a 'struct task_struct' so get_current() won't work."
        addi tp, a1, TASK_TI
#else
        move tp, a1
#endif
        ret
ENDPROC(__switch_to)

```

实现是比较简单的，这里稍微解读一下：

* `a0`和`a1`分别为进行上下文切换的两个`task_struct`地址
* 需要保存的执行状态被称作`thread_struct`，与体系结构相关，被定义在`asm/processor.h`中。或许有人会奇怪为什么保存的寄存器这么少，只有`ra`，`sp`，`s0`-`s11`。这个解释比较简单，注意它们在RISC-V ABI中被称作`Callee-Saved`寄存器，被调用函数方需要保存的寄存器，反之就是其他寄存器已经被调用方保存过了（栈上）。也就是说`__switch_to`只需要保存这些没有被保存的寄存器，结合栈上原有的寄存器，即可还原处理器状态。
* `tp`寄存器全称`thread pointer`，在内核态中用于保存当前`task_struct`的指针。上下文切换时，需要更改该寄存器。
* 注意`__switch_to`函数有返回值，返回值为`a0`。注意`a0`从头到尾没有变化，正确理解这个行为其实就是正确理解上下文切换的关键。
* 最后一个细节，`TASK_TI_CPU`等常量是哪里来的呢。这个问题本质为C与汇编的互通有无问题，如果想要在汇编中访问结构体的字段，其常规操作为结构体指针加上一个偏移量。`Kbuild`提供了`asm-offsets.h`机制，开发者只需要在`asm-offsets.c`中定义macro和其对应的结构体和结构体中的字段，内核的`Kbuild`即可自行生成`asm-offsets.h`供汇编代码引用。

```c
/* CPU-specific state of a task */
struct thread_struct {
	/* Callee-saved registers */
	unsigned long ra;
	unsigned long sp;	/* Kernel mode stack */
	unsigned long s[12];	/* s[0]: frame pointer */
	struct __riscv_d_ext_state fstate;
};
```

上面是不是少了什么东西？很明显少了浮点和向量寄存器的处理。很多书上其实写明白了，内核中并不使用浮点或者向量指令，因此这两类指令相关的寄存器管理是需要区别对待的。这方面一个比较普遍的优化原理就是内核对用户态进程对于浮点或者向量指令的使用进行检测，仅当用户态使用了时才在上下文切换时记录对应的状态。来看RISC-V是如何实现真正的上下文入口`switch_to`的：

```c
#define switch_to(prev, next, last)			\
do {							\
	struct task_struct *__prev = (prev);		\
	struct task_struct *__next = (next);		\
	if (has_fpu)					\
		__switch_to_aux(__prev, __next);	\
	((last) = __switch_to(__prev, __next));		\
} while (0)
```

和其他体系结构大同小异，`switch_to`是一个宏，它的参数意义大部分书中都有涉及，这里只详细解释一下`last`参数的行为在RISC-V下是如何实现的。首先一定要明确`__prev` 和`__next`这两个变量的声明，这两个变量是保存在**栈上**的，只要栈（指针）发生了改变，那么这两个变量的值就会发生改变。调用`__switch_to`函数时，参数通过`a0`传入，因此`__switch_to`的`a0`一定是上一次上下文切换时的`task_struct`，假设其为A。而当`__switch_to`执行完毕之后，由于栈指针发生了改变，变为A被上下文切换时的栈指针，此时的`__prev`变为了A被上下文切换时处理器执行的前一个任务。

回归重点，即当`has_fpu`为`true`时的`__switch_to_aux`路径。内核提供`CONFIG_FPU`选项配置内核对FPU的支持，在支持FPU的情况下，如果`status`寄存器的`SD`位被设置，说明需要保存FPU状态，此时内核调用`fstate_save`函数将FPU寄存器保存。如果被切换到的任务的浮点寄存器已被使用，则将其状态恢复：

```c
static inline void fstate_restore(struct task_struct *task,
				  struct pt_regs *regs)
{
	if ((regs->status & SR_FS) != SR_FS_OFF) {
		__fstate_restore(task);
		__fstate_clean(regs);
	}
}
```

## 中断、异常与系统调用

`start_kernel`函数的开头会调用`local_irq_disable`将当前CPU的中断处理关闭，这是非常正常的，毕竟这个阶段内核没有做好处理中断的准备，而此时的中断向量也是`head.S`中利用`trampoline`进行虚拟地址跳跃时设置的。RISC-V下的中断向量是`trap_init`中设置的，定义在`arch/riscv/traps.c`中，如下：

```c
void trap_init(void)
{
	/*
	 * Set sup0 scratch register to 0, indicating to exception vector
	 * that we are presently executing in the kernel
	 */
	csr_write(CSR_SCRATCH, 0);
	/* Set the exception vector address */
	csr_write(CSR_TVEC, &handle_exception);
	/* Enable interrupts */
	csr_write(CSR_IE, IE_SIE);
}
```

对于`CSR_SCRATCH`寄存器的设计用途，RISC-V的手册中讲的不明所以，这在后续的代码分析中就会明晰。`traps_init`函数随后将中断向量设置为`handle_exception`函数的地址，并将软件中断打开。在处理器不同状态之间跳转的代码一般实现在体系结构对应的`entry.S`中，RISC-V也不例外。这里注意`CSR_TVEC`的最低两位是中断处理模式，由于RISC-V指令为4字节对齐，那么这个模式位必为0，也就是所有的中断都会由`handle_exception`处理。

传统意义上的I/O中断，异常，以及系统调用都由`handle_exception`进行处理，这就意味着该函数的实现相对复杂，需要好好梳理。从这个函数中我们也可以分析出`CSR_SCRATCH`寄存器是如何使用的。Linux内核使用`CSR_SCRATCH`寄存器保存当前特权级对立特权级（内核态对应用户态，用户态对应内核态）的`tp`寄存器的值，且如果`CSR_SCRATCH`为0则表示中断是在内核态触发的。函数开头如下：

```asm
ENTRY(handle_exception)
	/*
	 * If coming from userspace, preserve the user thread pointer and load
	 * the kernel thread pointer.  If we came from the kernel, the scratch
	 * register will contain 0, and we should continue on the current TP.
	 */
	csrrw tp, CSR_SCRATCH, tp
	bnez tp, _save_context

_restore_kernel_tpsp:
	csrr tp, CSR_SCRATCH
	REG_S sp, TASK_TI_KERNEL_SP(tp)
_save_context:
	...
```

即先将当前`tp`的值与`CSR_SCRATCH`的值进行交换，如果发现`CSR_SCRATCH`的值为0，则明显是由内核态跳入异常处理的，此时需要将`tp`寄存器的值还原，并将此时的内核栈指针保存在`struct thread_info`的`kernel_sp`字段中。从函数中可以看到，`thread_info->kernel_sp`的值在每次进入异常处理时都会被写掉，但由于`handle_exception`记录了`pt_regs`，因此异常退出时该值可以被还原。

`_save_context`本质上是在栈上保存一个`pt_regs`，不赘述。随后`CSR_SCRATCH`被写0用以标记内核态，如前面所述。接下来装载`gp`寄存器，这对C语言运行环境是至关重要的。

```asm
	/* Load the global pointer */
.option push
.option norelax
	la gp, __global_pointer$
.option po
```

注意`norelax`属性可以要求链接器不对标记区域的指令进行重排优化操作。

### 中断

可以通过`CSR_CAUSE`的最高位确定当前是否是中断，所以有如下实现：

```asm
	la ra, ret_from_exception
	/*
	 * MSB of cause differentiates between
	 * interrupts and exceptions
	 */
	bge s4, zero, 1f

	/* Handle interrupts */
	move a0, sp /* pt_regs */
	tail do_IRQ
```

函数`do_IRQ`的返回地址被设置成了`ret_from_exception`，如下：

```asm
ret_from_exception:
	REG_L s0, PT_STATUS(sp)
	csrc CSR_STATUS, SR_IE
#ifdef CONFIG_RISCV_M_MODE
	/* the MPP value is too large to be used as an immediate arg for addi */
	li t0, SR_MPP
	and s0, s0, t0
#else
	andi s0, s0, SR_SPP
#endif
	bnez s0, resume_kernel
resume_userspace:
```

通过`CSR_STATUS`上的`SPP`位可以获取到`handle_exception`跳转执行时原先CPU位于的特权级，对于内核态和用户态需要区分对待。实际上这两者最本质的区别就是对抢占的处理，`resume_kernel`会根据内核编译时是否支持抢占执行对应的操作：如果支持抢占，则检查当前任务的`preempt_count`和`TIF_NEED_RESCHED`标志，进行抢占操作，反之则直接返回。注意这里这个`ret_from_exception`是由多个路径共享的。

`do_IRQ`中根据`CSR_CAUSE`的值确定中断来源：

```c
	switch (regs->cause & ~CAUSE_IRQ_FLAG) {
	case RV_IRQ_TIMER:
		riscv_timer_interrupt();
		break;
#ifdef CONFIG_SMP
	case RV_IRQ_SOFT:
		/*
		 * We only use software interrupts to pass IPIs, so if a non-SMP
		 * system gets one, then we don't know what to do.
		 */
		riscv_software_interrupt();
		break;
#endif
	case RV_IRQ_EXT:
		handle_arch_irq(regs);
		break;
	default:
		pr_alert("unexpected interrupt cause 0x%lx", regs->cause);
		BUG();
	}
```

其中，`handle_arch_irq`是中断控制器驱动通过调用`set_handle_irq`注册的中断处理函数。Linux内核中对于RISC-V专门实现了对应的时钟源：

```c
/* called directly from the low-level interrupt handler */
void riscv_timer_interrupt(void)
{
	struct clock_event_device *evdev = this_cpu_ptr(&riscv_clock_event);

	csr_clear(CSR_IE, IE_TIE);
	evdev->event_handler(evdev);
}
```

而`riscv_software_interrupt`只用于处理IPI中断，有机会的话单独分析。

### 系统调用

系统调用的实现非常简单，如果`handle_exception`判断进入异常的原因为系统调用，则会：

* 检查系统调用号合法性
* 处理系统调用tracer
* 通过`sys_call_table`和系统调用号拿到系统调用的函数指针，并跳转执行

```asm
	li t1, -1
	beq a7, t1, ret_from_syscall_rejected
	blt a7, t1, 1f
	/* Call syscall */
	la s0, sys_call_table
	slli t0, a7, RISCV_LGPTR
	add s0, s0, t0
	REG_L s0, 0(s0)
1:
	jalr s0
```

### 其他异常

除了系统调用和中断之外的异常处理通过`excp_vect_table`之中注册的函数指针完成：

```asm
	/* Handle other exceptions */
	slli t0, s4, RISCV_LGPTR
	la t1, excp_vect_table
	la t2, excp_vect_table_end
	move a0, sp /* pt_regs */
	add t0, t1, t0
	/* Check if exception code lies within bounds */
	bgeu t0, t2, 1f
	REG_L t0, 0(t0)
	jr t0
```

