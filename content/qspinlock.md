+++
title = "内核中qspinlock的实现分析"
date = 2020-05-28
draft = true

[taxonomies]
tags = ["kernel", "spinlock", "concurrency"]

+++

经典spinlock的实现原理是基于TAS（test-and-set）操作的，这类实现在cache-coherent内存上的性能非常堪忧，会极大的消耗互联中的资源，且存在livelock产生的风险。Linux内核在比较新的版本中尝试引入了qspinlock，试图解决传统spinlock的问题，提供更好的延展性。

## qspinlock的定义

qspinlock定义如下：

```c
typedef struct qspinlock {
        union {
                atomic_t val;

                /*
                 * By using the whole 2nd least significant byte for the
                 * pending bit, we can allow better optimization of the lock
                 * acquisition for the pending bit holder.
                 */
#ifdef __LITTLE_ENDIAN
                struct {
                        u8      locked;
                        u8      pending;
                };
                struct {
                        u16     locked_pending;
                        u16     tail;
                };
#else
                struct {
                        u16     tail;
                        u16     locked_pending;
                };
                struct {
                        u8      reserved[2];
                        u8      pending;
                        u8      locked;
                };
#endif
        };
} arch_spinlock_t;
```

注意qspinlock实质上就是一个32位的整型，定义中的union是用于访问32位整型中特定部分的写法。实际上qspinlock的代码看起来很短，实质上是非常复杂的，必须具有无锁编程等相关知识背景才能彻底理解它的实现，建议先看完`memory-barrier.txt`和`atomic_t.txt`等内核文档。其次，qspinlock其实是MCS锁的改进版，想要明白它的工作原理，首先要明白MCS锁是如何工作的，可以参考这篇[论文](https://bugzilla.kernel.org/show_bug.cgi?id=206115)。

我们可以从`asm-generic/qspinlock.h`中的注释看到`val`是如何定义的：

```c
/*
 * Bitfields in the atomic value:
 *
 * When NR_CPUS < 16K
 *  0- 7: locked byte
 *     8: pending
 *  9-15: not used
 * 16-17: tail index
 * 18-31: tail cpu (+1)
 *
 * When NR_CPUS >= 16K
 *  0- 7: locked byte
 *     8: pending
 *  9-10: tail index
 * 11-31: tail cpu (+1)
 */
```

可以看到，内核编译时支持的最大CPU数会影响`val`的定义，我们只看小于16K的情况。



https://lwn.net/Articles/590243/