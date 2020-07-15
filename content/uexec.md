+++
title = "TTM内存分配器分析"
date = 2020-04-26
draft = true

[taxonomies]
tags = ["kernel", "drm"]

+++

# execve系统调用分析

本文分析exece系统调用，用以为后续uexec实现打下基础，主要包括：

* 系统调用内部实现机制及框架
* ELF格式可执行程序的装载
* 用户态进程开始执行时的运行环境

## execve实现细节分析

execve系统调用实现在`fs/exec.c`中，本文分析的是3.18内核。

```c
YSCALL_DEFINE3(execve,
                const char __user *, filename,
                const char __user *const __user *, argv,
                const char __user *const __user *, envp)
{
        return do_execve(getname(filename), argv, envp);
}
```

而`do_execve`简单封装了一下参数后就调用`do_execve_common`函数。函数实质上分为两部分：

* 构建`struct linux_bprm`结构体
* 调用`exec_binprm`

首先要注意一点就是linux内核支持的可执行程序格式可不仅仅只有ELF，我们常见的脚本文件也算是一种可执行格式。当然，内核还支持许多其他的可执行格式，当然，使用过wine的或者qemu-user-static的人应该知道内核支持在用户态动态注册可执行程序格式。因此，`struct linux_bprm`摆明了就是用于抽象各个可执行程序格式装载执行时的通用参数。

```c
struct linux_binprm {
        char buf[BINPRM_BUF_SIZE];
#ifdef CONFIG_MMU
        struct vm_area_struct *vma;
        unsigned long vma_pages;
#else
# define MAX_ARG_PAGES  32
        struct page *page[MAX_ARG_PAGES];
#endif
        struct mm_struct *mm;
        unsigned long p; /* current top of mem */
        unsigned int
                cred_prepared:1,/* true if creds already prepared (multiple
                                 * preps happen for interpreters) */
                cap_effective:1;/* true if has elevated effective capabilities,
                                 * false if not; except for init which inherits
                                 * its parent's caps anyway */
#ifdef __alpha__
        unsigned int taso:1;
#endif
        unsigned int recursion_depth; /* only for search_binary_handler() */
        struct file * file;
        struct cred *cred;      /* new credentials */
        int unsafe;             /* how unsafe this exec is (mask of LSM_UNSAFE_*) */
        unsigned int per_clear; /* bits to clear in current->personality */
        int argc, envc;
        const char * filename;  /* Name of binary as seen by procps */
        const char * interp;    /* Name of the binary really executed. Most
                                   of the time same as filename, but could be
                                   different for binfmt_{misc,script} */
        unsigned interp_flags;
        unsigned interp_data;
        unsigned long loader, exec;
};
```

`do_execve_common`函数首先检查当前用户创建的进程是否已经超出限额：

```c
        if ((current->flags & PF_NPROC_EXCEEDED) &&
            atomic_read(&current_user()->processes) > rlimit(RLIMIT_NPROC)) {
                retval = -EAGAIN;
                goto out_ret;
        }
```

### bprm_mm_init

bprm_mm_init函数中会初始化新进程的地址空间，即申请一个`mm_struct`。由于很明显这里还是位于通用代码部分，所以是不进行装载操作的。可以看到，这里除了申请空白`mm_struct`之外就是向其中插入一个vm_area，当作栈用，如下：

```c
        vma->vm_end = STACK_TOP_MAX;
        vma->vm_start = vma->vm_end - PAGE_SIZE;
        vma->vm_flags = VM_SOFTDIRTY | VM_STACK_FLAGS | VM_STACK_INCOMPLETE_SETUP;
        vma->vm_page_prot = vm_get_page_prot(vma->vm_flags);
        INIT_LIST_HEAD(&vma->anon_vma_chain);

        err = insert_vm_struct(mm, vma);
```

`STACK_TOP_MAX`是一个平台相关的宏，在x86_64下定义如下：

```c
#define STACK_TOP_MAX           TASK_SIZE_MAX
/*
 * User space process size. 47bits minus one guard page.
 */
#define TASK_SIZE_MAX   ((1UL << 47) - PAGE_SIZE)
```

栈的大小定义为一个page。这里注意这一行：

```c
        bprm->p = vma->vm_end - sizeof(void *);
```

后续操作会向这个栈里填充数据，即execve传入的程序名称，参数，以及环境变量。

### prepare_binprm

该函数检查可执行文件的文件系统相关信息，作出相应操作。首先是检查可执行程序是否具有sgid或者suid位，然后设置对应的creds。随后拷贝可执行程序的开头128字节（新内核为256字节）到`linux_brpm->buf`缓冲区中备用。

### exec_binprm

这个函数本质上就是调用`search_binary_handler`，遍历所有支持的可执行格式，然后调用其`load_binary`函数指针。如果可执行格式能够识别这个文件，则`load_binary`调用成功。

## ELF格式文件的装载

内核中每一类可执行文件都对应一个`struct linux_binfmt`，ELF对应的`binfmt`实现在`fs/binfmt_elf.c`中。

```c
static struct linux_binfmt elf_format = {
        .module         = THIS_MODULE,
        .load_binary    = load_elf_binary,
        .load_shlib     = load_elf_library,
        .core_dump      = elf_core_dump,
        .min_coredump   = ELF_EXEC_PAGESIZE,
};
```

### load_elf_binary

函数首先拷贝整个elf header出来，然后检查其MAGIC：

```c
        loc->elf_ex = *((struct elfhdr *)bprm->buf);

        retval = -ENOEXEC;
        /* First of all, some simple consistency checks */
        if (memcmp(loc->elf_ex.e_ident, ELFMAG, SELFMAG) != 0)
                goto out;
```

如果ELF文件类型既不是`ET_EXEC`也不是`ET_DYN`则不认为它可以执行，对于非当前架构的ELF文件，也认为其不可执行。在做完简单检查之后，函数开始读取该ELF文件的Program Header，并进行处理：

* 对于对于PT_INERP类型Program Header，即动态链接器，函数打开链接器的ELF文件，并读取其ELF Header。检查其ELF文件格式的合法性。
* 对于`PT_GNU_STACK`类型的Program Header，函数读取其`p_flags`中是否有`PF_X`标志位，以此决定栈空间所占内存是否标记为可执行内存。