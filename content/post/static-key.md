+++
title = "Linux内核在RISC-V平台下的static key机制"
date = 2021-06-09

tags = ["kernel"]
+++

这两天在邮件列表中看到了使用static key对内核hot path进行优化的补丁，突然有了兴趣看看内核的底层实现。原理不多说，很容易懂，细节是魔鬼。

# static key

static key在内核中指一套特定的API，或者说特定的技术。本质上是一种刻意的在汇编层面的优化，目的是在汇编层面敲除掉特定的分支，达到减小指令数与优化流水线的目的。这个描述是比较抽象的，可以用一个具体的例子说明。

static key本质是内核的一种binary live patching技术，即在运行时动态修改特定的机器指令区域，达到特定目的的行为。举一个比较简单例子：FPU判断。我们知道内核中是不使用浮点指令的，但用户态可能会使用。一般情况下，内核只负责在进程上下文切换时保存FPU相关的寄存器现场。在这个情况下，内核根据hwcap获取的信息确定CPU是否支持FPU相关指令，以此决定是否对浮点寄存器进行保存。这个判断位于`switch_to`中，一个hot path中，如果使用一个全局的boolean变量进行判断，则会有以下缺点：

* 这个变量几乎等同一个常量，只有最初一次赋值，后续不再更改，但我们的代码后需要每次都对其进行检查
* 这个变量位于另一个cacheline中，需要远程访问
* 这里多了一个分支，要考虑分支预测和流水线的问题

static key本质是利用了gcc的`asm goto`特性，即内联汇编支持跳转到外部C代码中的特定label上，以此实现一个特殊的跳转指令。例如，如果不进行跳转的话，可以修改成`jmp 0`（或者nop）这个形式，直接执行下一条指令。反之，可以live patch成`jmp <addr>`的形式，跳转到特定分支。而这个分支具体的形式，或者相关状态的追踪，通过一个句柄完成，称作`static key`。

代码分析应从`include/linux/jump_label.h`入手，实际上`jump_label`和static key与各种probe机制是分不开的。

## struct static_key

`static key`相关API的核心是`struct static_key`结构体，为了保证兼容性，使得不支持jump label的架构支持同样的API，`struct static_key`有两套定义。在支持jump label的架构下，即`CONFIG_JUMP_LABEL`已定义时，`struct static_key`的定义如下：

```c
struct static_key {
        atomic_t enabled;
/*
 * Note:
 *   To make anonymous unions work with old compilers, the static
 *   initialization of them requires brackets. This creates a dependency
 *   on the order of the struct with the initializers. If any fields
 *   are added, STATIC_KEY_INIT_TRUE and STATIC_KEY_INIT_FALSE may need
 *   to be modified.
 *
 * bit 0 => 1 if key is initially true
 *          0 if initially false
 * bit 1 => 1 if points to struct static_key_mod
 *          0 if points to struct jump_entry
 */
        union {
                unsigned long type;
                struct jump_entry *entries;
                struct static_key_mod *next;
        };
};
```

反之，`struct static_key`仅仅为一个原子变量，API通过原子变量实现，没有使用live patch技术。在上面的定义中，`enabled`字段用于记录`static key`被使能的次数，与`preempt_count`的用途类似。而接下来的union本质上是一个指针，指针指向的地址应该为4字节对齐，这就使得LSB两位可以用作其他用途，用作注释中所述的标志位。

`struct static_key`可以有两个初始值，如下：

```c
#define JUMP_TYPE_FALSE         0UL
#define JUMP_TYPE_TRUE          1UL
#define JUMP_TYPE_LINKED        2UL
#define JUMP_TYPE_MASK          3UL

#define STATIC_KEY_INIT_TRUE                                    \
        { .enabled = { 1 },                                     \
          { .entries = (void *)JUMP_TYPE_TRUE } }
#define STATIC_KEY_INIT_FALSE                                   \
        { .enabled = { 0 },                                     \
          { .entries = (void *)JUMP_TYPE_FALSE } }
```

