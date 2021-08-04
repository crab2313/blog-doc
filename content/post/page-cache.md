+++
title = "Page Cache机制分析"
date = 2020-08-25
draft = true

[taxonomies]
tags = ["kernel", "filesystem", "memory-management"]

+++

* Page Fault路径分析
* Page Cache基本操作



本文试图分析Linux内核的Page Cache机制，其目的是加深对Linux内核的理解，为后续分析块设备的以及文件系统的实现打下基础。Page Cache本质上是内核实现的一套缓存机制，其目的是利用内存缓存对后端更加低速设备的访问操作，这一类设备最常见的就是硬盘、闪存等随机存储设备。

从这个思路出发，我们就要有一个后端设备作为数据的来源与目的地。在内核中并不直接使用块设备作为这个后端设备，而是使用`inode`。将一个inode内的数据作为后端时，需要有一个映射关系，即将inode以0为偏移量的PFN（Page Frame Number）映射到内核的物理page上（即对应的`struct page`）。这个映射关系由`struct address_space`表示，因此我们对Page Cache的分析主要围绕`struct address_space`进行。文档最后对Page Cache的同步机制进行简要分析。

前面提到了`struct inode`、`struct address_space`与`struct page`之间的联系，现在开始从代码中找到对应观点的支撑。首先可以看到`struct inode`之中存在`i_mapping`字段：

```c
struct inode {
    ...
	const struct inode_operations	*i_op;
	struct super_block	*i_sb;
	struct address_space	*i_mapping;
	...
};
```

且`struct address_space`中存在`host`字段：

```c
struct address_space {
	struct inode		*host;
    ...
}  __attribute__((aligned(sizeof(long)))) __randomize_layout;
```

这一对指针互相进行指向，使得`inode`与`address_space`关联了起来。

## address_space

```c
struct address_space {
	struct inode		*host;
	struct xarray		i_pages;
	gfp_t			gfp_mask;
	atomic_t		i_mmap_writable;
#ifdef CONFIG_READ_ONLY_THP_FOR_FS
	/* number of thp, only for non-shmem files */
	atomic_t		nr_thps;
#endif
	struct rb_root_cached	i_mmap;
	struct rw_semaphore	i_mmap_rwsem;
	unsigned long		nrpages;
	unsigned long		nrexceptional;
	pgoff_t			writeback_index;
	const struct address_space_operations *a_ops;
	unsigned long		flags;
	errseq_t		wb_err;
	spinlock_t		private_lock;
	struct list_head	private_list;
	void			*private_data;
} __attribute__((aligned(sizeof(long)))) __randomize_layout;
```

这里首先要理解的就是`a_ops`字段，它是一个`address_space_operations`的结构体，本质上就是内核中很常见的虚函数表。这说明内核中的`Page Cache`（即`address_space`）是分很多种类型的，每一种类型需要有自己独有的`method`。

TODO

## RISC-V对Page Fault处理的实现

以RISC-V为例子，分析Page Fault在RISC-V平台下的实现。当然，与平台相关的代码毕竟是非常少数的，大部分为平台无关的代码。之前的文档其时已经分析过RISC-V平台下的异常处理了，知道RISC-V触发异常之后，异常原因会保存在`CSR_CAUSE`寄存器（这里使用内核的说法，实际上是对应的`mcause`或者`scause`寄存器）。通过`CSR_CAUSE`的值，可以确定异常是否是中断，或者是`Page Fault`。从`arch/riscv/kernel/entry.S`中，可以看到内核对所有类型的`Page Fault`都使用同一个处理函数：

```c
ENTRY(excp_vect_table)
		......
        RISCV_PTR do_trap_ecall_m
        RISCV_PTR do_page_fault   /* instruction page fault */
        RISCV_PTR do_page_fault   /* load page fault */
        RISCV_PTR do_trap_unknown
        RISCV_PTR do_page_fault   /* store page fault */ 
excp_vect_table_end:
END(excp_vect_table)
```

接下来分析`do_page_fault`函数的处理流程。函数传入一个`struct pt_regs`其中保存了处理Page Fault必须的信息。从RISC-V标准可以看到，对于`Page Fault`，其对应触发`Page Fault`的虚拟地址保存在`CSR_TVAL`寄存器中，该寄存器的值会在处理异常的汇编代码中保存在`pt_regs->badaddr`中，如下：

```c
        REG_L s0, TASK_TI_USER_SP(tp)
        csrrc s1, CSR_STATUS, t0
        csrr s2, CSR_EPC
        csrr s3, CSR_TVAL
        csrr s4, CSR_CAUSE
        csrr s5, CSR_SCRATCH
        REG_S s0, PT_SP(sp)
        REG_S s1, PT_STATUS(sp)
        REG_S s2, PT_EPC(sp)
        REG_S s3, PT_BADADDR(sp)
        REG_S s4, PT_CAUSE(sp)
        REG_S s5, PT_TP(sp)
```

同样看到触发异常的原因被保存在了`pt_regs->cause`中。可以看到函数使用了大量的goto语句用以区分多种情况，接下来其作为划分进行分析。

### No Context

No Context翻译成中文就是没有上下文的意思，这个上下文是进行`Page Fault`处理的上下文。没有上下文的`Page Fault`很明显是非法，并且无法处理的。这个上下文最好理解的近义词就是进程地址空间，通过这个地址空间就可以找到地址对应的`VMA`。`No Context`的入口如下：

```c
	/*
	 * If we're in an interrupt, have no user context, or are running
	 * in an atomic region, then we must not take the fault.
	 */
	if (unlikely(faulthandler_disabled() || !mm))
		goto no_context;
```

也就是：

