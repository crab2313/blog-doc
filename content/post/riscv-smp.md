+++
title = "RISC-V架构下的SMP实现"
date = 2020-09-19
draft = true


tags = ["kernel", "risc-v", "smp"]

+++

这里SMP这个词比较费解，这是对称多处理的简称。实际上要分析的代码就是Linux内核围绕SMP相关的代码。主要包括：

* RISC-V下的SMP支持
* Linux内核对于SMP支持的通用框架

我们首先来分析Linux内核为SMP支持提供的通用框架。

## Linux内核的SMP框架

SMP是对陈多处理器的意思，那么首先就有一个问题，这样的系统是如何启动的。事实上，Linux内核对于SMP系统默认的启动模型为单CPU启动，称为`boot cpu`具有特殊地位，而其他CPU在内核启动初期处于待机状态。一旦`boot cpu`完成最初的初始化工作，那么就会唤醒其他的CPU，执行相应的任务。

TODO 内核框架入口

随后，我们需要特定的机制，用以进行CPU之间的通信。简单举例：

* `boot cpu`如何唤醒其他CPU
* CPU之间如何发送消息

这些实现都位于`kernel/smp.c`中，等待我们分析。

### 内核API概述

相关API可以在`include/linux/smp.h`中找到。	

### smp_call_function_single

该函数是实现CPU之间通信的一个典型函数，其完成的任务是让特定的CPU执行某一个函数。有了这个函数，就进而可以实现其他的API，如让全部CPU执行特定的函数。函数原型如下：

```c
int smp_call_function_single(int cpuid, smp_call_func_t func, void *info, int wait);
typedef void (*smp_call_func_t)(void *info);
```

各个参数的意义都是很显然的，其中`wait`参数的意义是要求函数等待目标CPU执行完`func`函数。函数开头看到如下数据结构：

```c
/*
 * structure shares (partial) layout with struct irq_work
 */
struct __call_single_data {
	union {
		struct __call_single_node node;
		struct {
			struct llist_node llist;
			unsigned int flags;
		};
	};
	smp_call_func_t func;
	void *info;
};

/* Use __aligned() to avoid to use 2 cache lines for 1 csd */
typedef struct __call_single_data call_single_data_t
	__aligned(sizeof(struct __call_single_data));
```