这就是`static key`相关API中定义的初始值。简单来说，就是定义这个`static key`的初始值是true还是false。内核通过一个比较巧妙的方式根据`static key`的初始值在编译时emit相应的汇编指令，这是一个比较值得学习的C语言trick。这个trick本质上是基于类型的条件编译。在定义了一个`struct static_key`后，我们如何静态检测其初始值是true还是false？这是一个很重要的信息，直接决定最后生成的branch的汇编应该是什么样的。很明显这里不能去检测结构体的值，因为那样是runtime检查，而不是compile-time检查。

为了做到compile-time检查初始值，可以利用C语言的类型系统，以及特定的编译器扩展。可以直接进行如下定义：

```c
struct static_key_true {
        struct static_key key;
};

struct static_key_false {
        struct static_key key;
};
```

并定义用于进行初始化的宏：

```c
#define STATIC_KEY_TRUE_INIT  (struct static_key_true) { .key = STATIC_KEY_INIT_TRUE,  }
#define STATIC_KEY_FALSE_INIT (struct static_key_false){ .key = STATIC_KEY_INIT_FALSE, }

#define DEFINE_STATIC_KEY_TRUE(name)    \
        struct static_key_true name = STATIC_KEY_TRUE_INIT

#define DEFINE_STATIC_KEY_TRUE_RO(name) \
        struct static_key_true name __ro_after_init = STATIC_KEY_TRUE_INIT
```

很明显，通过不同的宏定义出来的结构体虽然内存布局是一样的，但是类型是不一样的。因此可以通过编译器的扩展在编译阶段对其类型进行判断，方法如下：

```c
#define static_branch_likely(x)                                                 \
({                                                                              \
        bool branch;                                                            \
        if (__builtin_types_compatible_p(typeof(*x), struct static_key_true))   \
                branch = !arch_static_branch(&(x)->key, true);                  \
        else if (__builtin_types_compatible_p(typeof(*x), struct static_key_false)) \
                branch = !arch_static_branch_jump(&(x)->key, true);             \
        else                                                                    \
                branch = ____wrong_branch_error();                              \
        likely_notrace(branch);                                                         \
})
```

这里多余的代码会被编译器优化掉，本质上就是基于类型的条件编译。

## 分支实现

想象一下最终实现的分支应该是什么样子的。首先明白需要进行patch的是什么东西，在最理想的状态下，分支可以被这一套机制优化成如下形式：

```
		nop   -- the point of live patching
		aaa
		bbb   -- inline execution code 
		ccc
		...
	1:  branch target
```

在该情况下，分支几乎所有的时间内都会走nop开头的in-line分支代码。也就是说，在该情况下，这里插入分支的代价仅仅是多执行一个nop指令。这个nop指令的位置就是live patching的位置，如果我们想让这个branch走另一条branch路线（也就是if语句的另一个分叉），我们需要将nop修改为`jmp 1`语句，此时branch会跳转到label 1继续执行。

我们从原理上推导一下这个分支是什么样子的，应该提供什么样的API：

* jump label的使用方式应该与普通boolean变量一致，在代码层面是可以替换的。比如`if (condition)`可以无缝替换成`if (is_jump_label_enabled(&label))`这种形式。

* 上面想到的`is_jump_label_enabled`函数返回一个boolean值，且boolean值根据jump label的状态进行改变。这些jump label应该具有类似如下的实现：`nop; return true;`或者`jmp <label>;  .... label: return false; `。

* 上面提到的`jump label`可能有多处地方用到，我们需要记录下来这些jump label的位置信息。

* 最后需要提供一个动态patch**所有分支**的机制。


事实上，上面贴出的`static_branch_likely`宏即为我们假想的`is_jump_label_enabled`函数。内核使用了比较巧妙的方法实现了上述要求，并精准抽象出了平台相关与平台无关的实现。我们以RISC-V为平台相关的实现为基准分析`jump label`的实现。