```c
/*
 * Is the pagefault handler disabled? If so, user access methods will not sleep.
 */
static inline bool pagefault_disabled(void)
{
	return current->pagefault_disabled != 0;
}

/*
 * The pagefault handler is in general disabled by pagefault_disable() or
 * when in irq context (via in_atomic()).
 *
 * This function should only be used by the fault handlers. Other users should
 * stick to pagefault_disabled().
 * Please NEVER use preempt_disable() to disable the fault handler. With
 * !CONFIG_PREEMPT_COUNT, this is like a NOP. So the handler won't be disabled.
 * in_atomic() will report different values based on !CONFIG_PREEMPT_COUNT.
 */
#define faulthandler_disabled() (pagefault_disabled() || in_atomic())

#define in_atomic()	(preempt_count() != 0)
```

当当前进程上下文不存在，或者当前处于原子上下文时，又或当前进程显式使用`pagefault_disable()`函数关闭了`Page Fault`处理时，进入`No Context`处理。

```c
no_context:
	/* Are we prepared to handle this kernel fault? */
	if (fixup_exception(regs))
		return;

	/*
	 * Oops. The kernel tried to access some bad page. We'll have to
	 * terminate things with extreme prejudice.
	 */
	bust_spinlocks(1);
	pr_alert("Unable to handle kernel %s at virtual address " REG_FMT "\n",
		(addr < PAGE_SIZE) ? "NULL pointer dereference" :
		"paging request", addr);
	die(regs, "Oops");
	do_exit(SIGKILL);
```

注意`fixup_exception`函数，这里是内核一个`Exception Table`机制的入口。该机制维护一张表，表项如下：

```c
struct exception_table_entry
{
	unsigned long insn, fixup;
};
```

即一个地址->地址的映射关系。如果`fixup_exception`函数发现触发`Page Fault`的地址在这个表中，则将`regs->epc`改成对应的`fixup`值。这就实现了一个钩子跳转机制，内核中很多重要的功能都由该机制实现。后面会系统分析该机制的用途以及实现。

`No Context`处理的后半段比较简单，就是常见的`Oops`流程，在内核日志中打出寄存器，内核栈，并kill掉进程。

### VMalloc Fault

这个类型的`Page Fault`比较好理解，就是简单落到内核vmalloc地址空间的`Page Fault`：

```c
	/*
	 * Fault-in kernel-space virtual memory on-demand.
	 * The 'reference' page table is init_mm.pgd.
	 *
	 * NOTE! We MUST NOT take any locks for this case. We may
	 * be in an interrupt or a critical region, and should
	 * only copy the information from the master page table,
	 * nothing more.
	 */
	if (unlikely((addr >= VMALLOC_START) && (addr <= VMALLOC_END)))
		goto vmalloc_fault;
```

对其的处理也比较简单，首先检查是否是用户态进行的访问，如果是，直接`Segfault`：

```c
		/* User mode accesses just cause a SIGSEGV */
		if (user_mode(regs))
			return do_trap(regs, SIGSEGV, code, addr);
```

随后的工作就比较直观了，我们知道vmalloc分配的内存的页表其是就是保存在`init_mm`上的，这个上下文的页表被称作`Reference Page Table`（参考页表）。也就是说，当一个进程上下文第一次访问对应的内存地址时，会触发`Page Fault`，而这个`Page Fault`的工作就是将`init_mm->pgd`上对应区域的页表拷贝到当前进程上下文的页表中。这就是这里进行的工作。

### Bad Area

如何区分`Good Area`和`Bad Area`呢，它们的本质区别是可不可以找到对应的`backing store`。也就是能不能在对应进程上下文的内存地址空间中找到对应的VMA，使VMA包含该地址。函数首先使用`find_vma`函数查找到进程空间中第一个满足`addr < vm_end`的VMA，然后检查其开头是否小于等于`addr`，如果成立则很明显是`Good Area`。如果上述条件不能够满足，也不能草率认定是`Bad Area`，这是因为内核对进程栈进行了特殊处理：进程栈对应`VMA`使用`VM_GROWSDOWN`进行标记，且可以通过`expend_stack`函数动态进行向下增长。因此RISC-V平台下对应的处理代码如下：

```c
	vma = find_vma(mm, addr);
	if (unlikely(!vma))
		goto bad_area;
	if (likely(vma->vm_start <= addr))
		goto good_area;
	if (unlikely(!(vma->vm_flags & VM_GROWSDOWN)))
		goto bad_area;
	if (unlikely(expand_stack(vma, addr)))
		goto bad_area;
```

对于判定为`Bad Area`的情况，如果`Page Fault`来自用户态，可以简单触发`Sigfault`。否则，内核态的`Page Fault`需要当作`No Context`情况进行处理。

### Good Area

前面看到了如何判断`Page Fault`目标地址是否合法，按照上面方式判断出来的`Good Area`还需要进行二次判断。`VMA`上的`vm_flags`中标记了该VMA的用途，处理`Page Fault`的时候，RISC-V平台对于指令、读、写`Page Fault`进行了明确的区分，可以根据`CSR_CAUSE`寄存器的值进行二次检查，检查不通过的当作`Bad Area`进行处理：

```c
good_area:
	code = SEGV_ACCERR;

	switch (cause) {
	case EXC_INST_PAGE_FAULT:
		if (!(vma->vm_flags & VM_EXEC))
			goto bad_area;
		break;
	case EXC_LOAD_PAGE_FAULT:
		if (!(vma->vm_flags & VM_READ))
			goto bad_area;
		break;
	case EXC_STORE_PAGE_FAULT:
		if (!(vma->vm_flags & VM_WRITE))
			goto bad_area;
		flags |= FAULT_FLAG_WRITE;
		break;
	default:
		panic("%s: unhandled cause %lu", __func__, cause);
	}
```

随后调用内核公共代码`handle_mm_fault`函数处理`Page Fault`。随后只需要处理内存不够以及特定的信号即可，这里要注意`Page Fault`一直重试的情况，必须限定`Page Fault`重试的次数。

## handle_mm_fault

该函数是内核通用函数的入口，用以处理`Page Fault`，所有的架构相关代码都需要调用该函数。函数原型如下：

