+++
title = "MESA源码分析：VC4驱动"
date = 2023-01-06


tags = ["mesa", "gallium3d", "vc4"]

+++

本文分析vc4驱动的用户态部分，即Mesa中的用户态驱动实现。由于vc4是绝佳的学习范本，可以极大的促进对Mesa的理解，对vc4驱动的彻底分析理解是非常有必要的。本文从如下角度分析vc4的用户态驱动实现：

* 硬件工作原理、文档以及接口
* vc4驱动的内核驱动与用户态驱动交互
* vc4驱动的shader编译器实现
* vc4驱动如何与Mesa以及gallium3D框架集成

# 硬件分析

分析硬件驱动的时候一定要理解硬件。树莓派3作为一个嵌入式SoC平台，GPU这个词与PC平台并不是等价的。在PC平台，我们常将GPU称为显卡，这是因为PC平台使用PCIE总线，而GPU一般封装在一个独立的PCIE扩展卡上。同时，GPU这个词在PC上也泛指GPU、连同VPU和display controller，他们通常被做在同一块芯片上。而在嵌入式SoC平台上，由于基本上所有的IP都在SoC中，且可能来自不同的厂商，他们之间的差别就明显了，不再是一个整体。

* Display Controller。我们常说的显示控制器，实质上就是内核KMS API的硬件载体，是控制与驱动显示设备的pipeline。显示控制器从一个或多个framebuffer中获取数据，对其进行处理（缩放，旋转，color keying，叠加），然后输出到显示设备上（液晶显示屏，HDMI、DP口外接显示器等）。
* VPU。视频处理单元，一般是用户编码、解码视频流的硬件处理单元。
* GPU。这里常指进行3D加速的图形加速单元，就是OpenGL、Vulkan等3D API的硬件载体。
* 显存。在PC平台，显卡上一般板载独立显存，而在SoC上，使用与CPU共用的系统内存作为显存。

