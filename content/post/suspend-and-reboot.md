+++
title = "Linux内核对挂起的实现"
date = 2020-09-07
draft = true


tags = ["kernel", "power-management"]

+++

本文分析内核关于重启与挂起的实现，意图加深对Linux内核的理解，希望对调试相关代码有所帮助。阅读本文前，建议先阅读完相应的[内核文档](https://www.kernel.org/doc/html/latest/driver-api/pm/devices.html)。读完该内核文档后应当对内核的电源管理有所认知。

## 用户态入口

这一部分在用户态的入口为`/sys/power`，具体到挂起操作，则为`/sys/power/state`。对于代码的分析也理应从这里入手。内核中与电源管理框架相关的代码都放置于`kernel/power/`文件夹下，而与`sysfs`相关的代码则为`kernel/power/main.c`。从中可以找到对应kobject属性定义：

```c
#define power_attr(_name) \
static struct kobj_attribute _name##_attr = {	\
	.attr	= {				\
		.name = __stringify(_name),	\
		.mode = 0644,			\
	},					\
	.show	= _name##_show,			\
	.store	= _name##_store,		\
}

power_attr(state);
```

从`state_show`中可以看到，系统当前支持的挂起状态被保存在一个数组中：

```c
#ifdef CONFIG_SUSPEND
	suspend_state_t i;

	for (i = PM_SUSPEND_MIN; i < PM_SUSPEND_MAX; i++)
		if (pm_states[i])
			s += sprintf(s,"%s ", pm_states[i]);

#endif
```

而`state_store`则是用户态的功能入口，简单来说`store_decode`负责解析字符串，然后将其传递给`pm_suspend`函数，或者`hibernate`函数。可以看到系统suspend的状态是分别定义的，且为`suspend_state_t`类型：

```c
typedef int __bitwise suspend_state_t;

#define PM_SUSPEND_ON		((__force suspend_state_t) 0)
#define PM_SUSPEND_TO_IDLE	((__force suspend_state_t) 1)
#define PM_SUSPEND_STANDBY	((__force suspend_state_t) 2)
#define PM_SUSPEND_MEM		((__force suspend_state_t) 3)
#define PM_SUSPEND_MIN		PM_SUSPEND_TO_IDLE
#define PM_SUSPEND_MAX		((__force suspend_state_t) 4)
```

## pm_suspend

前面看到`pm_suspend`函数是实现功能的入口，该函数位于`kernel/power/suspend.c`中。函数除了简单的sanity check以及错误记录汇报之外，仅仅简单的调用了`enter_state`函数。下面就来分析这个函数。

### enter_state

函数必须确保自己是唯一进行suspend操作的实例，这通过`system_transition_mutex`实现，如果获取失败，则返回`-EBUSY`。我们这里只分析`Suspend to RAM`路径，即`state == PM_SUSPEND_MEM`的情况。因为其他类型的suspend相比于`Suspend to RAM`只是少做了几步操作而以。对于`PM_SUSPEND_MEM`来说，整个操作分成两部分：`suspend_prepare`和`suspend_devices_and_enter`。

### suspend_prepare

`suspend_prepare`实现了各种suspend类型都需要进行的一部分操作。简单来说就是进行事件通知与用户态冻结。函数先调用`sleep_state_supported`函数检测当前系统是否支持进入sleep状态。检测方式及比较简单，逻辑如下：

```c
static bool sleep_state_supported(suspend_state_t state)
{
	return state == PM_SUSPEND_TO_IDLE || (suspend_ops && suspend_ops->enter);
}
```

可以看到`PM_SUSPEND_TO_IDLE`事实上是纯软件的实现，只是冻结用户态，无需平台硬件的辅助。而其他的suspend类型则需要平台提供一个`suspend_ops`。

```c
static const struct platform_suspend_ops *suspend_ops;
```

很明显，我们现在最常见的就是ACPI了，后续会基于ACPI进行相应的分析。随后函数通过内核的notifier机制调用已经注册到`PM_SUSPEND_PREPARE`事件的回调函数。我们知道进入sleep的状态的一个重要步骤就是freeze用户态进程，这里通过调用`suspend_freeze_processes`完成这个步骤。

### suspend_devices_and_enter

该函数suspend所有的设备并让系统进入sleep状态。函数首先将系统将要进入的sleep状体存入：

```c
suspend_state_t pm_suspend_target_state;
EXPORT_SYMBOL_GPL(pm_suspend_target_state);
```

随后的调用顺序如下：

* `platform_suspend_begin`，对于`PM_SUSPEND_MEM`来说就是`suspend_ops->begin`
* `suspend_console`，该函数关闭console
* `dpm_suspend_start`
* `suspend_enter`

### suspend_enter

首先一定要注意到这些函数中调用的函数大致分为两类：platform和dpm。platform前缀的函数一般形式为：如果`PM_SUSPEND_IDLE`则不做任何事情，反之则调用`suspend_ops`中对应名称的回调函数。而dpm前缀的函数一般为内核dpm框架的入口，其目的是调用整个系统中所有设备驱动注册的`dev_pm_ops`中对应名称的回调函数。在认识到这一点后，函数的主要调用顺序为：

* `platform_suspend_prepare`
* `dpm_suspend_late`
* `platform_suspend_prepare_late`
* `dpm_suspend_noirq`
* `platform_suspend_prepare_noirq`

在这之后，如果`state == PM_SUSPEND_TO_IDLE`，则调用`s2idle_loop`进入等待状态。反之则调用下面一些列的函数：

```c
	error = suspend_disable_secondary_cpus();
	if (error || suspend_test(TEST_CPUS))
		goto Enable_cpus;

	arch_suspend_disable_irqs();
	BUG_ON(!irqs_disabled());

	system_state = SYSTEM_SUSPEND;

	error = syscore_suspend();
```

关闭所有非boot的CPU，关闭平台中断，然后调用`syscore_suspend`进入sleep状态。而`syscore_suspend`本质上是调用系统中注册的callback：

```c
	list_for_each_entry_reverse(ops, &syscore_ops_list, node)
		if (ops->suspend) {
			if (initcall_debug)
				pr_info("PM: Calling %pS\n", ops->suspend);
			ret = ops->suspend();
			if (ret)
				goto err_out;
			WARN_ONCE(!irqs_disabled(),
				"Interrupts enabled after %pS\n", ops->suspend);
```

可以看到系统中存在`syscore_ops_list`用于保存`syscore_ops`：

```c
struct syscore_ops {
	struct list_head node;
	int (*suspend)(void);
	void (*resume)(void);
	void (*shutdown)(void);
};

static LIST_HEAD(syscore_ops_list);
static DEFINE_MUTEX(syscore_ops_lock);
```

## 基于ACPI的suspend_ops

先从`suspend_ops`的定义入手，可以找到相关的定义和函数：

```c
static const struct platform_suspend_ops *suspend_ops;

struct platform_suspend_ops {
	int (*valid)(suspend_state_t state);
	int (*begin)(suspend_state_t state);
	int (*prepare)(void);
	int (*prepare_late)(void);
	int (*enter)(suspend_state_t state);
	void (*wake)(void);
	void (*finish)(void);
	bool (*suspend_again)(void);
	void (*end)(void);
	void (*recover)(void);
};
```

这里看到`platform_suspend_ops`的注释非常全，就不贴出来了。内核中可以使用`suspend_set_ops`设置该全局变量，这样大部分PMIC和固件的驱动就可以实现整个平台的suspend操作。这里我们主要分析`drivers/acpi/sleep.c`中`acpi_sleep_suspend_setup`中注册的`suspend_ops`。