Linux要求架构相关代码实现两个函数：`arch_static_branch`与`arch_static_branch_jump`。这两个函数的框架如下：

```
		nop （live patch point, can be patched to `jmp 1`）
		return false;
	1:   return true;
```

简单来说，这两个函数插入了一个如上分支结构，`arch_static_branch`直接将patch点填写为nop，而`arch_static_branch_jump`则将nop替换为了`jmp 1`，并返回true。他们俩的返回值表示整个分支结构执行时是否发生了跳转，返回false即表示线性执行，没有跳转，反之亦然。接下来分析具体代码实现，以`arch_static_branch`为例，该函数在RISC-V平台下的实现如下：

```c
static __always_inline bool arch_static_branch(struct static_key *key,
                                               bool branch)
{
        asm_volatile_goto(
                "       .option push                            \n\t"
                "       .option norelax                         \n\t"
                "       .option norvc                           \n\t"
                "1:     nop                                     \n\t"
                "       .option pop                             \n\t"
                "       .pushsection    __jump_table, \"aw\"    \n\t"
                "       .align          " RISCV_LGPTR "         \n\t"
                "       .long           1b - ., %l[label] - .   \n\t"
                "       " RISCV_PTR "   %0 - .                  \n\t"
                "       .popsection                             \n\t"
                :  :  "i"(&((char *)key)[branch]) :  : label);

        return false;
label:
        return true;
}
```

首先解释一下内联汇编的上半部分：

```c
                "       .option push                            \n\t"
                "       .option norelax                         \n\t"
                "       .option norvc                           \n\t"
                "1:     nop                                     \n\t"
                "       .option pop                             \n\t"
```

`.option push`与`.option pop`是通用的，不赘述。`norelax`和`norvc`两个是RISC-V特定的option，其语义分别为：

* 禁止汇编器和链接器对其进行relax操作，即在汇编和链接时更改指令顺序，进行优化
* 禁止汇编器和链接器生成compression扩展指令，亦即RV64GC中`C`表示的压缩指令。很明显差个压缩指令后nop的长度就不一样了。

这两个option可以保证nop指令正确的被插入到对应位置上。同时我们看到，nop指令所位于的地址被标记了一个label 1。随后这段内联汇编进行了另一件事情：

```c
                "       .pushsection    __jump_table, \"aw\"    \n\t"
                "       .align          " RISCV_LGPTR "         \n\t"
                "       .long           1b - ., %l[label] - .   \n\t"
                "       " RISCV_PTR "   %0 - .                  \n\t"
                "       .popsection                             \n\t"
```

`.pushsection`告诉汇编器接下来生成的东西需要放到名为`__jump_table`的section中，"aw"指allocatable与writable。而`.align RISCV_LGPTR`要求接下来的数据是指针对其的，RISCV_LGPTR在rv32上是2（4字节），在64位上是3（8字节）。随后通过`.long`插入了两个32位整数，分别为：

* 前面label 1的地址减去当前地址，即这个整数位于的地址
* 后面C label位于的地址减去当前地址，即这个整数位于的地址

最后，定义了一个指针，指针的值为函数参数`struct static_key *key`的值加上`branch`参数的值（true为1，false为0）后，减去当前地址的值。有关名为`__jump_label`的section分析在接下来进行，现在我们有了足够的理解来分析`static key`的API，即`static_key_likely`宏。

对于`static_key_likely`宏，我们需要根据初始条件来决定生成一个什么样的branch结构。对于定义为`struct static_key_true`的`static key`，我们知道其初始是true的，且它很有可能一直为true。因此，应该直接使用`arch_static_branch`生成填充为nop的branch，然后直接返回`!arch_static_branch()`，因为`static_key_likely(true)`的返回值应该为true，而`arch_static_branch`在不跳转的情况下返回值为false。

