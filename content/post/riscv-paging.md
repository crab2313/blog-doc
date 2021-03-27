+++
title = "Linux内核在RISC-V下的页表实现"
date = 2020-07-23
draft = true


tags = ["kernel", "risc-v", "memory"]

+++

本文分析Linux与RISC-V体系结构相关的页表实现。目的是通过仔细分析体系结构相关的代码，进一步加深对内核页表相关API实现的理解。

读代码与写代码一样，需要寻找入手点，对于页表实现来说可以直接想到的有两个：最抽象的和最不抽象的，我们从最不抽象的入手。最不抽象的部分很明显就是每级页表表项数据的定义，可以在`arch/riscv/include/asm/pgtable-bits.h`中找到相关定义。

```c
/*
 * PTE format:
 * | XLEN-1  10 | 9             8 | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0
 *       PFN      reserved for SW   D   A   G   U   X   W   R   V
 */

#define _PAGE_ACCESSED_OFFSET 6

#define _PAGE_PRESENT   (1 << 0)
#define _PAGE_READ      (1 << 1)    /* Readable */
#define _PAGE_WRITE     (1 << 2)    /* Writable */
#define _PAGE_EXEC      (1 << 3)    /* Executable */
#define _PAGE_USER      (1 << 4)    /* User */
#define _PAGE_GLOBAL    (1 << 5)    /* Global */
#define _PAGE_ACCESSED  (1 << 6)    /* Set by hardware on any access */
#define _PAGE_DIRTY     (1 << 7)    /* Set by hardware on any write */
#define _PAGE_SOFT      (1 << 8)    /* Reserved for software */
```

很直观，主要是抽象一下页表表项中每一位的定义，顺带包含一些mask与偏移量。接下来我们看`PTE`的定义，位于`arch/riscv/include/asm/page.h`：

```c
typedef struct {
	unsigned long pte;
} pte_t;
```

很多书中都有提到，但是这里再提一下：内核中默认`unsigned long`与`void *`是相同的，即当前架构指针的指针类型。当使用`unsigned long`时，我们更加倾向于将这个地址看作数据，用于处理。这里用`typedef`定义的`pte_t`本质上还是一个`unsigned long`类型的数据，只不过我们利用C语言的类型系统禁止用户将其当作`unsigned long`进行使用。