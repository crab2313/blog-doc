+++
title = "Linux内核在RISC-V架构下的spinlock实现"
date = 2020-08-20
draft = true


tags = ["kernel", "risc-v"]

+++

本文分析linux内核下对于spinlock的实现，具体到RISC-V体系结构。由于RISC-V体系结构下目前只是简单的实现了一个基于TAS的最基本的spinlock，本文的另一个附加任务就是分析Linux内核为各个平台下实现spinlock搭建起来的通用框架。

## 基本入口分析

对于内核通用代码来说，spinlock的入口自然是`include/linux/spinlock.h`，所以从这里入手。这个头文件开头的注释可以看到如下信息：

* `asm/spinlock_types.h`定义了`arch_spinlock_t`和`arch_rwlock_t`
* `linux/spinlock_types.h`定义通用类型与其对应的初始化函数
* `asm/spinlock.h`定义`arch_spin_*()`函数
* `linux/spinlock.h`定义通用的接口

当然以上定义都假定当前为SMP系统，很明显对于UP系统，spinlock是不需要的，全部定义为空函数即可。本文的一切分析假定为SMP系统。

我们最开始肯定会碰到一个费解的概念：`raw_spinlock_t`，这个概念需要从`rt-linux`时代说起。对于`rt-linux`，即`realtime linux`，一个对实时性能相当重要的优化就是将`spinning lock`替换为`sleeping lock`。将这个概念讲简单，就是将spinlock替换为mutex，但是很明显不可能完全进行替换，总有一部分spinlock依赖于spinlock特殊的语义。这时就引入了`raw_spinlock_t`，`raw_spinlock_t`就是真正的spinlock实现，而原先的`spinlock_t`的语义发生变化：在`PREEMPT_RT`条件下（即`rt-linux`）为mutex实现，否则由`raw_spinlock_t`实现。归根节底，要进行分析的是`raw_spinlock_t`。

## raw_spinlock_t

`raw_spinlock_t`的定义位于`include/linux/spinlock_types.h`，是真正意义上的通用代码实现：

```c
typedef struct raw_spinlock {
	arch_spinlock_t raw_lock;
#ifdef CONFIG_DEBUG_SPINLOCK
	unsigned int magic, owner_cpu;
	void *owner;
#endif
#ifdef CONFIG_DEBUG_LOCK_ALLOC
	struct lockdep_map dep_map;
#endif
} raw_spinlock_t;
```

可以看到，在关闭这两个调试选项之后，`raw_spinlock_t`本质上就剩下了一个`arch_spinlock_t`，很明显这就是spinlock的架构相关实现。对于这两个调试选项，后续进行单独分析。紧接着可以看到`spinlock_t`的定义：

```c
typedef struct spinlock {
	union {
		struct raw_spinlock rlock;

#ifdef CONFIG_DEBUG_LOCK_ALLOC
# define LOCK_PADSIZE (offsetof(struct raw_spinlock, dep_map))
		struct {
			u8 __padding[LOCK_PADSIZE];
			struct lockdep_map dep_map;
		};
#endif
	};
} spinlock_t；
```

前面提到了`spinlock_t`在`PREEMPT_RT`下应该定义为mutex，这里并没有体现，这是因为我们分析的主线内核没有打上rt补丁。`spinlock_t`的初始化macro平淡无奇，忽略。

内核中的`raw_spinlock_t`相关操作其实有两套实现，其主要区别在于spin状态下是否可以被抢占。内核提供了一个`GENERIC_LOCKBREAK`配置选项用以配置对应架构下对该特性的支持，当架构下该选项开启时，内核使用可以被抢占的spinlock实现。当内核开启部分spinlock相关调试选项时，也会使用可以被抢占的spinlock实现。

这里说明一下可以被抢占的正确性。我们知道一个CPU持有spinlock时，抢占是被关闭的。因此，不会出现一个CPU持有spinlock后，被抢占，新执行的任务又再次尝试持有该spinlock造成死锁的情况出现。

### 可抢占实现

该实现定义在`kernel/locking/spinlock.c`中，且内核使用如下方式切换两种实现：

```c
#if !defined(CONFIG_GENERIC_LOCKBREAK) || defined(CONFIG_DEBUG_LOCK_ALLOC)
/*
 * The __lock_function inlines are taken from
 * spinlock : include/linux/spinlock_api_smp.h
 * rwlock   : include/linux/rwlock_api_smp.h
 */
#else
...
```

该实现的一个优点是可以利用架构相关的relax实现，优化相关操作：

```c
#ifndef arch_read_relax
# define arch_read_relax(l)	cpu_relax()
#endif
#ifndef arch_write_relax
# define arch_write_relax(l)	cpu_relax()
#endif
#ifndef arch_spin_relax
# define arch_spin_relax(l)	cpu_relax()
#endif
```

