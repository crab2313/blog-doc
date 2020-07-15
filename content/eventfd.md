+++
title = "eventfd在内核中的实现"
date = 2017-12-19

[taxonomies]
tags = ["kernel", "eventfd"]

+++

eventfd是一个利用匿名文件描述符实现“等待/通知”通信机制的一种方式。它比较方便的一点是，eventfd不仅可以实现用户态与用户态之间的通信，也可以实现内核与用户态的通信。eventfd的实现比较简单易懂，主要在以下两个文件中：

* include/linux/eventfd.h
* fs/eventfd.c

而关于eventfd的详细说明可以参考man-pages：

```shell
man eventfd
```

由于eventfd的实现很简短（只有500行左右），下面详细地分析在内核中的实现。

# 在内核中的表示

很明显，eventfd需要一个保存特定的状态，内核使用eventfd_ctx来保存一个eventfd的状态信息。

```c
struct eventfd_ctx {
        struct kref kref;
        wait_queue_head_t wqh;
        __u64 count;
        unsigned int flags;
};
```

可以看到内核使用了一个kref来作为引用计数。使用一个64位的count字段来保存eventfd的内部计数器，使用flags字段保存eventfd的标志位。作为一个“等待/通知”机制，等待和唤醒操作是必不可少的，内核中实现这个操作的标准方式是用一个waitqueue，即wqh字段。

# 用户态接口

从man-pages中可以了解到，内核中提供了两个系统调用：eventfd和eventfd2。当eventfd2存在时，glibc默认使用eventfd2系统调用。这两个系统调用的区别是，eventfd2允许传入flags参数，而eventfd系统调用强制设置flags为0。在`fs/eventfd.c`文件的末尾，我们可以找到这两个系统调用。

```c
SYSCALL_DEFINE2(eventfd2, unsigned int, count, int, flags)
{        int fd, error;        struct file *file;
        error = get_unused_fd_flags(flags & EFD_SHARED_FCNTL_FLAGS);
        if (error < 0)                return error;
        fd = error;
        file = eventfd_file_create(count, flags);
        if (IS_ERR(file)) {
                error = PTR_ERR(file);
                goto err_put_unused_fd;
        }
        fd_install(fd, file);

        return fd;

err_put_unused_fd:
        put_unused_fd(fd);

        return error;
}

SYSCALL_DEFINE1(eventfd, unsigned int, count)
{
        return sys_eventfd2(count, 0);
}
```

简单观察可以发现eventfd是eventfd2的简单包装（将flags设置为0），所以我们集中分析eventfd2做了什么。首先调用`get_unused_fd_flags`在当前进程中找来一个未使用的文件描述符并返回，同时设置其标志位。可以从eventfd.h文件中发现，`EFD_SHARED_FCNTL_FLAGS`实际上的定义如下：

```C
#define EFD_CLOEXEC			O_CLOEXEC
#define EFD_NONBLOCK                    O_NONBLOCK
#define EFD_SHARED_FCNTL_FLAGS		(O_NONBLOCK | O_CLOEXEC)
```

也就是说，eventfd的`EFD_NONBLOCK`和`EFD_CLOEXEC`与fcntl对应的`O_*`标志位是公用的。然后eventfd2系统调用又调用了`eventfd_file_create`创建了struct file结构体，最后使用fd_install函数将这个struct file结构体关联到当前的任务中。最后所有的问题都集中到了`eventfd_file_create`函数中，忽略掉一些sanity check，可以在`eventfd_file_create`函数中看到以下代码：

```c
struct file *eventfd_file_create(unsigned int count, int flags)
{
        struct file *file;
        struct eventfd_ctx *ctx;
        ......
        ctx = kmalloc(sizeof(*ctx), GFP_KERNEL);
        ......
        file = anon_inode_getfile("[eventfd]", &eventfd_fops, ctx,
                                  O_RDWR | (flags & EFD_SHARED_FCNTL_FLAGS));
        ......
        return file;
}
```

