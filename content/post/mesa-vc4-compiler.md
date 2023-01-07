+++
title = "MESA源码分析：VC4驱动 - 编译器实现"
date = 2023-01-07


tags = ["mesa", "gallium3d", "vc4"]

+++

这里先笼统的概括一下编译器部分的工作原理，细节后面再探讨。整个vc4的编译器后端一共进行了两次转换，多级优化，并实现了shader cache。从代码中可以发现，对一个shader编译是非常昂贵的操作，挑选特定的关键字（即shader的特征状态），根据关键字确定一个shader的唯一性，从而实现cache shader的操作，能极大的减少shader的编译次数，提升总体性能和帧率稳定性。这里列举一下想要进行分析的后端功能点：

* 多级转换，最终生成shader二进制指令
* shader cache的工作原理
* 多级优化（lowering）的实现原理

对于多级转换，实际上进行了如下几个转换操作：

* NIR转换成QIR。NIR是目前Mesa主流的IR，是Mesa的GLSL编译器生成TGSI中间表示后，一般要转换而成的中间表示。以前的Mesa驱动一般直接处理TGSI，而现在的Mesa驱动一般将TGSI通过共用功能模块转换成NIR后进行处理。NIR是一种比较便于优化的SSA表示方法。而QIR则是vc4根据自身的QPU特性设计出来的IR表示，这以转换阶段则直接将优化过的NIR转换为QIR表示。
* QIR转换成二进制表示。vc4中实现了code emitter，通过读取QIR，而emit出最终的二进制指令。整个emit过程基本是按照相应的模板进行翻译操作。最后通过优化的方式将ALU A和ALU M相关的操作整合到一起，形成最终的二进制程序。



# QIR

QIR是vc4编译器后端的IR表示。NIR经过几轮优化之后，转换成QIR，进一步通过code emitter生成最终的shader指令程序。QIR的指令使用`qinst`表示，可以看到qinst提供了list_head用于串到一个链表上，且提供了一个op，表示操作类型，并存在source operand和dest operand，以及多个flag。

```c
struct qinst {
        struct list_head link;

        enum qop op;
        struct qreg dst;
        struct qreg src[3];
        bool sf;
        bool cond_is_exec_mask;
        uint8_t cond;
};
```

operand的类型为`struct qreg`，本质上是寄存器，至于为什么这么抽象，是因为QPU本质上不支持MMIO，而是只支持register mapped I/O。也就是I/O实际上是通过读取累加器或者寄存器实现的。

```c
struct qreg {
        enum qfile file;
        uint32_t index;
        int pack;
};
```

`struct qreg`中使用qfile字段表示寄存器位于的存储区域，举上几个例子：

* 使用TEMP类型标志临时的寄存器类型
* 使用VARY表示从VARYING差值硬件中读取出来的数据
* 使用UNIF表示从UNIFORM变量中读取出来的类型
* 使用VPM表示存VPM存储区域中读取出来的数据

除此之外，还有很多类型，这里不一一列举。对于临时寄存器，`qir_get_temp`函数负责申请一个这样的临时寄存器。`vc4_compile`中记录的所有的临时寄存器的申请指令，使得我们可以根据临时寄存器快速定位到申请这个临时寄存器的指令。最后一提，这个映射关系存储在`vc4_compile->defs`数组中。在这里可以看出，vc4_compile本质上就是记录QIR编译结果的结构体。

所有的指令生成操作从概念上是像一个`vc4_compile`中的特定block添加一个指令，即emit。emit类型的操作实际上使用类似`qir_<OP-NAME>`这个的函数名称，通过特定的模板生成而来，如`QIR_ALU0`和`QIR_ALU1`等，具体可以参考源码。这里只以其中一个举例其原理，`QIR_ALU1`:

```c
#define QIR_ALU1(name)                                                   \
static inline struct qreg                                                \
qir_##name(struct vc4_compile *c, struct qreg a)                         \
{                                                                        \
        return qir_emit_def(c, qir_inst(QOP_##name, c->undef,            \
                                        a, c->undef));                   \
}                                                                        \
static inline struct qinst *                                             \
qir_##name##_dest(struct vc4_compile *c, struct qreg dest,               \
                  struct qreg a)                                         \
{                                                                        \
        return qir_emit_nondef(c, qir_inst(QOP_##name, dest, a,          \
                                           c->undef));                   \
}
```