```c
vm_fault_t handle_mm_fault(struct vm_area_struct *vma, unsigned long address,
			   unsigned int flags, struct pt_regs *regs);
```

其中，`flags`定义如下：

```c
/**
 * Fault flag definitions.
 *
 * @FAULT_FLAG_WRITE: Fault was a write fault.
 * @FAULT_FLAG_MKWRITE: Fault was mkwrite of existing PTE.
 * @FAULT_FLAG_ALLOW_RETRY: Allow to retry the fault if blocked.
 * @FAULT_FLAG_RETRY_NOWAIT: Don't drop mmap_lock and wait when retrying.
 * @FAULT_FLAG_KILLABLE: The fault task is in SIGKILL killable region.
 * @FAULT_FLAG_TRIED: The fault has been tried once.
 * @FAULT_FLAG_USER: The fault originated in userspace.
 * @FAULT_FLAG_REMOTE: The fault is not for current task/mm.
 * @FAULT_FLAG_INSTRUCTION: The fault was during an instruction fetch.
 * @FAULT_FLAG_INTERRUPTIBLE: The fault can be interrupted by non-fatal signals.
 *
 * About @FAULT_FLAG_ALLOW_RETRY and @FAULT_FLAG_TRIED: we can specify
 * whether we would allow page faults to retry by specifying these two
 * fault flags correctly.  Currently there can be three legal combinations:
 *
 * (a) ALLOW_RETRY and !TRIED:  this means the page fault allows retry, and
 *                              this is the first try
 *
 * (b) ALLOW_RETRY and TRIED:   this means the page fault allows retry, and
 *                              we've already tried at least once
 *
 * (c) !ALLOW_RETRY and !TRIED: this means the page fault does not allow retry
 *
 * The unlisted combination (!ALLOW_RETRY && TRIED) is illegal and should never
 * be used.  Note that page faults can be allowed to retry for multiple times,
 * in which case we'll have an initial fault with flags (a) then later on
 * continuous faults with flags (b).  We should always try to detect pending
 * signals before a retry to make sure the continuous page faults can still be
 * interrupted if necessary.
 */
```

函数的返回值是一个bitmask，定义如下：

```c
typedef __bitwise unsigned int vm_fault_t;

/**
 * enum vm_fault_reason - Page fault handlers return a bitmask of
 * these values to tell the core VM what happened when handling the
 * fault. Used to decide whether a process gets delivered SIGBUS or
 * just gets major/minor fault counters bumped up.
 *
 * @VM_FAULT_OOM:		Out Of Memory
 * @VM_FAULT_SIGBUS:		Bad access
 * @VM_FAULT_MAJOR:		Page read from storage
 * @VM_FAULT_WRITE:		Special case for get_user_pages
 * @VM_FAULT_HWPOISON:		Hit poisoned small page
 * @VM_FAULT_HWPOISON_LARGE:	Hit poisoned large page. Index encoded
 *				in upper bits
 * @VM_FAULT_SIGSEGV:		segmentation fault
 * @VM_FAULT_NOPAGE:		->fault installed the pte, not return page
 * @VM_FAULT_LOCKED:		->fault locked the returned page
 * @VM_FAULT_RETRY:		->fault blocked, must retry
 * @VM_FAULT_FALLBACK:		huge page fault failed, fall back to small
 * @VM_FAULT_DONE_COW:		->fault has fully handled COW
 * @VM_FAULT_NEEDDSYNC:		->fault did not modify page tables and needs
 *				fsync() to complete (for synchronous page faults
 *				in DAX)
 * @VM_FAULT_HINDEX_MASK:	mask HINDEX value
 *
 */
```

这些原因需要结合代码一同分析。函数的开头是对计数器的处理：

```c
	count_vm_event(PGFAULT);
	count_memcg_event_mm(vma->vm_mm, PGFAULT);

	/* do counter updates before entering really critical section. */
	check_sync_rss_stat(current);
```

没有什么值的注意的地方。函数根据这个page所在的VMA属性调用`hugetlb_fault`或者`__handle_mm_fault`，很明显对于通过大页进行映射的要进行区分处理。

### __handle_mm_fault

fault handler中反复用到的一个数据结构是`struct vm_fault`：

```c
struct vm_fault {
	struct vm_area_struct *vma;	/* Target VMA */
	unsigned int flags;		/* FAULT_FLAG_xxx flags */
	gfp_t gfp_mask;			/* gfp mask to be used for allocations */
	pgoff_t pgoff;			/* Logical page offset based on vma */
	unsigned long address;		/* Faulting virtual address */
	pmd_t *pmd;			/* Pointer to pmd entry matching
					 * the 'address' */
	pud_t *pud;			/* Pointer to pud entry matching
					 * the 'address'
					 */
	pte_t orig_pte;			/* Value of PTE at the time of fault */

	struct page *cow_page;		/* Page handler may use for COW fault */
	struct page *page;		/* ->fault handlers should return a
					 * page here, unless VM_FAULT_NOPAGE
					 * is set (which is also implied by
					 * VM_FAULT_ERROR).
					 */
	/* These three entries are valid only while holding ptl lock */
	pte_t *pte;			/* Pointer to pte entry matching
					 * the 'address'. NULL if the page
					 * table hasn't been allocated.
					 */
	spinlock_t *ptl;		/* Page table lock.
					 * Protects pte page table if 'pte'
					 * is not NULL, otherwise pmd.
					 */
	pgtable_t prealloc_pte;		/* Pre-allocated pte page table.
					 * vm_ops->map_pages() calls
					 * alloc_set_pte() from atomic context.
					 * do_fault_around() pre-allocates
					 * page table to avoid allocation from
					 * atomic context.
					 */
};
```

从它的注释上能够看到，这个结构体将作为参数传递给对应`VMA`上的`fault`回调函数。

TODO

## Sync系统调用

通过分析`sync`和`syncfs`系统调用，深入理解linux内核的page cache和swap实现。事实上这个系统调用可以拆分称好几个部分，我们先来分析第一个部分，即flusher内核线程。