反之，对于定义为`struct static_key_false`的`static key`，其初始值为false，且很有可能以后一直是true的。因此我们应该调用`arch_static_branch_jump`生成一个已经填充为jmp指令的branch结构。后续key状态改变，且发生live patch后，这个branch又回到了nop状态，与true对应，此时执行最为高效。

用同样方法可以推导出`static_key_unlikely`的实现，关键以下几点：

* `static_key_{likely,unlikely}`的返回值是什么（默认情况下，即live patch前的）
* `arch_static_branch{,_jump}`的返回值是什么（默认情况下，即live patch前的）

前面看到`arch_static_branch`与`arch_static_branch_jump`往`__jump_label`段中填充了数据。实际上，这三个数据在内核中通过`struct jump_entry`表示：

```c
struct jump_entry {
        s32 code;
        s32 target;
        long key;       // key may be far away from the core kernel under KASLR
};
```

对于生成PIC代码的架构，可以定义`CONFIG_HAVE_ARCH_JUMP_LABEL_RELATIVE`，此时`struct jump_entry`就是上面的定义。否则，架构应该自行定义`struct jump_entry`，并使用绝对地址来保存上面几个值。这几个值实际上可以通过取自身的地址，并加上这个值本身（即偏移量）来还原，举例如下：

```c
static inline unsigned long jump_entry_code(const struct jump_entry *entry)
{
        return (unsigned long)&entry->code + entry->code;
}

static inline struct static_key *jump_entry_key(const struct jump_entry *entry)
{
        long offset = entry->key & ~3L;

        return (struct static_key *)((unsigned long)&entry->key + offset);
}
```

很明显，`struct jump_entry`是对一个已经插入内核代码段的分支的描述，记录了一些基本信息。

## 核心状态逻辑

理解这一套机制的核心就是理解它的状态逻辑，毕竟原理相当简单，很容易就陷入了自以为明白的情况中。通过阅读代码，我总结了如下重点，可以快速理解核心状态逻辑：

* 一定要明确`struct static_key`和`struct jump_entry`的关系。`struct static_key`是对我们想要的布尔变量的抽象，即取代原先`if`语句中的条件变量。`struct jump_entry`则是对生成分支的抽象，记录我们生成的特定分支的信息，这个信息放置于一个名为`__jump_label`的section中。注意真正生成的分支已经插到各种text段中了。
* `struct jump_entry`，记录了一个分支的信息，便于我们后续追踪。包括分支处于的位置，分支是unlikely还是likely的，分支对应于哪一个`struct static_key`，后面还会看到`struct jump_entry`还记录了分支是否位于`__init`段中。
* `struct static_key`记录了这个抽象布尔变量的值（counter形式，类似于preempt_enable等API，可以以类似栈的形式enable或者disable这个key）。还实现对所有与它关联的`struct jump_entry`的反向引用，以及这个`struct static_key`的初始状态，即初始为true还是初始为false。
* 而真正位于`text`段上分支的状态只有一种，我们称其为分支填充状态，只有nop和jump两种情况。nop状态下，live patch点上写的是nop，分支会直接执行in-line的代码，反之，live patch点上填充的是一个jump语句，使得CPU在执行到这里时跳转到out-of-line的代码上。
* 分支的填充状态由`struct static_key`和`struct jump_entry`共同encode。简单来说，当前分支的填充状态可以通过如下表达式进行计算：`（likely/unlikely）^ (true/false)`。举例来说，对于一个likely的jump entry（即它有大概率为true），那么当这个key为true时，它一定是nop状态的。这是由于这套系统就是这么设计的，上述表达式实质上就是我们想要的结果，即一个likely true的分支，在key为true时，一定应该设置成最高效的nop状态。`static_key_likely`和`static_key_unlikely`实际上就是根据这个表达式设计的，根据这个表达式生成分支，以获取最优的性能。这个表达式本质上就是jump label设计的核心逻辑。让likely和unlikely的分支在命中时处于最高效的形式中。