先明白`qir_emit_def`和`qir_emit_nodef`的区别：上面看到对于新的TEMP寄存器，需要定义他的指令的地址，这个记录通过`vc4_compile->defs`数组进行记录。所以`qir_emit_def`就很好理解，这里的`_def`后缀指添加的这条指令定义了一个新的TEMP寄存器变量。具有dest的指令不需要定义一个新的TEMP寄存器，因为他已经有操作的destination了，而不需要重新定义一个。这里的`QIR_ALU1`是指有一个source operand的ALU指令。

```c
static void
qir_emit(struct vc4_compile *c, struct qinst *inst)
{
        list_addtail(&inst->link, &c->cur_block->instructions);
}

/* Updates inst to write to a new temporary, emits it, and notes the def. */
struct qreg
qir_emit_def(struct vc4_compile *c, struct qinst *inst)
{
        assert(inst->dst.file == QFILE_NULL);

        inst->dst = qir_get_temp(c);

        if (inst->dst.file == QFILE_TEMP)
                c->defs[inst->dst.index] = inst;

        qir_emit(c, inst);

        return inst->dst;
}

struct qreg
qir_get_temp(struct vc4_compile *c)
{
        struct qreg reg;

        reg.file = QFILE_TEMP;
        reg.index = c->num_temps++;
        reg.pack = 0;

    	// .......

        return reg;
}
```

从commit信息中可以看到QIR是一个基于SSA的通用IR。上面也看到SSA的特有结构，即TEMP寄存器，后面会分析PHI节点是如何实现的。但是SSA的另一个特征是通过block（或者准确来说basic block）组织的，下面来分析QIR中这一部分怎么实现。

QIR中，使用`struct qblock`来表示一个block，这个结构体的内容比较零散，目前我们只需要知道：

* 每一个block创建的时候会有一个唯一的整数index
* block中存在instructions链表用于保存block中的指令
* QIR中使用`qir_new_block`创建并添加一个新的block
* `vc4_compile`存在blocks字段记录这个编译结果中所有的blocks，并使用`cur_block`指针指向当前添加指令时的目标block

# NIR转换为QIR

## nir_to_qir

NIR在经过各种优化之后，在`vc4_shader_ntq`函数中调用`nir_to_qir`转换成QIR。

```c
static void
nir_to_qir(struct vc4_compile *c)
{
        if (c->stage == QSTAGE_FRAG && c->s->info.fs.uses_discard)
                c->discard = qir_MOV(c, qir_uniform_ui(c, 0));

        ntq_setup_inputs(c);
        ntq_setup_outputs(c);

        /* Find the main function and emit the body. */
        nir_foreach_function(function, c->s) {
                assert(strcmp(function->name, "main") == 0);
                assert(function->impl);
                ntq_emit_impl(c, function->impl);
        }
}
```

其中ntq应该就是`Nir To Qir`的缩写。`ntq_setup_inputs`的功能比较直观，配置好QIR程序的输入。我们知道shader程序应该有多个输入变量的，这里要将他们从NIR映射到QIR中。

* ntq_set_inputs本质上是生成指令，将输入shader参数读取到一部分TEMP变量中
* 

# 寄存器分配器

这里先谈谈我对寄存器分配器的理解，然后结合vc4的实现进行分析。一般情况下IR会假定寄存器个数是无限多个，但是实际上可用的物理寄存器的个数是有限的。所以code generation中比较重要的一环就是将IR中使用的临时（虚拟）寄存器进行转换，将物理寄存器分配到相应的虚拟寄存器上。实现这个过程的代码单元被称作寄存器分配器。