对于树莓派3平台，我们先不管VPU与Display Controller，其GPU是有比较齐全的[官方文档](https://docs.broadcom.com/doc/12358545)的。GPU的工作方式比较特别，需要从几个方面理解。

* 接口。这里的接口是指从CPU和GPU角度上的接口。对于PC，GPU一般通过PCIE接口与PC进行通信，也就是通过中断、PCI config space与BAR空间的方式进行访问。而对于ARM，一般是通过其他的高速总线实现的，但是一般不带独立显存，而直接使用物理内存作为显存。不论那种实现，GPU都是具备访问物理内存的能力的。事实上，GPU的接口可以粗略划分为寄存器与command ring。
* command ring。寄存器接口很好理解，对于PCIE，一般是某个BAR上映射的寄存器，而对于ARM即是MMIO映射的设备寄存器。command ring是一种机制，GPU和CPU通过ring buffer进行通信。简单来说，CPU申请一块物理内存，并在其中写好需要执行的GPU（认识）指令，在一块内存不够大的情况下可以通过指针串起来。CPU可以通过设备寄存器等接口告知GPU这段内存的存在，告诉其开头和结尾，此时GPU可以通过DMA读取该段内存，开始执行指令。这样做可以使得CPU控制GPU的行为，并让双方高效通信。
* 显存。这里指GPU可以利用的存储空间，一般分为系统内存和独立显存。对于复杂的GPU，一般让其携带MMU，通过页表进行显存管理。与CPU的页表类似，但是需要严格区分系统内存和独立显存，以及显存区域的格式（比如tile显存）。也就是说GPU一般具有比较复杂的内存系统，有虚拟内存，进程等（即上下文的概念）。
* Stream Processor。GPU实质上是由非常多的小处理器组成的，每组小处理器上同一个时刻运行同一个小程序，一般称作kernel。这组小处理器一般高度并行化，共用资源，有自己的指令集和高速缓存。实际上驱动程序的一个任务就是将shader编译成GPU的流处理器可以执行的指令。GPU的编程手册中一般会对stream processor支持的指令集做详细介绍。
* Hardware Scheduler。Stream Processor的管理是一个复杂的工作，一般有专门的硬件单元对Stream Processor进行管理和调度。
* Hardware Context。Stream Processor存在上下文的概念，本质上就是让Stream Processor恢复到某工作状态需要的全部状态。与CPU一样，可以通过将硬件上下文dump到内存中保存上下文，随后将上下文状态装入到硬件中恢复执行状态，实现上下文切换和管理。
* Blitter。Blitter实际上是`Block Image Transmitter`的缩写，本质上是逻辑电路单元，可以快速的将一段内存移动到其他位置上。GPU中一般利用这个单元对texture和bitmap进行传输。

# Control List

Control List在Videocore 4中起到command ring的作用，也就是CPU向GPU提交任务的手段。Videocore 4的手册在Section 9描述了其具体格式，但是没有详细典型用法。简单来说Control List（简称cl）是有多个记录组成的列表，每个记录由一个id开头，确定记录类型，然后紧跟相应格式的数据，大致有下面几种类型：

* 顶点列表
* 状态数据
* 系统控制操作
* 分支、跳转操作

多个不同类型的记录组成一个完成的Control List。videocore 4中存在两个独立的引擎，用以处理Control List，分为binner和render，两个引擎并行运行且处理不同类型的Control List，相关资料太多了，详见手册。简单来说，由于videocore 4是比较简单的显卡，command ring是比较简单的形式。驱动工作过程中，只需要根据需要将Control List构造好，再通过配置寄存器将其提交给GPU，GPU就会按照相应的命令进行工作。

简单记录一下MESA中对Control List的实现，主要分成两个部分：

* 基础设施
* 自动生成的代码

基础设施简单来说，就是方便自动代码生成以及提供给外部调用的接口。主要有`struct vc4_cl`：

```c
struct vc4_cl {
        void *base;
        struct vc4_job *job;
        struct vc4_cl_out *next;
        struct vc4_cl_out *reloc_next;
        uint32_t size;
#ifndef NDEBUG
        uint32_t reloc_count;
#endif
};
```

本质上`base`指针指向一个分配好的内存，这段内存使用MESA内部的ralloc进行分配，这个分配器的特点就是支持树型分配，方便释放。每次分配内存时，需要指定一个parent，这里使用该Control List对应的`vc4_job`结构体的内存区域作为parent。释放parent内存时，可以将child内存一起释放，起到高效分配内存的效果。`next`指针指向下一个应该写入的内存区域，本质上就是这个Control List在申请到内存上写入内容的尾巴指针。而申请的大小是由`size`字段记录，可以通过`cl_ensure_space`调用类似remalloc的接口将申请到的内存进行扩展。

除此之外，还提供了类似像`cl_u8`、`cl_u16`这样的接口，用于像缓冲区中填充数据。这些接口实际上并不会直接让外接用到，而是提供给一个由XML文件经过解析后生成的c源文件使用的。简单来说，这个XML文件会为每一中Control List记录定义一个相应的结构体，我们只需要将该结构体填充好，然后调用`cl_emit`宏，即可利用由XML生成的填充函数，将缓冲区按照定义好的格式进行填充。

TODO: reloc

## Job

Job是什么？从Mesa的角度来看，job是可以提交给内核态的任务，使用`struct vc4_job`进行抽象。看过前面内核代码分析的文档的话，应该明白内核提供的接口相当简单，仅仅是一个名为`SUBMIT_CL`的ioctl。因此，`vc4_job`实质上是一个能让内核态将command ring整个驱动起来的一个任务所需要的所有信息。由于job包含所有ioctl需要的信息，工作时，可以直接通过ioctl将job递交给内核，然后在内核中的队列等待执行。原先我们看到，队列管理也是相当简单的，因为videocore 4的引擎一次只能执行一个job。

将`vc4_job`递交给内核的接口为`vc4_job_submit`，本质上是调用了接近同名的ioctl，所以函数本身的行为就是构建ioctl的参数。这里有一个简单的优化：

```c
        if (!job->needs_flush)
                goto done;
... ...
done:
        vc4_job_free(vc4, job);
```

即`vc4_job`中存在一个`needs_flush`字段，标记这个job是否需要提交给内核，没有标记的直接free掉。

很明显job是寄生在context上的概念，是context的一部分。而context在创建的时候，由`vc4_job_init`在context上进行创建。可以看到这个函数简单创建了两个哈希表：

* vc4->jobs - 映射vc4_job_key到vc4_job
* vc4->write_jobs - 映射 vc4_resource到vc4_job，用于检索哪些job向对应的`vc4_resource`写入了

简单来说就是一个cache刷新机制，每当新建job时，必须将原先操作了对应的buffer的job提交给内核。

对于job，我们最常见的一个接口就是`vc4_get_job`和`vc4_get_job_for_fbo`。前者是通用的，而后者是前者的特殊形式。简单来说，`vc4_get_job`接收两个参数，即`cbuf`和`zsbuf`，这二者实际上就组成了一个framebuffer，也就是说，创建的job的操作对象就是这两个buffer。为了追踪这个job，就需要利用到前面提到的两个哈希表了。大部分情况下，驱动都是直接调用`vc4_get_job_for_fbo`，为当前绑定到context的framebuffer创建相应的job。

TODO

## 工作机制

这里没有接触过移动GPU的人会很疑问这里的binning和render到底是个什么东西，为什么要区分这两个阶段。这个区分本质上是因为vc4的pipeline实际上是基于tile的渲染。tiling based rendering在很多地方都有提到，这里只给出名字和基本的工作原理。tile在英文中一般是指大小一样的小块，在这里特指特定大小的像素区域。tile based rendering本质上是一种改进后的渲染方式，这种改进能够很大程度上优化硬件所使用的资源，最大的优化就是显存带宽以及缓存本地性。顾名思义，其本质上就是将整个frame buffer分成多个tile，一般为16x16大小或者32x32大小，然后对每一个tile进行单独渲染，最后将渲染结果组合到一起，形成整个framebuffer。

仅仅使用这种优化其实并不够，还有一个明显的优化：我们可以首先通过vertex shader计算出每一个顶点经过映射后，最终的座标，并得到这个顶点与其他顶点组整的形状是是否会影响到这个tile上渲染的结果。所以，这个优化操作在vc4的硬件中就被抽象成了单独的操作，称作binning，其本质就是完成上述的运算步骤，得到一个tile与其相关的顶点数据。只不过硬件设计上更进了一步，不仅仅完成了上述步骤，还能够自动的生成接下来对这个tile进行渲染动作的control list。

也就是说，binning阶段实际上我们输入了一个control list，而其最终的处理硬件PTB，会向我们返回一组control list，被称作rendering control list，我们只需要进行简单处理，即可重新发送给CLE引擎，让其完成rendering操作。

control list总结起来，数据分成三部分，它的作用是驱动整个pipeline：

* 状态数据，把整个pipeline想象成一个巨大状态机，可以通过control list改变其中的特定状态
* shader，现在都是可编程渲染管线，最起码要指定vertex shader和fragment shader
* primitive list，简单来说就是vertex，和对vertex的解读，这两部分与opengl的API对应

整个vc4硬件引擎可看做一个巨大的状态机，control list中的数据本质上是在更新其中的各个状态，并驱动其执行。因此，仅当我们需要更改状态的时候，才需要添加特定的record，其他情况下，可以使用现有的状态。

具体到control list，他是分成两份的：

* binning
* render

但是，从手册中可以看到，binning过程中会生成相应的render control list，只需要简单组装即可。所以render list的生成和推送是在内核中实现的。因此，我们在用户态实际上就只需要指定binning list和剩下的两部分数据。注意，从手册中可以看到，很多record中引用相应的物理地址，用作DMA，这些物理地址在用户态应该是无法知道的，猜想这些位置会空出来，然后在内核态帮忙补全。

从手册的section 9中可以看到，对于binning类型的control list，其格式有一定要求：

* binning模式的control list以`TILE_BINNING_MODE_CONFIGURATION(112)`类型record开头
* binning模式的control list以`START_TILE_BINNING`标记control list的启动
* binning模式的control list应该以`FLUSH`结尾，将生成的render list刷出

从videocore 4的control list引擎如何工作，我们可以得出驱动程序的设计。简单来说，就是驱动需要追踪control list引擎的状态改变，在状态发生改变时，生成对应的record去修改状态。除此之外，还要精确追踪当前的control list（即vc4_job），在必须进行一次提交的时候，进行一次提交。比如说一次完整的pipeline工作流程，或者连续两个不可合并成一个batch的操作等。

## 代码实现

用户态vc4驱动的核心功能是构建control list，而从内核态相关代码来看，vc4的用户态驱动只需要构建binning control list，简称bcl。vc4硬件中的PTB引擎会替我们生成一部分的render control list，简称rcl。为此，vc4驱动应该实现一套比较方便的API，用于快速生成，表示，导出一个control list，这里主要研究这个实现。

vc4对一个control的表示为`struct vc4_cl`，这个结构体比较简单，贴出：

```c
struct vc4_cl {
        void *base;
        struct vc4_job *job;
        struct vc4_cl_out *next;
        struct vc4_cl_out *reloc_next;
        uint32_t size;
#ifndef NDEBUG
        uint32_t reloc_count;
#endif
};
```

其初始化函数为`vc4_init_cl`，从初始化函数可以看到：base是一段申请的内存，后续可以使用类似realloc的方式进行扩展申请。next和reloc_next依然是指针，注意虽然这里是一个结构体指针，但是这个结构体指针只有声明没有定义，这里是一个trick，用来进行类型检查。size储存了申请的空间大小，而next则保存的是下一个应当写入的字节的位置，也就是这个buffer的数据尾部。

接下来介绍几个helper：

* cl_advance：将指针加上n
* cl_start/cl_end：cl_start返回next指针。而`cl_end`则设置next指针。这两个函数的用法实际上是用来标志操作开头和结尾的，操作以cl_start开头，拿出next指针，然后对next指针进行操作，最后使用cl_end将next指针设置回去。
* cl_u8 cl_u16 cl_u32 cl_ptr等等：向缓冲区尾部写入数据，并调用cl_advance移动指针到对应位置

接下来是重头戏，即`cl_emit`宏，这个宏设计的比较巧妙。事实上，存在一个XML文件，以结构化数据的形式记录了每一个record的id值与其相依的数据字段，vc4驱动通过一个parser解析这个XML文件，然后生成以下符号：

```c
#define cl_packet_header(packet) V3D21_ ## packet ## _header
#define cl_packet_length(packet) V3D21_ ## packet ## _length
#define cl_packet_pack(packet)   V3D21_ ## packet ## _pack
#define cl_packet_struct(packet)   V3D21_ ## packet
```

其中，最重要的是`_pack`函数，它的内容是直接调用上面提到的helper，向缓冲区中填写相关的数据。这时候`cl_emit`的实现就比较明了了：

```c
#define cl_emit(cl, packet, name)                                \
        for (struct cl_packet_struct(packet) name = {            \
                cl_packet_header(packet)                         \
        },                                                       \
        *_loop_terminate = &name;                                \
        __builtin_expect(_loop_terminate != NULL, 1);            \
        ({                                                       \
                struct vc4_cl_out *cl_out = cl_start(cl);        \
                cl_packet_pack(packet)(cl, (uint8_t *)cl_out, &name); \
                VG(VALGRIND_CHECK_MEM_IS_DEFINED(cl_out,         \
                                                 cl_packet_length(packet))); \
                cl_advance(&cl_out, cl_packet_length(packet));   \
                cl_end(cl, cl_out);                              \
                _loop_terminate = NULL;                          \
        }))                                                      \
```

其利用for循环，首先构造了一个临时变量，表示record的header结构体，这个结构体要求使用者在for循环语句的body中进行填充。这个for循环精心构造，使得其body只会被调用一次，且调用完成之后，即会调用`_pack`函数，根据使用者填充的header结构体的值，将构造好的数据写入到缓冲区中。举个例子：

```c
 cl_emit(bcl, FLAT_SHADE_FLAGS, flags) {
      flags.flat_shade_flags = 1 << 2;
 }
```

除此之外，还有一个比较重要的实现就是reloc了。从手册中我们知道，很多record中实际是可以引用物理地址的，但很明显，从用户态驱动的角度，它并不能知道虚拟地址对应的物理地址。因此，这部分内容需要内核帮助处理，而用户态只需要递交给内核态record中引用的数据，从实现的角度上来看，这些数据都是放在某一个BO中的。因此，从实际出发，这些物理地址上都是填写的BO的handle（32位整数）以及相应的offset。

# Gallium3D

## Draw

接下来分析一些常见的`pipe_context`上的回调函数，起到入手control list分析的作用。目前仅仅依靠一份比较抽象的文档，在没有深厚驱动背景的情况下，是不容易进行深入分析，需要要利用一切可以利用的资源，利用残片拼出全局。

## Clear

```c
static void
vc4_clear(struct pipe_context *pctx, unsigned buffers, const struct pipe_scissor_state *scissor_state,
          const union pipe_color_union *color, double depth, unsigned stencil);
```

这个clear回调函数实际上对应着opengl中一个非常简单的操作，即`glClear`，建议先查看[文档](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glClear.xhtml)。简单来说，这是一个`state-using`函数，其目的是根据先前使用`glClearColor`、`glClearDepth`或者`glClearStencil`函数设置的三个值（填充值），将对应的buffer进行clear操作。

从函数的参数上可以很明显的得出如下信息：

* buffer参数实际上就是`glClear`的bitmask，可以由COLOR,DEPTH,STENCIL组成
* 后续的`color`,`depth`，`stencil`三个参数实际上就是对应的填充值，仅当bitmask中存在对应位的时候生效
* `scissor_state`实际上对应`glClear`中的scissor box，限制生效的区域

所以总结起来，该回调函数的操作就是根据buffer bitmask将由`pipe_scissor_state`指定的区域填充成指定的值。



### shader

shader在list中实际上也是通过一个record体现的，叫做`SHADER_REC`。shader这部分还需要查手册，并理解相关的编译器，后面再完善。

### 状态数据

`vc4_job`中存在一个dirty标志，当标志特定状态需要更新时，生成的binning list就会添加特定的record

### primitive list

前面提到primitive list包含三部分内容，vertex数据本身，对数据的解读，以及render命令本身。我们知道gallium3D中通过resource抽象一块显存区域。很明显，我们需要使用resource存储vertex数据，简单来说就是一个BO。

## draw_vbo

draw_vbo回调函数简直是一个大杂烩，简单来说，它需要直接实现opengl接口多种不同的render指令：

* 直接绘制（direct draw）
* 间接绘制（indirect draw）
* 实例化绘制（instanced draw）
* 基于索引的绘制（indexed draw）
* MultiDraw

简单来说，opengl的render指令分成基于索引的与非基于索引的，即`glDrawElements`和`glDrawArrays`及其变体。在此之上，存在实例化绘制与间接绘制的变体，也存在添加简单偏移量的`BaseVertex`变体。`draw_vbo`必须完整的处理这些情况，所以其输入参数是比较复杂的。

```c
   void (*draw_vbo)(struct pipe_context *pipe,
                    const struct pipe_draw_info *info,
                    unsigned drawid_offset,
                    const struct pipe_draw_indirect_info *indirect,
                    const struct pipe_draw_start_count_bias *draws,
                    unsigned num_draws);
```

现在我们来从一个比较逻辑化的方式来理解这个函数的参数。首先我们知道opengl中存在类似`glMultiDrawArrays`这样的函数，其本质就是将多个`glDrawArrays`压缩成一个调用，一次完成。其参数就是简单的将`glDrawArrays`替换成了一个数组。这个函数对应到`draw_vbo`回调函数，就是`draws`与`num_draws`两个参数。

```c
struct pipe_draw_start_count_bias {
   unsigned start;
   unsigned count;
   int index_bias; /**< a bias to be added to each index */
};
```

本质上，`start`与`count`即对应`glDrawArrays`的两个参数，而`index_bias`则对应`BaseVertex`的变体。驱动可以根据硬件的特性选择是否对这个场景进行优化，也可以选择直接调用`util_draw_multi`函数，将MultiDraw单纯的拆成一个一个地执行。

对于instanced draw，我们可以把普通的`glDrawArrays`一般化为instance数为一的instanced draw。所以，`info->instace_count`参数用于标记instance的个数，如果不为一，则需要进行额外处理。`indirect`参数是否为NULL直接表明这个绘制是否是间接的，对于间接的绘制，`indirect`参数中储存了相应buffer的信息。

## vc4_draw_vbo

该函数即为对vc4的`draw_vbo`回调函数的实现。首先看到这个函数没有对multidraw进行优化：

```c
        if (num_draws > 1) {
                util_draw_multi(pctx, info, drawid_offset, indirect, draws, num_draws);
                return;
        }
```

直接调用`util_draw_multi`将multidraw拆分成单个的draw，然后继续调用draw_vbo回调函数进行处理。



## shader实现总览

这里还是先讲一下架构吧，vc4总体利用了MESA提供的各种helper。shader本身在gallium是被看作一个CSO（Constant State Object）的，本质上具有独立的状态对象，而不是直接将状态记录到context中。作为CSO，具有独立的状态对象，是具有三种类型的相关函数的，状态对象可以使用这三个函数分别创建，绑定，删除。`vc4_program.c`文件中的相关实现，即负责管理这个Shader CSO对象。这个文件名称的由来肯定是因为Shader实际在OpenGL中是被链接成一个Program进行管理的。

回到Mesa，我们知道用户态驱动的一个主要的功能就是将Shader Language编写的shader程序，编译成硬件能够在流处理器上执行的二进制程序。事实上，Mesa实现了共用的编译器前端功能，而后端明显需要各个硬件的驱动程序自行实现。Mesa提供的编译器前端首先将Shader程序进行处理，生成一种称作TGSI的中间表示，这个中间表示进一步被翻译成称作NIR的一种基于SSA的中间表示。很明显vc4需要读取NIR中间表示，然后转换成自己的IR，最后生成相应的可执行程序。这个vc4自己的IR被称作QIR，与其硬件结构的QPU相对应。

最后来讲一下编译器入口在哪里。以Vertex Shader为例，在经过绑定操作，即`vc4_vp_state_bind`回调函数后，这个shader即绑定到了context中：

```c
vc4_vp_state_bind(struct pipe_context *pctx, void *hwcso)
{
        struct vc4_context *vc4 = vc4_context(pctx);
        vc4->prog.bind_vs = hwcso;
        vc4->dirty |= VC4_DIRTY_UNCOMPILED_VS;
}
```

在我们调用draw_vbo时，则会在生成control list的时候进行shader状态的更新，本质上是通过调用`vc4_update_compiled_shaders`进行的：

```c
bool
vc4_update_compiled_shaders(struct vc4_context *vc4, uint8_t prim_mode)
{
        vc4_update_compiled_fs(vc4, prim_mode);
        vc4_update_compiled_vs(vc4, prim_mode);

        return !(vc4->prog.cs->failed ||
                 vc4->prog.vs->failed ||
                 vc4->prog.fs->failed);
}
```

函数进一步调用了`vc4_get_compiled_shader`获取编译后的shader，如果此时shader没有被编译过，则调用`vc4_shader_ntq`进行编译操作。



## resource

这里实际上就是`struct pipe_resource`，这是用来表示资源的对象，具象一点就是vertex buffer和texture这种。本质上应该是一个BO，这是因为目前内核DRM驱动大部分使用GEM作为显存管理的前端接口。所以大部分的驱动的`pipe_resource`子类实际上就是额外附加了一个bo。

以前分析过GEM相关的代码，实际上BO在用户态就是一个handle，一个整数。但是vc4驱动自己wrap了一个`struct vc4_bo`类型，起到管理特定资源的作用，本质上是在BO的基础上实现一些优化，比如缓存map等。

```c
struct vc4_bo {
        struct pipe_reference reference;
        struct vc4_screen *screen;
        void *map;
        const char *name;
        uint32_t handle;
        uint32_t size;

        /* This will be read/written by multiple threads without a lock -- you
         * should take a snapshot and use it to see if you happen to be in the
         * CL's handles at this position, to make most lookups O(1).  It's
         * volatile to make sure that the compiler doesn't emit multiple loads
         * from the address, which would make the lookup racy.
         */
        volatile uint32_t last_hindex;

        /** Entry in the linked list of buffers freed, by age. */
        struct list_head time_list;
        /** Entry in the per-page-count linked list of buffers freed (by age). */
        struct list_head size_list;
        /** Approximate second when the bo was freed. */
        time_t free_time;
        /**
         * Whether only our process has a reference to the BO (meaning that
         * it's safe to reuse it in the BO cache).
         */
        bool private;
};
```

vc4驱动实现了一个BO cache，每当申请`struct vc4_bo`对象时，都会先从这个cache中查找。这个BO cache是per screen的，也就是所有的context共用一个BO cache。

```c
        struct vc4_bo_cache {
                /** List of struct vc4_bo freed, by age. */
                struct list_head time_list;
                /** List of struct vc4_bo freed, per size, by age. */
                struct list_head *size_list;
                uint32_t size_list_size;

                mtx_t lock;

                uint32_t bo_size;
                uint32_t bo_count;
        } bo_cache;
```

简单来看就是两个free list，分别按大小和存在时长排序，除此之外还记录了BO cache的基础信息，大小和内部的BO个数。

## records

CLIP_WINDOW

CONFIGURATION_BITS

CLIPPER

VIEWPORT_OFFSET

FLAT_SHADE_FLAGS

SHADER_RECORD