```c
void ksys_sync(void)
{
        int nowait = 0, wait = 1;

        wakeup_flusher_threads(WB_REASON_SYNC);
        iterate_supers(sync_inodes_one_sb, NULL);
        iterate_supers(sync_fs_one_sb, &nowait);
        iterate_supers(sync_fs_one_sb, &wait);
        iterate_bdevs(fdatawrite_one_bdev, NULL);
        iterate_bdevs(fdatawait_one_bdev, NULL);
        if (unlikely(laptop_mode))
                laptop_sync_completion();
}

SYSCALL_DEFINE0(sync)
{
        ksys_sync();
        return 0;
}
```

## flusher

许多内核书中都会提到pdflusher内核线程，这个内核线程在新的内核中已经消失了。在稍微旧一些的内核中可以找到`flusher:x-y`内核线程，这个内核线程在最新的内核中也看不到了。我们大致知道这个内核线程的作用在特定条件下将内核中的dirty page回写到其backing-store中（一般为block设备）。下面就以它为切入点分析对应的实现。

可以看到`wakeup_flusher_threads`的主要操作为：对于每一个位于`bdi_list`链表中的表项（bakcing_dev_info）向其保存的每个`bdi_writeback`调用`wb_start_writeback`函数。这里一下子出现许多新东西，我们一个一个分析。首先从概念上能够明白`backing_dev_info`用于描述一个`address_space`，即一组page cache的后端存储设备，具体细节先不深究。那么现在则需要明白`bdi_writeback`代表什么。

可以看到`wakeup_flusher_threads`最终调用了`wb_start_writeback`，实质上就是调用了`wb_wakeup`函数，如下：

```c
static void wb_wakeup(struct bdi_writeback *wb)
{
	spin_lock_bh(&wb->work_lock);
	if (test_bit(WB_registered, &wb->state))
		mod_delayed_work(bdi_wq, &wb->dwork, 0);
	spin_unlock_bh(&wb->work_lock);
}
```

实质上是在`bdi_wq`这个`workqueue`上调度起了`wb->dwork`这个`work_struct`，因此可以从`bdi_writeback`的初始化函数`wb_init`中找到`wb->dwork`是如何初始化的。

```c
	INIT_DELAYED_WORK(&wb->dwork, wb_workfn);
```

因此，这个`work_struct`中执行的函数为`wb_workfn`，只需要分析这个函数的行为。函数的行为比较清晰：

* 在`current->flags`中加入`PF_SWAPWRITE`标志位，使当前线程可以写入swap
* 区分两种情况，即内存压力大且`wb->state`中没有`WB_registered`标志时，调用`writeback_inode_wb`函数，回写1024个inode。否则，调用`wb_do_writeback`函数进行彻底的回写。
* 最后，根据`wb`中的`work_list`是否为空，即是否还有没有进行的回写，选择是否立即将`dwork`加入到`workqueue`或者延时一段事件加入`workqueue`。然后擦除调`current->flags`中的`PF_SWAPWRITE`标志。

我们只看`wb_do_writeback`函数的实现，从实现中可以看到`struct wb_writeback_work`用于描述一项回写任务，并被保存在`bdi_writeback`的`work_list`链表中。`wb_do_writeback`函数的处理过程可分为三步：

* 不停的从`work_list`中拿出`wb_writeback_work`，然后调用`wb_writeback`进行处理
* 调用`wb_check_start_all`
* 调用`wb_check_old_data_flush`和`wb_check_background_flush`

### wb_writeback

我们来看这个用于处理`wb_writeback_work`的函数。前面看到，在`wb_do_writeback`函数中，`wb->work_list`上的项目被依次取出，然后被`wb_writeback`处理。`wb_writeback`实质是个会在特定条件下break的死循环：

* `work->nr_pages <= 0`即work中要求的回写page数已经达到
* work为background或者kupdate类型，且`wb->work_list`中还有新的work，这里很明显常规（周期性的）回写操作要让步于刻意触发的回写操作
* 对于background类型的work，如果已经回写到特定的阈值，则停止
* 如果`wb->b_more_io`为空，则停止。这里稍微提一下，`b_more_io`中保存的inode为上次回写操作没有写完而遗留下来的inode。

然后我们来看具体的操作，首先注意一个有可能中途改变的变量：

```c
	oldest_jif = jiffies;
	work->older_than_this = &oldest_jif;
```

这个变量被保存在`work`中，其用意是告诉底层代码，当inode置为dirty时的时间戳早于oldest_jif时，就将其回写。对于background和kupdate类型的work，它的值会实时变动，意在试图回写当前系统中符合条件的所有inode。

```c
		if (work->for_kupdate) {
			oldest_jif = jiffies -
				msecs_to_jiffies(dirty_expire_interval * 10);
		} else if (work->for_background)
			oldest_jif = jiffies;
```

之后，我们需要理解`bdi_writeback`中三个非常重要的队列的用途：

```c
	struct list_head b_dirty;	/* dirty inodes */
	struct list_head b_io;		/* parked for writeback */
	struct list_head b_more_io;	/* parked for more writeback */
	struct list_head b_dirty_time;	/* time stamps are dirty */
	spinlock_t list_lock;		/* protects the b_* lists *
```

在旧的内核中，这几个队列其实是位于superblock对象上的。这也意味着旧的内核中，回写操作是以superblock为单位的，而新内核中，回写是以`bdi_writeback`为工作单位的。这几个队列的解释如下：

* b_dirty保存了已经dirty的inode，即每当inode内缓存在内存中page被修改之后，inode就会被放入这个队列
* b_io保存目前希望进行回写操作的inode，由于inode回写涉及block I/O操作，因此得名
* b_more_io，前面提到了，单次回写操作没有完全清空b_io队列时，会将剩余的部分放入这里。这个队列的用途是防止livelock，即旧的inode一直没有处理完而导致新的inode无法进行处理