在没有对应的实现时，使用`cpu_relax()`。`BUILD_LOCK_OPS`宏用于生成我们想要的实现，本质是一个代码模板。我们来分析这个模板：

```c
#define BUILD_LOCK_OPS(op, locktype)					\
void __lockfunc __raw_##op##_lock(locktype##_t *lock)			\
{									\
	for (;;) {							\
		preempt_disable();					\
		if (likely(do_raw_##op##_trylock(lock)))		\
			break;						\
		preempt_enable();					\
									\
		arch_##op##_relax(&lock->raw_lock);			\
	}								\
}									\
```

上面是模板的一部分，生成形如`__raw_spin_lock(raw_spinlock_t *lock)`类似的函数。实现比较直观，注意到它使用了`do_raw_spin_trylock`类似的函数。irqsave版本类似：

```c
unsigned long __lockfunc __raw_##op##_lock_irqsave(locktype##_t *lock)	\
{									\
	unsigned long flags;						\
									\
	for (;;) {							\
		preempt_disable();					\
		local_irq_save(flags);					\
		if (likely(do_raw_##op##_trylock(lock)))		\
			break;						\
		local_irq_restore(flags);				\
		preempt_enable();					\
									\
		arch_##op##_relax(&lock->raw_lock);			\
	}								\
									\
	return flags;							\
}	
```

但是带bh后缀的要注意一下：

```c
void __lockfunc __raw_##op##_lock_bh(locktype##_t *lock)		\
{									\
	unsigned long flags;						\
									\
	/*							*/	\
	/* Careful: we must exclude softirqs too, hence the	*/	\
	/* irq-disabling. We use the generic preemption-aware	*/	\
	/* function:						*/	\
	/**/								\
	flags = _raw_##op##_lock_irqsave(lock);				\
	local_bh_disable();						\
	local_irq_restore(flags);					\
}	\
```

注意这里关闭bottom half时，同样需要关闭硬件中断。再关注一下前缀的区别，即`_raw_`与`__raw`的区别。实际上`_raw`前缀的函数都是由对应`__raw`前缀的函数实现，但是可以通过对应的CONFIG控制是否进行内联操作：

```c
#ifndef CONFIG_INLINE_SPIN_LOCK
void __lockfunc _raw_spin_lock(raw_spinlock_t *lock)
{
	__raw_spin_lock(lock);
}
EXPORT_SYMBOL(_raw_spin_lock);
#endif

#ifdef CONFIG_INLINE_SPIN_LOCK
#define _raw_spin_lock(lock) __raw_spin_lock(lock)
#endi
```

注意这个`__lockfunc`前缀，其定义如下：

```c
#define __lockfunc __attribute__((section(".spinlock.text")))
```

其实际用途就是将对应的函数放到名为`.spinlock.text`的section中，很明显这样同时阻止了编译器将其内联化。最后要注意上面是存在`_raw_spin_lock`这个导出的symbol的。

`do_raw_spin_trylock`其实也有两套实现，debug版本和非debug版本，这里只看非debug版本：

```c
static inline int do_raw_spin_trylock(raw_spinlock_t *lock)
{
	int ret = arch_spin_trylock(&(lock)->raw_lock);

	if (ret)
		mmiowb_spin_lock();

	return ret;
}
```

这里注意`mmiowb_spin_lock`的作用，后续分析I/O操作时会看到其具体用途。之后，可能还有一个疑问，对应的unlock操作到哪里去了？很明显unlock不需要区分是否可以抢占，所以unlock的实现只有一份，在下面会提到。

### 不可抢占实现

前面看到这两个实现是不共存的，因此在`spinlock_api_smp.h`中可以看到不可抢占的实现：

```c
#if !defined(CONFIG_GENERIC_LOCKBREAK) || defined(CONFIG_DEBUG_LOCK_ALLOC)
static inline unsigned long __raw_spin_lock_irqsave(raw_spinlock_t *lock)
{
	unsigned long flags;

	local_irq_save(flags);
	preempt_disable();
	spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
	/*
	 * On lockdep we dont want the hand-coded irq-enable of
	 * do_raw_spin_lock_flags() code, because lockdep assumes
	 * that interrupts are not re-enabled during lock-acquire:
	 */
#ifdef CONFIG_LOCKDEP
	LOCK_CONTENDED(lock, do_raw_spin_trylock, do_raw_spin_lock);
#else
	do_raw_spin_lock_flags(lock, &flags);
#endif
	return flags;
}
...
```

事实上，这套实现与lockdep框架深度集成。