经过多年的改进，寄存器分配的实现比较固定了，是通过一个图论问题进行抽象的。以所有临时寄存器为顶点，对于任意两个顶点（即临时寄存器），如果他们的生命周期重叠（即在同一时刻，他们俩都活着需要被下面的语句使用），那么将这两个顶点连起来。在这个图被构造出来之后，寄存器分配的问题即转换成了一个着色问题：假设物理寄存器的个数为k，那么找出一种着色方案，使得相邻的两个顶点没有相同的颜色。

这个问题是一个NR-hard的问题，没有有效的算法，但是实际上大家都使用一个比较经典的经验算法。外加，这个问题在经验算法失效的时候，存在另一种解决方案，即将虚拟寄存器的值缓存到内存中，从而进一步简化问题。所以实践中，这个问题是比较好解决的。

在GPU场景下，上面的抽象忽略了一个重要的问题，即GPU的寄存器往往不是统一个架构，也就是说很多情况下，GPU的寄存器不能仅仅看成相等的一类寄存器，即每个寄存器都是相同的可以互相替换，存在各种形式的专用寄存器。针对这种情况，有人提出了class的概念，并扩展了原先的图论问题。可以参考这篇[论文](https://user.it.uu.se/~svenolof/wpo/AllocSCOPES2003.20030626b.pdf)，事实上Mesa的寄存器分配器也是根据这篇论文实现的。

这个方法实质上清晰的定义了寄存器的“目标模型”：<Regs, Conflict, Classes>

* Regs是寄存器集合。寄存器集合中的寄存器名，实质上不是物理寄存器的集合，而是一个概念上的寄存器集合。只要指令集中有对该寄存器名称的操作，那么这个寄存器名称就可以添加到这个寄存器集合中。毕竟很明显，会出现寄存器重叠的情况，比如x86下就有单独操64位寄存器的低32位寄存器的寄存器别名。
* Conflict是一个relation（离散数学里的概念），是对称和自反性的，作用与寄存器集合。这个relation实际上描述一组不能同时分配出去的寄存器，和上面一样，典型的场景就是寄存器重叠，即两个寄存器名字对应的物理寄存器在一个区域上，还是上面x86的那个例子。
* Classes是一组寄存器类，每个寄存器类是一个寄存器集合的子集。寄存器类这个概念比较难理解，简单来讲，有些指令集operation限制其operand于一个特定的寄存器集合子集中，说成白话，就是有些指令会要求指令结果必须是只能保存在某些特定的寄存器，即对operand的寄存器选择有限制。

从工程概念的角度上来看，明白上面这些实际上已经能够理解Mesa中对寄存器分配器的使用了。但是我还想具体写下这个算法的实现原理。简单来说，算法首先构建这个图，然后在图中找一个顶点，这个顶点满足特定的属性要求，即local colorable。这个属性实际上意味着，对于任意这个顶点的相邻顶点的着色分配，我们可以找出一个新的颜色，使这个顶点与其他相邻顶点没有颜色冲突。

一旦找到这个属性的节点，那么就可以将这个顶点从图中去除掉，然后获得一个新图。重复这个过程，直到最后得到一个没有顶点的空图。然后反向将原来去除的节点依次添加回图（最后去除的最先添加），并给新添加的这个顶点赋予一个与其相邻节点都不相同的颜色。重复这个过程，即得到一个完成着色的图，按照图的定义，我们即得到了一个寄存器分配。这个过程中实际上存在情况，找不到local colorable属性的顶点，碰到这样的情况，我们就根据经验方法从图中去掉一个顶点，把这个顶点对应的temp寄存器放到内存空间中暂存，这个操作称作spill。去除掉一个顶点后得到的新图，重头开始这样的算法。最后一个问题就是怎么判断local colorable属性，论文中提出了pq test，可以参考论文的描述与证明。

## 寄存器集合构建

回到Mesa中，Mesa实现了上述基于class的寄存器分配器，并作为公共代码被大部分驱动所用。从原理上，我们可以理解，想要使用这样的寄存器分配器，我们实际上需要：

* 定义寄存器集合
* 定义寄存器集合中的冲突
* 定义寄存器class
* 计算temp变量的生命周期，并获取冲突情况

`vc4_register_allocate.c`文件中实现vc4的寄存器分配。其中，`vc4_alloc_reg_set`函数中进行了寄存器的定义。函数中可以看到，`ra_alloc_reg_set`用于申请寄存器集合，得到一个`struct ra_regs`对象。`ra_alloc_contig_reg_class`函数用于申请class，得到`struct ra_class`对象。从函数中可以看到，定义了如下的寄存器：

* QPU中的累加器R0-R4
* 寄存器bank A中的0-31号寄存器
* 寄存器bank B中的0-31号寄存器

并定义了如下的class：

* reg_class_any[i] R4 A0-A13 A15 (A16-A31) B0-B13 B15 (B16-B31)
* reg_class_a_or_b[i] - A0-A13 A15 (A16-A31) B0-B13 B15 (B16-B31)
* reg_class_a_or_b_or_acc[i] - R0-R3 A0-A13 A15 (A16-A31) B0-B13 B15 (B16-B31)
* reg_class_r4_or_a[i] - R4 A0-A13 A15 (A16-A31)
* reg_class_a[i] - A0-A13 A15 (A16-A31)
* reg_class_r0_r3 - R0-R3

其中的[i]表示这个class分为0和1两个，分别对应整个bank的空间与bank的下半部空间，括号里的是0号class中才有的。这是因为threaded fragment shader场景下会将整个bank分成上下部分，分别给运行在QPU两个线程上的fragment shader使用。

函数最后调用`ra_set_finalize`函数，进行了pq test的预计算工作。

## 冲突图构建

得到寄存器集合，以及class的定义之后，构建图：

```c
        struct ra_graph *g = ra_alloc_interference_graph(vc4->regs,
                                                         c->num_temps);
```

得到`struct ra_graph`结构体。从上面对整个算法的原理上的理解，构建这个图要做的事情如下：

* 得到冲突关系。事实上vc4的寄存器没有定义寄存器冲突关系，也就是寄存器没有重叠关系。那么仅剩的冲突关系就是由于temp寄存器生命周期冲突导致的寄存器分配冲突关系。要得到这个冲突关系，我们首先要计算temp寄存器的生命周期，然后通过比对得到冲突关系。
* 为每一个顶点分配寄存器类。每个顶点都有其对应的寄存器类，表明这个顶点所表示的temp寄存器可以选择的寄存器类型。后面可以看到，vc4驱动中通过枚举qir指令，根据指令的类型为temp寄存器指定其对应的寄存器类。

`qir_calculate_live_intervals`函数用于计算临时变量的生命周期。一个temp变量的生命周期由两个变量表示，分别是生命周期的开始和结束：temp_start和temp_end函数。

可以看到进行一个简单的优化：

```c
        for (uint32_t i = 0; i < c->num_temps; i++) {
                map[i].temp = i;
                map[i].priority = c->temp_end[i] - c->temp_start[i];
        }
        qsort(map, c->num_temps, sizeof(map[0]), node_to_temp_priority);
        for (uint32_t i = 0; i < c->num_temps; i++) {
                temp_to_node[map[i].temp] = i;
        }
```

大概意思就是，计算每个temp变量的生命周期长度，根据生命周期长度进行一个排序，然后得到一个temp变量到顶点的映射。由于Mesa中的寄存器分配器优先分配低标号寄存器给低标号顶点（temp变量），这导致程序中最初的变量会被分配给累加器，一般是shader程序的输入参数，这使得shader参数长期占用累加器。通过排序，做这个重映射，使得生命周期短的temp变量优先分配累加器，尽可能多的利用累加器，使整个程序的寄存器冲突减小，分配更高效。从而提升了整体性能。

顶点的class定义实现比较简单，驱动为每一个temp寄存器（即顶点）定义一个class_bit变量，形成一个数组。每一个class_bit的值是下面几个flags的组合：

```c
#define CLASS_BIT_A			(1 << 0)
#define CLASS_BIT_B			(1 << 1)
#define CLASS_BIT_R4			(1 << 2)
#define CLASS_BIT_R0_R3			(1 << 4)
```

初始情况下，这四个flag是都设置了的，每一个flag实质上表示这个temp寄存器可以使用的物理寄存器类型，如`CLASS_BIT_R4`表示这个temp寄存器可以使用寄存器R4。随后遍历所有的指令，根据规则消减相应的flag，我们首先可以看到对R4的处理：

```c
                if (qir_writes_r4(inst)) {
                        /* This instruction writes r4 (and optionally moves
                         * its result to a temp), so nothing else can be
                         * stored in r4 across it.
                         */
                        for (int i = 0; i < c->num_temps; i++) {
                                if (c->temp_start[i] < ip && c->temp_end[i] > ip)
                                        class_bits[i] &= ~CLASS_BIT_R4;
                        }

                        /* If we're doing a conditional write of something
                         * writing R4 (math, tex results), then make sure that
                         * we store in a temp so that we actually
                         * conditionally move the result.
                         */
                        if (inst->cond != QPU_COND_ALWAYS)
                                class_bits[inst->dst.index] &= ~CLASS_BIT_R4;
                } else {
                        /* R4 can't be written as a general purpose
                         * register. (it's TMU_NOSWAP as a write address).
                         */
                        if (inst->dst.file == QFILE_TEMP)
                                class_bits[inst->dst.index] &= ~CLASS_BIT_R4;
                }
```

回顾一下R4，R4是一个只读的累加器，Spec中可以看到，这个累加器在大部分情况下用于读取外部单元给出的数据。之所以称作只读的，是因为QPU的ALU指定对于写入的operand没有像read operand那样的mux，而是使用32-37号寄存器编码累加器，即在A或者B bank中32-37寄存器被映射到累加器，而本应表示R4的36被映射到`TMU_NOSWAP`，这从实质上就杜绝了R4的写入。那么，`qir_writes_r4(inst)`本质上是检测这个指令会不会造成R4的间接写入：

```c
bool
qir_writes_r4(struct qinst *inst)
{
        switch (inst->op) {
        case QOP_TEX_RESULT:
        case QOP_TLB_COLOR_READ:
        case QOP_RCP:
        case QOP_RSQ:
        case QOP_EXP2:
        case QOP_LOG2:
                return true;
        default:
                return false;
        }
}
```

很明显，对于会造成R4隐式写入的指令，只要它位于某个temp变量的生命周期中，那么这个temp变量本身就不能被分配给R4寄存器。对于非R4隐式写入的指令，其目标temp寄存器明显不能分配为R4，因为R4是只读的。

随后就是根据指令的限制对指令的operand进行限制。最后根据得到的class_bit数组，对顶点标记其属于的class。

最后，遍历temp变量，根据生命周期添加图的边：

```c
        for (uint32_t i = 0; i < c->num_temps; i++) {
                for (uint32_t j = i + 1; j < c->num_temps; j++) {
                        if (!(c->temp_start[i] >= c->temp_end[j] ||
                              c->temp_start[j] >= c->temp_end[i])) {
                                ra_add_node_interference(g,
                                                         temp_to_node[i],
                                                         temp_to_node[j]);
                        }
                }
        }
```

最后，调用`ra_allocate`，并收集temp到物理寄存器的映射。

# 指令构造器

无论如何，shader最终都要编译成QPU能够认识的二进制指令然后执行。那么其中必不可少的一个功能就是生成一条指令的二进制表示，一个`uint64_t`类型的整数。其方式实现比较简单，可以轻松想想出来，即通过提供各种字段的enum，和字段的配置结构，一级一级的提供构造指令所用的函数。到最后，提供一套比较类似的接口，举例如下：

```c
uint64_t
qpu_a_MOV(struct qpu_reg dst, struct qpu_reg src);
```

为了生成字段，首先要定义字段的位置，长度。`vc4_qpu_defines.h`中包含了相关的enum定义，简单先来看下：

```c
#define QPU_MASK(high, low) ((((uint64_t)1<<((high)-(low)+1))-1)<<(low))
/* Using the GNU statement expression extension */
#define QPU_SET_FIELD(value, field)                                       \
        ({                                                                \
                uint64_t fieldval = (uint64_t)(value) << field ## _SHIFT; \
                assert((fieldval & ~ field ## _MASK) == 0);               \
                fieldval & field ## _MASK;                                \
         })

#define QPU_GET_FIELD(word, field) ((uint32_t)(((word)  & field ## _MASK) >> field ## _SHIFT))

#define QPU_UPDATE_FIELD(inst, value, field)                              \
        (((inst) & ~(field ## _MASK)) | QPU_SET_FIELD(value, field))
```

本质上，通过两个抽象确定字段在指令中的位置：`_MASK`，`_SHIFT`。然后`QPU_GET_FILED`，`QPU_SET_FILED`通过这两个宏将数值更新到字段上。可以通过如下类似的方式，更改一个指令的相应字段：

```c
        inst |= QPU_SET_FIELD(QPU_SIG_NONE, QPU_SIG);
        inst |= QPU_SET_FIELD(QPU_A_OR, QPU_OP_ADD);
        inst |= QPU_SET_FIELD(QPU_R_NOP, QPU_RADDR_A);
        inst |= QPU_SET_FIELD(QPU_R_NOP, QPU_RADDR_B);
```

vc4使用`struct qpu_reg`表示一个寄存器，注意指令中的寄存器字段是需要mux填充的，也就是指令中有专门的一个mux字段，表明指令操作的是哪种类型的存储空间（累加器，寄存器集合A，寄存器集合B）：

```c
struct qpu_reg {
        enum qpu_mux mux;
        uint8_t addr;
};
```

有多个构造函数可以方便的快速构造对应类型的寄存器表示，这里不列举了。但事实上这些helper本质上还是给更上一层的接口使用的，即直接生成指令表示的函数。以`qpu_NOP`举例：

```c
uint64_t
qpu_NOP()
{
        uint64_t inst = 0;

        inst |= QPU_SET_FIELD(QPU_A_NOP, QPU_OP_ADD);
        inst |= QPU_SET_FIELD(QPU_M_NOP, QPU_OP_MUL);

        /* Note: These field values are actually non-zero */
        inst |= QPU_SET_FIELD(QPU_W_NOP, QPU_WADDR_ADD);
        inst |= QPU_SET_FIELD(QPU_W_NOP, QPU_WADDR_MUL);
        inst |= QPU_SET_FIELD(QPU_R_NOP, QPU_RADDR_A);
        inst |= QPU_SET_FIELD(QPU_R_NOP, QPU_RADDR_B);
        inst |= QPU_SET_FIELD(QPU_SIG_NONE, QPU_SIG);

        return inst;
}
```

总之，接口的形式为类似`qpu_<INSTR>`的函数，其返回值为uint64_t，即真正构造出来的机器指令。而函数的参数则根据指令的不同而不同，一般为相应的寄存器参数。由于ALU指令大多有cond_add和sig字段，所以提供了`qpu_set_cond_add`和`qpu_set_sig`来设置相应的字段。

# Code Emitter

Code emitter的本质工作是实现从QIR到二进制可执行QPU（shader）程序的转换。其实现位于`vc4_qpu_emit.c`文件中，主要接口为`vc4_generate_code`与`vc4_generate_code_block`。其实现基本逻辑就是遍历所有的QIR指令，然后做简单的一对一翻译，有时候会根据需要插入额外指令。

## Queue

这里的queue即指队列，也指一个接口。shader程序实际上就是一堆`uint64_t`指令的集合，vc4中，这些指令被放到一个队列（链表）中。`queue`接口的实现如下：

```c
static void
queue(struct qblock *block, uint64_t inst)
{
        struct queued_qpu_inst *q = rzalloc(block, struct queued_qpu_inst);
        q->inst = inst;
        list_addtail(&q->link, &block->qpu_inst_list);
}
```

本质上就是创建一个链表元素，然后添加到链表尾部。除此之外，由于实现需要，还提供了两个接口，用于更改链表最后一个元素中指令的Condition字段：

```c
static void
set_last_cond_add(struct qblock *block, uint32_t cond)
{
        *last_inst(block) = qpu_set_cond_add(*last_inst(block), cond);
}

static void
set_last_cond_mul(struct qblock *block, uint32_t cond)
{
        *last_inst(block) = qpu_set_cond_mul(*last_inst(block), cond);
}
```

## vc4_generate_code_block

函数原型如下：

```c
static void
vc4_generate_code_block(struct vc4_compile *c,
                        struct qblock *block,
                        struct qpu_reg *temp_registers);
```

首先明确code block实际上类似与编译器中的basic block的概念，即一个顺序执行（没有分支跳转）的最小代码单元，函数的第二个参数block即为这样一个basic block单元，由QIR组成。函数的第三个参数为前面分析过的寄存器分配器的输出，即物理寄存器到temp寄存器的一个分配。有了这些前置条件，即可开始分析代码。

我们知道一个SSA形式的指令有类似如下的样式：

```
dest = op (src0, src1, src2, ......)
```

而我们要做的实质上就是将这种形式的QIR转换成QPU能够执行的指令。其中最简单的就是op的转换，QIR是为QPU设计的，其基本的`QOP_*` opcode本质上可以简单转换为QPU执行的opcode。随后就是src和dest的转换，他们的类型为QIR中的`struct qfile`：

```c
struct qreg {
        enum qfile file;
        uint32_t index;
        int pack;
};
```

本质上也是非常贴合QPU指令级的设计，我们以`QFILE_TEMP`和`QFILE_VARY`为例，简单分析。`QFILE_TEMP`为QIR中的临时寄存器表示，在前面的寄存器分配过程中，我们得到临时寄存器到物理寄存器的映射，并且函数参数也传入了映射结果，所以对于`QFILE_TEMP`的转换非常简单：

```c
                        case QFILE_TEMP:
                                src[i] = temp_registers[index];
                                if (qinst->src[i].pack) {
                                        assert(!unpack ||
                                               unpack == qinst->src[i].pack);
                                        unpack = QPU_SET_FIELD(qinst->src[i].pack,
                                                               QPU_UNPACK);
                                        if (src[i].mux == QPU_MUX_R4)
                                                unpack |= QPU_PM;
                                }
                                break;
```

简单读取这个映射即可。对于`QFILE_VARY`来说更为简单，因为这个寄存器的值实际上是映射到寄存器bank B上的，只需要简单配置即可：

```c
                        case QFILE_VARY:
                                src[i] = qpu_vary();
                                break;
```

实际上比较复杂的是`QFILE_VPM`，后面单独来一小结分析。

经过上面的分析，整体逻辑还是非常简单的，后面仅挑比较费解的细节分析。

## VPM

`QFILE_VPM`的写入非常简单，而其读取比较复杂。

```c
                        case QFILE_VPM:
                                setup_for_vpm_read(c, block);
                                assert((int)qinst->src[i].index >=
                                       last_vpm_read_index);
                                (void)last_vpm_read_index;
                                last_vpm_read_index = qinst->src[i].index;
                                src[i] = qpu_ra(QPU_R_VPM);
                                break;
```

VPM是外部的硬件模块，其读取方式比较复杂，需要经过配置。简单来说，读取时要对FIFO进行配置（写入一个寄存器），随后通过读取另一个寄存器得到VPM从FIFO传出的值。

## vc4_generate_code

函数首先调用上面提到的寄存器分配器接口对寄存器进行分配操作：

```c
        struct qpu_reg *temp_registers = vc4_register_allocate(vc4, c);
        if (!temp_registers)
                return;
```

然后，对于Coord和Vertex shader，先配置好VPM的输出：

```c
        switch (c->stage) {
        case QSTAGE_VERT:
        case QSTAGE_COORD:
                c->num_inputs_remaining = c->num_inputs;
                queue(start_block, qpu_load_imm_ui(qpu_vwsetup(), 0x00001a00));
                break;
        case QSTAGE_FRAG:
                break;
        }
```

接下来遍历所有的blocks，然后调用`vc4_generate_code_block`：

```c
        qir_for_each_block(block, c)
                vc4_generate_code_block(c, block, temp_registers);
```

最后函数调用`qpu_schedule_instructions`，剩下的都是细节，主要与QPU硬件对二进制可执行程序的一些强制要求有关。

# 指令调度器

TODO