即使用了比较常见的`anon_inode_getfile`函数创建了一个匿名的inode，打开这个inode得到struct file结构体，对应的fops为eventfd_fops。接下来分析这VFT中注册的回调函数的行为。

# eventfd的回调函数

紧接上一节，我们得到了一个名为`eventfd_fops`的`file_operations`，如下：

```c
static const struct file_operations eventfd_fops = {

#ifdef CONFIG_PROC_FS
        .show_fdinfo    = eventfd_show_fdinfo,
#endif
        .release        = eventfd_release,
        .poll           = eventfd_poll,
        .read           = eventfd_read,
        .write          = eventfd_write,
        .llseek         = noop_llseek,
};
```

.llseek是一个空的占位函数，可以不去理会。

# release

```c
void eventfd_ctx_put(struct eventfd_ctx *ctx)
{
        kref_put(&ctx->kref, eventfd_free);
}

static int eventfd_release(struct inode *inode, struct file *file)
{
        struct eventfd_ctx *ctx = file->private_data;
        wake_up_poll(&ctx->wqh, POLLHUP);
        eventfd_ctx_put(ctx);
        return 0;
}
```

还是比较好懂的，首先使用POLLHUP作为key调用`wake_up_poll`唤醒waitqueue，然后将`eventfd_ctx`的引用计数减一。如果引用计数为0，那么就调用kfree直接释放掉`eventfd_ctx`占用的内存。

# 读取操作

`eventfd_read`回调函数中并没有做太多事情，简单的检查了参数之后，就将所有工作委托给了`eventfd_ctx_read`函数。现在先描述一下这个函数的行为，然后解释它的实现细节。

函数首先根据传进来的no_wait参数确定是否应该做阻塞操作，阻塞停止的判断标准是`ctx->count > 0`，也就是如果eventfd内部的计数器为大于0的话，就会停止阻塞。随后函数调用`eventfd_ctx_do_read`更新eventfd计数器的值，并根据ctx->wqh是否正常工作以POLLOUT为参数调用`wake_up_locked_poll`函数通知还在ctx->wqh上做poll操作的任务。

`eventfd_ctx_do_read`的行为比较简单：如果ctx->flags中有EFD_SEMAPHORE标志，那么就将ctx->count减去1，否则就清零。

```c
static void eventfd_ctx_do_read(struct eventfd_ctx *ctx, __u64 *cnt)
{
        *cnt = (ctx->flags & EFD_SEMAPHORE) ? 1 : ctx->count;
        ctx->count -= *cnt;
}
```

可以看到`eventfd_ctx`中没有自己的锁，所以它用的是wqh.lock这个spinlock，这个spinlock是与waitqueue共用的，所以`eventfd_ops.poll`操作的实现需要考虑很多data race出现的情况。为了实现等待操作，函数首先将自身任务放入ctx->wqh中，然后进入一个死循环，在循环开始将自身运行状态设置为`TASK_INTERRUPTIBLE`，这标志着任务进入等待状态，并可以接收信号。接下来检查ctx->count是否大于0，如果是则宣告等待结束。最后检测当前任务是否有到来的信号，如果有，那么此次等待操作失败，返回-ERESTARTSYS，文件系统层会重新进行此次操作。注意我们是不能拿着spinlock做上下文切换的，所以调用schedule前应该要把spinlock放回去，在schedule返回的时候再重新持有spinlock。死循环结束时，将当前任务从ctx->wqh中移除，并重置当前运行状态为TASK_RUNNING。

# 写入操作

`eventfd_write`的实现和`eventfd_read`非常相似，没有什么需要过多解释的细节。`ctx->count`可以保存的最大值是0xfffffffffffffffe即`UULONG_MAX-1`，每次`eventfd_write`操作会增加`ctx->count`的值，如果ctx->count的值增加之后会超过`UULONG_MAX-1`那么`eventfd_write`会根据`O_NONBLOCK`是否设置决定是阻塞等待还是直接返回`-EAGAIN`。

