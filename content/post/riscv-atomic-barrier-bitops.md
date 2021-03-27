+++
title = "Linux内核在RISC-V架构下的内存屏障与原子操作"
date = 2020-08-14


tags = ["kernel", "risc-v", "isa"]

+++

内存一致性模型是一个体系结构中至关重要的一部分，本质上为软件与硬件之间的契约。软件开发人员可以从内存一致性模型中得知硬件进行内存操作时可能的行为，这是多线程共享内存操作正确性的基石。RISC-V的内存模型被称作RVWMO（RISC-V Weak Memory Order），本质上是`Release Consistency`与`Relaxed Consistency`的结合体。本文试图从Linux对内存一致性模型的抽象API入手，分析这些抽象API的用法用例以及对应RISC-V体系结构的实现。

题外话，来总结一下想要理解这部分内容需要哪些知识储备：

* GCC內联汇编基础。上述提到的抽象API有很大一部分都是使用具体体系结构下的汇编代码实现，因此掌握GCC提供的扩展內联汇编是读懂对应实现的必要基础。建议看这本[简易教程](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-3.html)和[GCC官方文档](https://gcc.gnu.org/onlinedocs/gcc-10.2.0/gcc/Extended-Asm.html)。
* 计算机体系结构相关理论。这里没什么多说的，目前大部分体系结构手册都假定读者具有扎实的计算机体系结构理论基础。这里推荐[《A Primer on Memory Consistency and Cache Coherence, 2nd》](https://www.morganclaypool.com/doi/10.2200/S00962ED2V01Y201910CAC049)，里面有引用一些paper，建议一读。
* 对应体系结构手册。说到底，内存一致性是体系结构不可分离的一部分，因此熟读体系结构手册是理解对应操作的有力途径。
* 阅读内核文档。内核的`memory-barrier.txt`、`atomic_t.txt`以及`atomic_bitops.txt`都是极其重要的说明文档。

## 内存屏障

RISC-V的内存屏障全部由`fence`指令实现，第一次读RISC-V手册的人一般会难以相信`fence`的设计竟然这么简单。`fence`指令形式如下：

```asm
fence [iorw], [iorw]
```

其中`[iorw]`为`iorw`中的任选几个字母，其中：

* i：设备内存输入（读取）
* o: 设备内存输出（写入）
* r：普通内存读取
* w：普通内存写入

整个指令`fence [set1], [set2]`的语义为：

* 定义predecessor集合为该`fence`指令之前所有属于`[set1]`类型的指令之集合
* 定义successor集合为该`fence`指令之后所有属于`[set2]`类型的指令之集合
* 则其他RISC-V Hart或者外部设备不会观测到successor集合中的指令在predecessor集合中的指令之前发生

Linux内核中，内存屏障相关的定义都在体系结构对应文件夹下的`include/asm/barrier.h`中。其中，对于`fence`指令的抽象如下：

```c
#define RISCV_FENCE(p, s) \
	__asm__ __volatile__ ("fence " #p "," #s : : : "memory")
```

很显然是一条简单的內联汇编，`__volatile__`与`memory`都是用于阻止编译器进行优化的常规操作。后续的实现都是围绕这个宏进行的。

内核中对这些宏的实现策略比较简单，基本原理如下：

* 在`asm-generic`中的`barrier.h`中定义默认实现，如果目标架构没有对应实现，则启动默认实现
* 在`arch/include/asm/barrier.h`中定义架构特定实现，一般为内联汇编定义

### mb && rmb && wmb

这三个操作与其对应的`smp_*`之间最大的区别就是需要对设备内存也生效，所以它们的实现如下：

```c
#define mb()		RISCV_FENCE(iorw,iorw)
#define rmb()		RISCV_FENCE(ir,ir)
#define wmb()		RISCV_FENCE(ow,ow)
```

### smp_mb && smp_rmb && smp_wmb

首先一定明确`smp_`前缀所代表的含义，即“用于SMP（symmetric multi-processor）的”。因此，在内核支持SMP时，它们被定义为其对应的`__smp_*`，反之则定义为`barrier()`，即普通的编译器内存屏障，防止编译器进行内存访问重排。前面也提到了，这几个操作并不需要考虑到设备内存，因此它们定义如下：

```c
#define __smp_mb()	RISCV_FENCE(rw,rw)
#define __smp_rmb()	RISCV_FENCE(r,r)
#define __smp_wmb()	RISCV_FENCE(w,w)
```

### smp_load_acquire && smp_store_release

```c
#define __smp_store_release(p, v)					\
do {									\
	compiletime_assert_atomic_type(*p);				\
	RISCV_FENCE(rw,w);						\
	WRITE_ONCE(*p, v);						\
} while (0)

#define __smp_load_acquire(p)						\
({									\
	typeof(*p) ___p1 = READ_ONCE(*p);				\
	compiletime_assert_atomic_type(*p);				\
	RISCV_FENCE(r,rw);						\
	___p1;								\
})
```

还是跟前面提到的一样，带`smp_`前缀的宏都是只在`CONFIG_SMP`开启时有定义，否则为空操作。`load acquire`的定义就是在这个`barrier`之后的读写不能出现在它之前，且`load acquire`是读取操作，所以自然使用`RISCV_FENCE(r, rw)`。`store release`同理。

## 原子操作

可以从`arch/riscv/Kconfig`中看到RISC-V平台在非64位时会选择`CONFIG_GENERIC_ATOMIC64`，如下：

```
config RISCV
        def_bool y
        select ARCH_CLOCKSOURCE_INIT
        select ARCH_SUPPORTS_ATOMIC_RMW
        ......
        select GENERIC_ATOMIC64 if !64BIT   <== 
        select GENERIC_CLOCKEVENTS
        ......
```

这点很好理解：在32位下RISC-V架构无法保证64位操作的原子性，因此内核使用通用的64位原子操作实现，通过自旋锁实现64位原子操作，这在`arch/riscv/include/asm/atomic.h`开头中有体现：

```c
#ifdef CONFIG_GENERIC_ATOMIC64
# include <asm-generic/atomic64.h>
#else
# if (__riscv_xlen < 64)
#  error "64-bit atomics require XLEN to be at least 64"
# endif
#endi
```

原子变量的定义是跨平台的，位于`include/linux/types.h`中：

```c
typedef struct {
	int counter;
} atomic_t;

#ifdef CONFIG_64BIT
typedef struct {
	s64 counter;
} atomic64_t;
#endif
```

这个原子性由硬件保证，一般来说，一个架构的word大小数据在对齐访问的情况下是可以保证原子性的，具体需要翻看手册。随后是`fence`指令实现的`release acquire`语义：

```c
#define __atomic_acquire_fence()					\
	__asm__ __volatile__(RISCV_ACQUIRE_BARRIER "" ::: "memory")

#define __atomic_release_fence()					\
	__asm__ __volatile__(RISCV_RELEASE_BARRIER "" ::: "memory");
```

这个用于`atomic-fallback.h`中自动生成的函数。对于读写这种`non-RMW`操作，如你所见就是这么简单：

```c
static __always_inline int atomic_read(const atomic_t *v)
{
	return READ_ONCE(v->counter);
}
static __always_inline void atomic_set(atomic_t *v, int i)
{
	WRITE_ONCE(v->counter, i);
}
```

还是那句话，硬件保证原子性，内核只要保证生成的指令不走样就行了，这也是使用`READ_ONCE`和`WRITE_ONCE`的原因。接下来就是重头戏：对于`RMW`操作实现。RISC-V中定义了原子操作指令，即被称为`A`的扩展，Linux内核默认要求其已被实现。内核中通过內联汇编模板的方式实现这些操作，也算是比较简洁的了。

对于`RMW`类的原子操作，我们主要关注其：

* 功能正确性
* 内存序正确性

### 指令简介

先来简单介绍一下RISC-V的原子操作指令。很简单，几句话就可以描述清楚：

* RISC-V的原子操作指令命名类如`amo{op}.{w/d}.{rl/aq/aqrl}`。第一部分描述功能，如`amoadd`和`amoswap`等等，其中`amo`是`atomic memory operation`的缩写。第二部分为操作数据的长度，w（word）表示32位，d（double word）表示32位。第三部分比较有意思，RISC-V的原子操作指令中encode了两位，分别`acquire`和`release`，使其具有了内存序属性，看得出来是对OS进行了高度优化的。
* RISC-V的原子操作指令编码了三个寄存器：`rs1`、`rs2`和`rd`。其中`rs1`为原子变量的内存地址，`rs2`是该操作的另一个操作数（operand）。指令执行时，首先从`rs1`指向的内存中取出原子变量的值，保存在`rd`寄存器，然后与`rs2`进行操作，最后将结果保存回`rs1`指向的内存地址。这里注意RISC-V的原子操作可以将变量的地址保存在`rd`寄存器，如果不需要可用`zero`寄存器当作`rd`。

### 无返回值原子操作函数

顾名思义，就是没有返回值的`RMW`类原子操作函数，注意这类函数在内核中是没有内存序要求的。可以看到这类函数使用一个通用的模板生成：

```c
#define ATOMIC_OP(op, asm_op, I, asm_type, c_type, prefix)		\
static __always_inline							\
void atomic##prefix##_##op(c_type i, atomic##prefix##_t *v)		\
{									\
	__asm__ __volatile__ (						\
		"	amo" #asm_op "." #asm_type " zero, %1, %0"	\
		: "+A" (v->counter)					\
		: "r" (I)						\
		: "memory");						\
}			
```

很简单，唯一需要注意的就是这个`+A`。从GCC官方的[文档](https://gcc.gnu.org/onlinedocs/gcc/Machine-Constraints.html#Machine-Constraints)中可以看到，`A`是RISC-V中单独定义的，表示一个存放着内存地址的寄存器变量。且返回值寄存器被设置成`zero`，以示忽略。

```c
#define ATOMIC_OPS(op, asm_op, I)					\
        ATOMIC_OP (op, asm_op, I, w, int,   )				\
        ATOMIC_OP (op, asm_op, I, d, s64, 64)
#endif

ATOMIC_OPS(add, add,  i)
ATOMIC_OPS(sub, add, -i)
ATOMIC_OPS(and, and,  i)
ATOMIC_OPS( or,  or,  i)
ATOMIC_OPS(xor, xor,  i)
```

### 有返回值原子操作函数

内核中，有返回值原子操作函数分为`fetch`和`return`。这二者的区别为`fetch`返回原子变量原有的值，而`return`返回原子变量经过操作的值。我们可以从内核中的`atomic_t.txt`文档中知道，默认情况下，有返回值的原子操作函数都是有内存序的。且函数具有一些加了特殊后缀的变体，可以指定内存序语义，如`_relaxed`、`_acquire`和`_release`。

```c
#define ATOMIC_FETCH_OP(op, asm_op, I, asm_type, c_type, prefix)	\
static __always_inline							\
c_type atomic##prefix##_fetch_##op##_relaxed(c_type i,			\
					     atomic##prefix##_t *v)	\
{									\
	register c_type ret;						\
	__asm__ __volatile__ (						\
		"	amo" #asm_op "." #asm_type " %1, %2, %0"	\
		: "+A" (v->counter), "=r" (ret)				\
		: "r" (I)						\
		: "memory");						\
	return ret;							\
}									\
static __always_inline							\
c_type atomic##prefix##_fetch_##op(c_type i, atomic##prefix##_t *v)	\
{									\
	register c_type ret;						\
	__asm__ __volatile__ (						\
		"	amo" #asm_op "." #asm_type ".aqrl  %1, %2, %0"	\
		: "+A" (v->counter), "=r" (ret)				\
		: "r" (I)						\
		: "memory");						\
	return ret;							\
}
```

可以看到`fetch`函数的內联汇编模板也分两套，对于`_relaxed`函数，没有加上`.aqrl`，即不指定内存序语义。`return`函数实际上就是`fetch`函数返回值经过重新计算得出，不再赘述，注意想清楚`atomic`的操作究竟在哪，就不会有`return`函数不是原子操作的错觉。

### atomic_fetch_add_unless && atomic_sub_if_positive

RISC-V结构下对这两个函数做了实现，且都是利用了`LR/SC`指令。先来看一下它们实现的功能：

* `atomic_fetch_add_unless`有两个额外参数`a`和`u`，进行操作时，如果原子变量的值与`u`不相等，则将其加上`a`，并返回原先的值。
* `atomic_sub_if_positive`从名字上就可以看出来功能：如果原子变量的值是正的，则将其减去参数传入的值，并返回最后的结果。

前面提到这两个操作是`LR/SC`指令实现的，那么先简介一下这对指令是如何工作的：

* LR指令是`load reserved`的缩写，它首先会读取一个内存地址的值，然后在该内存地址做标记。
* SC指令是`store conditional`的缩写，它的作用写将一个值写入一个内存地址。对于同一个`HART`，`SC`首先检查标记的值是否正确，如果正确才进行写入操作，否则返回错误。注意无论如何对应地址的标记都会被`SC`指令清除。
* LR和SC配合使用，其意义在于：如果SC指令成功执行，则意味着在LR到SC指令这一段时间内，没有其他的`HART`对这个地址进行访问（因为标记没有失效）。

```c
	__asm__ __volatile__ (
		"0:	lr.w     %[p],  %[c]\n"
		"	beq      %[p],  %[u], 1f\n"
		"	add      %[rc], %[p], %[a]\n"
		"	sc.w.rl  %[rc], %[rc], %[c]\n"
		"	bnez     %[rc], 0b\n"
		"	fence    rw, rw\n"
		"1:\n"
		: [p]"=&r" (prev), [rc]"=&r" (rc), [c]"+A" (v->counter)
		: [a]"r" (a), [u]"r" (u)
		: "memory");
```

可以看到本质是通过循环调用`LR/SC`对，不断尝试，如果成功，则说明这段时间内没有人访问原子变量，操作成功独占，故而肯定是原子的。