循环break掉的逻辑前面已经分析过了，下面是循环运行的逻辑：

1. 如果b_io队列为空，则调用`queue_io`函数向其中增加inode，该函数的逻辑后面进行分析
2. 如果work->sb不为空，即work指定了对应的superblock，则调用`writeback_sb_inodes`，反之调用`__writeback_inodes_wb`
3. 更新带宽信息
4. 如果b_more_io为空，则循环结束，反之则试图等待这个队列中的inode可用，然后回到1执行

来看`queue_io`的实现逻辑，这里实质上就是内核处理旧的inode和新来的inode的逻辑。实际上`queue_io`先将`b_more_io`队列中的元素移动到`b_io`的开头，然后通过检测时间戳将`b_dirty`和`b_dirty_time`队列中的一部分元素放到队列开头。这里注意这个操作有个细节：`b_dirty`和`b_dirty_time`中元素放入`b_io`时会根据superblock进行排序，同一个superblock的会被放到一起。如果`b_io`中加入了新的元素的话，则将`wb->state`加入`WB_has_dirty_io`标志。

### writeback_sb_inodes

上面看到，在`work->sb`不为空时，‵wb_writeback‵会调用`writeback_sb_inodes`，下面就来分析这个函数。首先明确函数的参数，函数的注释上写着会回写一部分位于`sb`上的inode。因此，函数的结构为遍历`wb->b_io`，然后根据情况进行操作，我们注意到一个比较奇怪的判断：

```c
	while (!list_empty(&wb->b_io)) {
		struct inode *inode = wb_inode(wb->b_io.prev);
		struct bdi_writeback *tmp_wb;

		if (inode->i_sb != sb) {
			if (work->sb) {
				/*
				 * We only want to write back data for this
				 * superblock, move all inodes not belonging
				 * to it back onto the dirty list.
				 */
				redirty_tail(inode, wb);
				continue;
			}

			/*
			 * The inode belongs to a different superblock.
			 * Bounce back to the caller to unpin this and
			 * pin the next superblock.
			 */
			break;
		}
```

即`work->sb`是否为空会影响函数后续的行为，在`work->sb`不为空的情况下会将不属于`sb`的inode重新放入`b_dirty`队列，反之则停止处理`b_io`队列上的inode。这是因为一但`work->sb`不为空，则整个work即被限制到`work->sb`上的inode进行回写操作，其他的inode可以忽略。而`work->sb`为空时，work面向的是所有的superblock上的inode，且前面看到了`b_io`上的inode会根据superblock进行分组排序，因此，一旦发现superblock不同了，则需要停止执行，将控制权返回到调用方，让其尝试持有新superblock的锁，然后重新调用`writeback_sb_inodes`进行操作（这就是`__writeback_inodes_wb`函数的逻辑）。

接下来是一些常规情况的处理：

* 处于`I_NEW`、`I_FREEING`或者`I_WILL_FREE`状态的inode会被重新放入`b_dirty`
* 在非WB_SYNC_ALL（即非sync系统调用codepath）情况下，处于`I_SYNC`状态的inode会被放入`b_more_io`中。
* 在WB_SYNC_ALL状态下，函数会等待位于`I_SYNC`状态下inode的回写完成
* 随后调用`__writeback_single_inode`进行回写
* 如果回写完成后inode仍然dirty，则将其再次放入`b_more_io`中

### __writeback_inodes_sb

该函数实现更加简单，即对于每一个`b_io`上的inode，首先通过`inode->i_sb`获取其所属的superblock，然后尝试获取该superblock的锁，最后调用`writeback_sb_inodes`函数回写该superblock上的所有inode。在获取锁失败的情况下，将该inode放入`b_dirty`队列。

## kswapd

kswapd是`page reclaim`的一个重要的机制，在内核中一般被称作`async page reclaim`，即它是异步的，而不是`direct page reclaim`那种同步实现。kswapd本质是将对应过程放到内核线程中实现，这使得它不需要占死CPU资源，且对应codepath中是可以睡眠的。事实上，内核为每一个NUMA节点都创建了对应的kswapd内核线程，因此kswapd实际的名字类如`kswapd%d`，其中`%d`为NUMA节点的编号。在kswapd的初始化函数`kswapd_init`中，可以看到该实现：

```c
static int __init kswapd_init(void)
{
	int nid;

	swap_setup();
	for_each_node_state(nid, N_MEMORY)
 		kswapd_run(nid);
	return 0;
}

int kswapd_run(int nid)
{
	pg_data_t *pgdat = NODE_DATA(nid);
	int ret = 0;

	if (pgdat->kswapd)
		return 0;

	pgdat->kswapd = kthread_run(kswapd, pgdat, "kswapd%d", nid);
	...
}
```

也就是说，分析的重点需要放到`kswapd`函数中，该函数是内核线程内执行的函数。上面可以看到，内核线程传入的指针为对应NUMA节点的`pg_data_t`，可以猜到内核线程与外部是通过该结构体中的特定字段进行交流的。kswapd函数中，首先通过NUMA节点绑定的CPU设置内核线程允许运行的CPU：

```c
	pg_data_t *pgdat = (pg_data_t*)p;
	struct task_struct *tsk = current;
	const struct cpumask *cpumask = cpumask_of_node(pgdat->node_id);

	if (!cpumask_empty(cpumask))
		set_cpus_allowed_ptr(tsk, cpumask);
```

随后设置内核线程`task_struct`上的flags，使其特殊化，即通过特殊的flags标记其为一个`Memory Allocator`，在内核管理代码中具体特殊的身份：

```c
	tsk->flags |= PF_MEMALLOC | PF_SWAPWRITE | PF_KSWAPD;
```

随后来分析kswapd用以与外界通信的两个字段是什么含义：

```c
	WRITE_ONCE(pgdat->kswapd_order, 0);
	WRITE_ONCE(pgdat->kswapd_highest_zoneidx, MAX_NR_ZONES);
```