# POLL操作

`eventfd_poll`函数看起来是最短最简单的一个，其实是最复杂的一个。前面提到，`eventfd_ctx`直接将`ctx->wqh.lock`当作自己的锁，这与poll操作是冲突的，因为`poll_wait函数`会先获取`ctx->wqh.lock`。为了保证函数功能正确，必须考虑所有情况。

```c
static unsigned int eventfd_poll(struct file *file, poll_table *wait)
{
        struct eventfd_ctx *ctx = file->private_data;
        unsigned int events = 0;
        u64 count;

        poll_wait(file, &ctx->wqh, wait);
        count = READ_ONCE(ctx->count);

        if (count > 0)
                events |= POLLIN;
        if (count == ULLONG_MAX)
                events |= POLLERR;
        if (ULLONG_MAX - 1 > count)
                events |= POLLOUT;

        return events;
}
```

首先可以确定的是，`eventfd_poll`不能像read和write那样在开头获取`ctx->wqh.lock`。原因前面提到了，`poll_wait`里面调用了`add_wait_queue`，也会去拿这个spinlock。在已经持有锁的情况下再去获取这个锁，就会造成死锁。从代码中可以看到，从`poll_wait`以下的代码是没有锁保护的，这就需要确定所有情况保证代码不出现竞争。

`READ_ONCE`是一个特殊的宏，可以保证对`ctx->count`的读取有且只有一次。由于指令重排的作用，这一行：

```c
        count = READ_ONCE(ctx->count);
```

可能会向上移动到`poll_wait`中的`add_wait_queue`的临界区里，但是不会移动到临界区之上。这是因为spinlock在ACCQUIRE操作时的语义隐含：所有在持有操作之后的内存操作都会在持有操作之后完成。翻译成白话就是在获取spinlock之后做的内存操作都不会重排到获取spinlock操作之前。可能出现的竞争为：

```c
         *     poll                               write
         *     -----------------                  ------------
         *     lock ctx->wqh.lock (in poll_wait)
         *     count = ctx->count
         *     __add_wait_queue
         *     unlock ctx->wqh.lock
         *                                        lock ctx->qwh.lock
         *                                        ctx->count += n
         *                                        if (waitqueue_active)
         *                                          wake_up_locked_poll
         *                                        unlock ctx->qwh.lock
         *     eventfd_poll returns 0
```

这个是安全的，因为write的`wake_up_locked_poll`函数还会把在poll的任务唤醒一次。

# 内核通知机制

内核可以使用`eventfd_signal`函数从内核一侧实现write操作，可以使用`eventfd_etx_remove_wait_queue`实现内核端的read操作，代码比较简单：

```c
__u64 eventfd_signal(struct eventfd_ctx *ctx, __u64 n)
{
        unsigned long flags;

        spin_lock_irqsave(&ctx->wqh.lock, flags);
        if (ULLONG_MAX - ctx->count < n)
                n = ULLONG_MAX - ctx->count;
        ctx->count += n;
        if (waitqueue_active(&ctx->wqh))
                wake_up_locked_poll(&ctx->wqh, POLLIN);
        spin_unlock_irqrestore(&ctx->wqh.lock, flags);

        return n;
}
int eventfd_ctx_remove_wait_queue(struct eventfd_ctx *ctx, wait_queue_t *wait,
                                  __u64 *cnt)
{
        unsigned long flags;

        spin_lock_irqsave(&ctx->wqh.lock, flags);
        eventfd_ctx_do_read(ctx, cnt);
        __remove_wait_queue(&ctx->wqh, wait);
        if (*cnt != 0 && waitqueue_active(&ctx->wqh))
                wake_up_locked_poll(&ctx->wqh, POLLOUT);
        spin_unlock_irqrestore(&ctx->wqh.lock, flags);

        return *cnt != 0 ? 0 : -EAGAIN;
}
```