状态解码相关的实现举例如下：

```c
static enum jump_label_type jump_label_type(struct jump_entry *entry)
{
        struct static_key *key = jump_entry_key(entry);
        bool enabled = static_key_enabled(key);
        bool branch = jump_entry_is_branch(entry);

        /* See the comment in linux/jump_label.h */
        return enabled ^ branch;
}
```

## __jump_label段

我们注意到有如下定义：

```c
extern struct jump_entry __start___jump_table[];
extern struct jump_entry __stop___jump_table[];
```

同时从链接脚本中可以看到如下定义：

```c
#define JUMP_TABLE_DATA                                                 \
        . = ALIGN(8);                                                   \
        __start___jump_table = .;                                       \
        KEEP(*(__jump_table))                                           \
        __stop___jump_table = .;
```

也就是说，`__start___jump_table`是`__jump_table`段的开头。可以直接在内核中找到这个段是在哪里被处理的，即`jump_label_init`函数。

`jump_label_init`被main函数调用，是在内核初始化过程中调用的比较早的。大概位置刚好就是内核打出`Command Line:` 那行日志后。函数首先讲整个section中的jump_entry根据key的值进行排序，排序过程是一个in-place的swap过程。经过排序后，`__jump_label`段中的元素就是`struct static_key`连续的了，即对应一个`struct static_key`的`struct jump_label`在段中是连续存在的了。随后，函数遍历`__jump_label`段中的所有元素，并进行如下操作：

* 对所有初始化为nop形式的branch调用`arch_jump_label_transform_static`。RISC-V上该函数为空，按照我的理解，其他平台上如x86，可能采取先将nop填写为其他数据，随后通过初始化修改nop的行为。可能只特定于x86这种变长指令集，至少arm64下这个函数也是空的。
* 初始化`struct static_key`的`entries`字段，将其指向连续`struct jump_entry`的第一个元素。
* 对于所有位于`__init`段的branch，调用`jump_entry_set_init`函数。此举改变了`struct jump_entry`的标记，使得后续可以辨别一个`struct jump_entry`是否位于`__init`段中。

## 状态翻转

状态翻转在更改`struct static_key`的状态时进行，简单来说，当我们想把一个key的状态从false设置为true的时候，需要将其对应的所有的`struct jump_entry`做live patch操作，翻转其的状态。我们以一个简单的API入口举例：

```c
void static_key_enable_cpuslocked(struct static_key *key)
{
        STATIC_KEY_CHECK_USE(key);
        lockdep_assert_cpus_held();

        if (atomic_read(&key->enabled) > 0) {
                WARN_ON_ONCE(atomic_read(&key->enabled) != 1);
                return;
        }

        jump_label_lock();
        if (atomic_read(&key->enabled) == 0) {
                atomic_set(&key->enabled, -1);
                jump_label_update(key);
                /*
                 * See static_key_slow_inc().
                 */
                atomic_set_release(&key->enabled, 1);
        }
        jump_label_unlock();
}
EXPORT_SYMBOL_GPL(static_key_enable_cpuslocked);

void static_key_enable(struct static_key *key)
{
        cpus_read_lock();
        static_key_enable_cpuslocked(key);
        cpus_read_unlock();
}
EXPORT_SYMBOL_GPL(static_key_enable);
```

本质上，函数根据`jump_label_can_update`函数确定一个`struct jump_entry`是否可以live patch。一般情况下只有`__init`段中的代码无法live patch。确认完成后，使用架构相关的`arch_jump_label_transform`将一个`struct jump_entry`进行live patch操作。对于RISC-V平台，由于指令是定长的，仅仅是根据`jump_entry_target`生成jump指令，或者简单使用nop对特定位置的u32进行更新而已。注意，这个更新需要刷新icache，invalidate特定的地址区域，否则无法立即生效。