想当然可以明白，这两个字段的写入方肯定是唤醒kswapd内核线程的一方，即想要进行`page reclaim`的一方。以此为线索，可以找到`wakeup_kswapd`函数，分析该函数，定可明白其与`kswapd`的交互方式。函数原型如下：

```c
void wakeup_kswapd(struct zone *zone, gfp_t gfp_flags, int order,
		   enum zone_type highest_zoneidx);
```

很明显可以看到`zone`参数是需要进行回收的`zone`，主要是要根据它得到对应的NUMA节点的`pd_data_t`进而得到`kswapd`等待的waitqueue。`order`是要回收的连续页面的大小，`highest_zoneidx`是需要进行回收的最高zone索引。

```c
	pgdat = zone->zone_pgdat;
	curr_idx = READ_ONCE(pgdat->kswapd_highest_zoneidx);

	if (curr_idx == MAX_NR_ZONES || curr_idx < highest_zoneidx)
		WRITE_ONCE(pgdat->kswapd_highest_zoneidx, highest_zoneidx);

	if (READ_ONCE(pgdat->kswapd_order) < order)
		WRITE_ONCE(pgdat->kswapd_order, order);
```

原理上是比较字段当前的大小，如果发现比参数小，则进行更新。这里可以注意到一个细节：如果`page reclaim`操作失败次数超过了上限，则会采取措施，进行`direct reclaim`或者唤醒`kcompactd`，整理碎片。函数最后经过一系列检查之后，唤醒`pgdat->kswapd_wait`队列，此时对应NUMA节点上的kswap即被唤醒。

### prepare_kswapd_sleep

该函数仅当kswapd可以睡眠时返回true。函数首先唤醒所有在`pdgat->pfmemalloc_wait`队列里的等待的任务。该机制应该是一个防止throttle的机制，后面有机会会进行分析。随后函数检查当前NUMA节点是否已经超过了最大重试次数，在超过最大重试次数之后，函数直接返回true，让kswapd睡眠。该行为实质上是因为内核认为超过最大重试次数的NUMA节点已经没有救了，放弃治疗，认为其不可能回收出空间了。最后函数调用`pgdat_balanced`函数检查这个NUMA节点已经是`balance`的了，这里的`balance`即`page cache`与`swap`的平衡，即整个`kswapd`执行的操作被称作`balance`。如果当前NUMA节点是已经处于`balance`状态，则返回true让`kswapd`进入睡眠状态。在上面两个条件都不满足的情况下，返回false，让kswapd进入工作状态。

```c
	if (waitqueue_active(&pgdat->pfmemalloc_wait))
		wake_up_all(&pgdat->pfmemalloc_wait);

	/* Hopeless node, leave it to direct reclaim */
	if (pgdat->kswapd_failures >= MAX_RECLAIM_RETRIES)
		return true;

	if (pgdat_balanced(pgdat, order, highest_zoneidx)) {
		clear_pgdat_congested(pgdat);
		return true;
	}

	return false;
```

### pgdat_balanced

该函数检测对于给定的`order`和`highest_zoneidx`，NUMA节点是否处于`balance`状态。这里的`order`是指$2^{order}$连续的page，即NUMA节点是否可以分配出这样一段连续的page。`highest_zoneidx`则限定了这段连续page位于的zone。该函数简单枚举从0到`highest_zoneidx`的所有zone，然后调用`zone_water_mark_ok_safe`函数检测：

```c
	for (i = 0; i <= highest_zoneidx; i++) {
		zone = pgdat->node_zones + i;

		if (!managed_zone(zone))
			continue;

		mark = high_wmark_pages(zone);
		if (zone_watermark_ok_safe(zone, order, mark, highest_zoneidx))
			return true;
	}
```

`zone_watermark_ok_safe`函数本质上是确保zone的buddy allocator可以分配出特定`order`数的连续page。其实现原理也是如此：

```c
	/* For a high-order request, check at least one suitable page is free */
	for (o = order; o < MAX_ORDER; o++) {
		struct free_area *area = &z->free_area[o];
		int mt;

		if (!area->nr_free)
			continue;

		for (mt = 0; mt < MIGRATE_PCPTYPES; mt++) {
			if (!free_area_empty(area, mt))
				return true;
		}
```

### kswapd_try_to_sleep

该函数从名称可以看出是尝试让kswapd进入睡眠状态，函数原型如下：

```c
static void kswapd_try_to_sleep(pg_data_t *pgdat, int alloc_order,
				int reclaim_order, unsigned int highest_zoneidx);
```

从前面的分析可以猜到，`alloc_order`是调用`wakeup_kswapd`的一方想要kswapd回收出来的连续页面，而`reclaim_order`是kswapd上次工作时回收出来的最大连续页面。函数首先处理waitqueue：

```c
	long remaining = 0;
	DEFINE_WAIT(wait);

	if (freezing(current) || kthread_should_stop())
		return;

	prepare_to_wait(&pgdat->kswapd_wait, &wait, TASK_INTERRUPTIBLE);
```

如果允许的话，函数进行一段比较短的睡眠（HZ/10，即0.1秒），在睡眠之前会唤醒kcompactd内核线程。kcompactd内核线程负责内核的内存compact操作，本质上是一类内存碎片整理操作，尝试通过页面数据交换的方式在zone上还原出更大的连续内存。

```c
		/*
		 * We have freed the memory, now we should compact it to make
		 * allocation of the requested order possible.
		 */
		wakeup_kcompactd(pgdat, alloc_order, highest_zoneidx);

		remaining = schedule_timeout(HZ / 10);
```

整个操作分为两个睡眠操作：即第一段的短睡眠和后续的长睡眠。长睡眠不带timeout，只能等待其他人唤醒，且长睡眠只有前面的短睡眠没有被提前唤醒（即到了timeout之后被唤醒）并且`prepare_kswapd_sleep`函数返回true之后才能进行。在长睡眠没有进行时，会根据短睡眠是否被提前唤醒来增加两个事件计数器。

```c
		if (remaining)
			count_vm_event(KSWAPD_LOW_WMARK_HIT_QUICKLY);
		else
			count_vm_event(KSWAPD_HIGH_WMARK_HIT_QUICKLY);
```

这里分析一下这两个计数器增长的条件。首先明确kswapd的行为，即如果kswapd将对应NUMA节点的内存恢复到高水位后，会进行短睡眠，如果短睡眠没有被提前唤醒，则进行长睡眠。`KSWAPD_LOW_WMARK_HIT_QUICKLY`增长的条件是长睡眠没有进行且remaning不为0（即短睡眠正常进行但被提前唤醒），很明显短睡眠能正常进行说明此时NUMA节点位于高水位，但是被提前唤醒说明NUMA节点快速的到达低水位以下。而`KSWAPD_HIGH_WMARK_HIT_QUICKLY`增长的条件为长睡眠没有正常进行且remaing为0（进而说明此时`prepare_kswapd_sleep`返回false），这说明短睡眠没有进行或者正常进行完毕（此时水位可高或者低于高水位），而长睡眠没有进行说明水位低于高水位。这里可能逻辑是没有考虑到短睡眠没有进行的情况，我认为这里有一个内核bug，小概率下短睡眠和长睡眠都没有进行的时候即水位始终低于高水位的时候，`KSWAPD_HIGH_WMARK_HIT_QUICKLY`也会自增，后面有时间找机会验证一下，发个patch。

### balance_pgdat

该函数就是各大内核书中提到的异步`page reclaim`的入口。该函数会在`kswapd`函数中不断被调用，用以`balance`特定NUMA节点的内存。函数入口可以看到内次运行该函数的时候都会增加`PAGEOUTRUN`计数器。

首先理解priority（优先级）的概念，在`page reclaim`上下文中，priority即表示`page reclaim`扫描内存的粒度（totoal_size >> priority），也表示本次扫描的迫切程度，二者实质上是共通的，即越迫切的情况下越要以更小的粒度扫描内存。`balance_pgdat`会从默认priority（DEF_PRIORITY=12）开始，不断以更高的优先级进行`page reclaim`操作，直到满足退出条件。

```c
	sc.priority = DEF_PRIORITY;
	do {
        ......
		if (raise_priority || !nr_reclaimed)
			sc.priority--
    } while (sc.priorty >= 1);
```

循环中会调用`kswapd_shrink_node`进而调用`shrink_node`函数进行`page reclaim`操作，目前暂且将其看作一个黑盒子。而`shrink_node`函数的调用方需要向其传递一个`struct scan_control`接口体，用以控制内存扫描的精确行为，生成此次操作的`scan_control`是`balance_pgdat`的任务之一。

```c
	struct scan_control sc = {
		.gfp_mask = GFP_KERNEL,
		.order = order,
		.may_unmap = 1,
	}；
```

由于是内核线程，故而`gfp_mask`为`GFP_KERNEL`。`order`由函数参数传入。

```c
		sc.reclaim_idx = highest_zoneidx;
```

首先在函数开头就看到是一个boost机制：

```c
	nr_boost_reclaim = 0;
	for (i = 0; i <= highest_zoneidx; i++) {
		zone = pgdat->node_zones + i;
		if (!managed_zone(zone))
			continue;

		nr_boost_reclaim += zone->watermark_boost;
		zone_boosts[i] = zone->watermark_boost;
	}
	boosted = nr_boost_reclaim;
```

其具体实现与`zone->watermark_boost`有关，后续有机会可以分析这一特性。可以看到在NUMA节点处于`balance`状态且没有boost需求时，退出循环：

```c
		/*
		 * If boosting is not active then only reclaim if there are no
		 * eligible zones. Note that sc.reclaim_idx is not used as
		 * buffer_heads_over_limit may have adjusted it.
		 */
		if (!nr_boost_reclaim && balanced)
			goto out;
```

接下来会看到一个比较重要的行为：

```c
		sc.may_writepage = !laptop_mode && !nr_boost_reclaim;
		sc.may_swap = !nr_boost_reclaim;
		if (sc.priority < DEF_PRIORITY - 2)
			sc.may_writepage = 1;
```

即boost模式下不进行page回写以及swap操作。在`laptop_mode`模式下也不进行page回写操作，这自然是尽量阻止磁盘设备被唤醒，提升笔记本设备的续航。但在优先级足够高时，必须允许page回写操作。

### kswapd_shrink_node

该函数实质上是使用`shrink_node`函数进行`page reclaim`操作。首先计算需要进行reclaim的大小：

```c
	sc->nr_to_reclaim = 0;
	for (z = 0; z <= sc->reclaim_idx; z++) {
		zone = pgdat->node_zones + z;
		if (!managed_zone(zone))
			continue;

		sc->nr_to_reclaim += max(high_wmark_pages(zone), SWAP_CLUSTER_MAX);
	}
```

可以看到是传入NUMA节点中所有索引小于`highest_zoneidx`的zone的高水位。

TODO: 为什么是high_wmark_pages，不科学，不应该是high_wmark_pages(zone) - 当前free pages数么

TODO: 这应该要分析`page claim`具体行为才能看出来到底为什么要这样写了吧

## Page Reclaim

### struct lruvec

`struct lruvec`其是就是内核记录page list的单位，正如书上经常提到的`active list`和`inactive list`等都保存在这个结构体中。目前Linux内核在`lruvec`中保存五个page list:

```c
#define LRU_BASE 0
#define LRU_ACTIVE 1
#define LRU_FILE 2

enum lru_list {
	LRU_INACTIVE_ANON = LRU_BASE,
	LRU_ACTIVE_ANON = LRU_BASE + LRU_ACTIVE,
	LRU_INACTIVE_FILE = LRU_BASE + LRU_FILE,
	LRU_ACTIVE_FILE = LRU_BASE + LRU_FILE + LRU_ACTIVE,
	LRU_UNEVICTABLE,
	NR_LRU_LISTS
};
```

注意主要区分`ANON/FILE`和`INAVTIVE/ACTIVE`，它们分别表示list内的page有无文件作为backing store，是否频繁访问。最后一个`UNEVICTABLE`中保存的page是无法被swap out的，关键字:mlock和mlockall。`struct lruvec`的定义如下：

```c
struct lruvec {
	struct list_head		lists[NR_LRU_LISTS];
	/*
	 * These track the cost of reclaiming one LRU - file or anon -
	 * over the other. As the observed cost of reclaiming one LRU
	 * increases, the reclaim scan balance tips toward the other.
	 */
	unsigned long			anon_cost;
	unsigned long			file_cost;
	/* Non-resident age, driven by LRU movement */
	atomic_long_t			nonresident_age;
	/* Refaults at the time of last reclaim cycle, anon=0, file=1 */
	unsigned long			refaults[2];
	/* Various lruvec state flags (enum lruvec_flags) */
	unsigned long			flags;
#ifdef CONFIG_MEMCG
	struct pglist_data *pgdat;
#endif
};
```

其定义本身是比较简单的，但是一定要明白`struct lruvec`是嵌在哪里的。一般来说大部分书上都会提到这几个list是NUMA节点独立的，但事实并非如此，这是因为内核引入了`Memory CGroup`。查看`pg_data_t`的定义，可以发现：

```c
typedef struct pglist_data {
    ......
	/* Fields commonly accessed by the page reclaim scanner */

	/*
	 * NOTE: THIS IS UNUSED IF MEMCG IS ENABLED.
	 *
	 * Use mem_cgroup_lruvec() to look up lruvecs.
	 */
	struct lruvec		__lruvec;

	unsigned long		flags;

	ZONE_PADDING(_pad2_)

	/* Per-node vmstats */
	struct per_cpu_nodestat __percpu *per_cpu_nodestats;
	atomic_long_t		vm_stat[NR_VM_NODE_STAT_ITEMS];
} pg_data_t;
```

可以看到内核大写的警告开发者当`MEMCG`即`Memory CGroup`启用时，这一字段是没有用到的。也就是说，仅有当`CONFIG_MEMCG=n`时，`lruvec`才是保存在`pg_data_t`中的，否则保存在`struct mem_cgroup_per_node`中，即由NUMA节点和`Memory CGroup`共同定位一个`lruvec`。

### shrink_lruvec

该函数原先名称为`shrink_node_memcg`，在内核支持`Memory CGroup`之后改名，其目的是对一个lruvec进行reclaim操作。

TODO

### shrink_list

函数本意是根据给定page list的类型调用`shrink_active_list`或者`shrink_inactive_list`。除此之外，要注意`scan_control`中提供了一个`may_deactivate`字段，其意义目前看起来就是要求不能减小某个active的page list。可以看到：

```c
	if (is_active_lru(lru)) {
		if (sc->may_deactivate & (1 << is_file_lru(lru)))
			shrink_active_list(nr_to_scan, lruvec, sc, lru);
		else
			sc->skipped_deactivate = 1;
		return 0;
	}
```

如果该字段生效且阻止了shrink操作，则`skipped_deactivate`会被置1。

### isolate_lru_pages

这个函数的注释比较详细，这里先粗略讲一下原理。内核中page reclaim是一条非常频繁的codepath，而每次操作LRU的时候都需要持有`pgdat->lru_lock`。这使得这个锁的竞争非常激烈，降低了系统的整体性能。因此，一个最常用的优化就是先持有锁，然后快速将LRU需要进行操作的一部分从LRU上移除，并放到一个本地的list中，然后释放掉锁。由于这个新的list是本地的，因此无需关心同步问题，可以放心操作。`isolate_lru_pages`即为进行这一系列操作的函数，其原型如下：

```c
static unsigned long isolate_lru_pages(unsigned long nr_to_scan,
		struct lruvec *lruvec, struct list_head *dst,
		unsigned long *nr_scanned, struct scan_control *sc,
		enum lru_list lru);
```

参数的介绍在注释中都有，不再赘述。

### shrink_active_list

函数开头定义了三个list：

```c
	LIST_HEAD(l_hold);	/* The pages which were snipped off */
	LIST_HEAD(l_active);
	LIST_HEAD(l_inactive);
```

前面提到了`isolate_lru_pages`进行的优化，事实上`l_hold`即为用于临时存放从LRU上取下的一部分page的list。

```c
	nr_taken = isolate_lru_pages(nr_to_scan, lruvec, &l_hold,
				     &nr_scanned, sc, lru);
```

接下来，对于`l_hold`里的每一个page，都进行如下操作：

1. 调用`cond_resched`函数，主动释放CPU资源，允许被抢占
2. 将page从`l_hold`中取下
3. 如果page被标记为`unevictable`，则调用`putback_lru_page`将其重新放回LRU中
4. 如果page在目标`Memory CGroup`中有进程引用，且位于`VM_EXEC`属性的区域里，且当前list是`FILE`类型的话，将page放入`l_active`中
5. 否则放入`l_inavtive`中。
6. 最后将这几个list放回LRU中。

### shrink_node

无论同步还是异步`page reclaim`操作，其最终入口都是`shrink_node`函数。从名字上看，其目的是对一个NUMA节点中的内存进行`page reclaim`操作。函数的第一个操作是获取要进行操作的`lruvec`：

``` c
	target_lruvec = mem_cgroup_lruvec(sc->target_mem_cgroup, pgdat);
```

也就是说，`scan_control`上的`target_mem_cgroup`可以控制进行操作的`Memory CGroup`。前面看到，一个lruvec是由NUMA节点和`Memory CGroup`共同确定的，一般情况下，如果`target_mem_cgroup`为NULL，则使用`root_mem_cgroup`。

