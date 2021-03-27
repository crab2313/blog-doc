+++
title = "DRM显示框架分析"
date = 2021-03-26
tags = ["kernel", "drm"]
+++

真的没有想到真的有人能看到这篇东西，所以我准备抽出时间把这个东西整理完。本文是我阅读内核DRM子系统的一些随笔，其目的是记录我对内核代码的阅读过程和自己的一些理解，希望景嘉微的大佬们多多指正。

接触linux桌面比较多的人一定会对DRM这个名字比较熟悉。网上也有大把资料在解释DRM到底是什么。DRM主要分为KMS与Render两大部分，本文实质上是在分析DRM中KMS相关的框架实现。而Render相关的API是特定于驱动的，内核并不为用户态提供一个通用的IOCTL接口。如果后续时间充足，我将会分析一下VC4的Render API，以及相应的用户态实现。从功能上讲，KMS负责搭建显示控制器的pipeline，并控制显示硬件将图像缓冲区scanout到屏幕上，而如何加速生成framebuffer中的内容则是3D引擎（即Render API）负责的事情。

对于KMS，有如下要素：

* 显存管理器
* modesetting API
* 显示API

在研究初期，研究案例最好硬件无关，且比较容易懂原理，且能够正常运行，可以参考virtio-gpu和QXL虚拟显卡。等到对内核DRM子系统有一定的理解后，可以分析简单的显卡硬件，如VC4（树莓派3B的显卡）。

# 阅读路径

需要看的东西有点多，甚至说比较乱。先列举一下：

内核中的内容：

* DRM驱动通用代码：包括GEM，KMS
* AMD显卡相关的代码：AMDGPU，RADEON，AMDKFD（通用计算ROCM框架内核驱动）

用户态代码：

* MESA： OpenGL state tracker， gallium 3D， vulkan， egl（重点），gbm
* libdrm：基本为内核提供的IOCTL的wrapper

我想要重点理解的部分：

* context到底是什么？如OpenGL和egl创建的context
* mesa的架构
* wayland渲染的基本原理
* DRI到底由什么构成

这里我觉得还是先从MESA这里着手，毕竟内核驱动缺少文档，且我对接口层到底怎么用还是不是很熟悉。看一下简单的DUMB驱动如何实现也是一个理解KMS比较好的方法。

# GEM

## GEM与TTM之间的关系

TTM是内核最初的DRM显存管理器，其设计思想是试图为所有的显卡驱动提供一个公共的API。TTM后面被认为是失败的，其API与实现复杂不可控，没有人愿意用他。后来intel吸取教训，设计了GEM，其设计较为灵活，只提供基本的实现，部分功能需要驱动程序通过驱动自定的接口进行扩展。GEM与TTM在特性上的主要区别为GEM不支持管理独立显存，只支持UMA，而TTM两种都支持。目前的DRM驱动程序中，基本都使用GEM作为前端为用户态提供接口，而涉及到管理独立显存的时候，则借助TTM作为后端实现管理功能。

## 框架初始化

驱动程序可以自己选择是否使用GEM框架，如果选择使用，则需要在注册`drm_driver`到DRM core中时，在`driver_features`里设置DRIVER_GEM标志。在驱动加载时，DRM框架会自动初始化GEM。

## handle与name

handle是gem_object的引用，其作用域被限制在drm文件描述符内，用于用户态程序引用内核态的drm_gem_object。当drm文件描述符被关闭时，所有的handle都会被关闭。而name则是drm_gem_object的名称，为一个32-bit整数。name的作用域是全局的，因此被直接保存在drm_gem_object内。默认情况下drm_gem_object的name都为0，表示其是未命名的。用户态可以通过FLINK为drm_gem_object命名，之后系统内的其他进程可以通过对应的name对该drm_gem_object进行访问。

handle的创建是创建GEM对象的一个步骤。目前GEM对象的创建一般是通过设备相关的API进行实现的，驱动程序可以通过`drm_gem_handle_create`函数从对应drm文件中创建一个handle返回给用于态。关于handle，gem core中还提供了`drm_gem_handle_delete`和`drm_gem_handle_lookup`函数。事实上，handle的管理是通过idr实现的。

## drm_gem_object

该对象是GEM内存管理的核心。GEM目前提供的功能是不完全的，部分空缺需要驱动自行填补，因此GEM框架要求驱动在`drm_gem_object`的基础上实现自己的GEM对象。这个操作实际上就是将`drm_gem_object`嵌入到驱动自己定义的`{driver}_gem_object`中。`drm_gem_object`定义在`include/drm/drm_gem.h`中，且有详细的注释，这里不再赘述，重点分析一些字段。

`filp`是一个指向`struct file`的指针，要理解它的作用就必须理解真正的内存是如何分配的。前面提到了GEM只支持UMA，即显卡使用RAM做为显存，这就又涉及到了两种情况。在PC中由于IOMMU的存在，显卡并不强制要求连续的物理内存，因此GEM可以使用SHMFS做为存储后端，此时filp指针就指向对应的文件描述符。而在嵌入式应用场景下，大部分设备没有IOMMU，因此必须要求连续的物理内存，此时filp为NULL，驱动通过CMA申请到连续的物理内存用作存储后端。注意，GEM框架并不负责管理存储后端，只提供了一些基本的helper，而存储的分配与释放完全由驱动程序控制。

TODO： 研究一下SHMFS

`vma_node`简单来说是保存了这个object的mmap偏移量。这里又需要提及`drm_gem_object`向用户态的映射问题。GEM目前基本上是通过两种方式实现MAP gem_object到用户态的：设备相关IOCTL和基于drm文件描述符的mmap。后者通过一个虚拟的偏移量确定mmap究竟在映射哪一个`drm_gem_object`。该机制的实现细节需要单独讨论。

`dma_buf`字段是一个`struct dma_buf`类型的字段。这里又设计drm-prime框架的细节了。简单来说，通过name来实现进程间共享`drm_gem_object`存在显而易见的安全性问题，毕竟name是全局的。攻击者可以通过特定的pattern推测或者枚举`drm_gem_object`的name，达到窃取数据的目的。后续GEM集成了dma_buf，可以通过dma_buf实现共享。

## 用户态映射

用户态映射实质上就是将`drm_gem_object`描述的存储空间映射到用户态进程的虚拟地址空间，让用户态进程可以随机读写。正如前面提到的，这里只分析GEM提供的方式，即基于drm文件描述符的mmap。我们知道mmap系统调用的原型如下：

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset); 
```

GEM框架可以为每一个`drm_gem_object`绑定一个特定虚拟offset，通过offset辨别需要映射到用户态的`drm_gem_object`。该机制需要drm文件描述符与`drm_gem_object`的共同协作，前面看到`drm_gem_object`中提供了vma_node用于保存该信息，驱动程序需要自行调用`drm_gem_create_mmap_offset`为一个`drm_gem_object`注册一个虚拟offset。虚拟offset需要通过设备相关的IOCTL传递给用户态。`drm_device`中存在vma_offset_manager用于统一管理mmap的虚拟offset，其后端基本上为drm_mm与红黑树缓存的组合。

# DUMB

(TODO)

# GBM

前面看到基于GEM的驱动对外是没有提供统一的内存管理接口的，至少Buffer Object创建销毁等操作是需要自行提供设备相关的即口进行实现的。用户态没有统一的接口对缓冲区进行管理，这导致某些特定用户态程序的开发的困难，如wayland compositor。MESA项目提供了libgbm，抽象并实现了一个通用的buffer管理API。这里记录对该API进行的探讨。

## Note

经过代码分析，gbm实际上来源于MESA内存OpenGL实现的`internal/drm_interface.h`，也就是Mesa OpenGL实现的一个私有兼容层。

## gbm_device

`gbm_device`是DRM设备的抽象，管理所有的分配出来的BO（Buffer Object）。自然而然：

* 一个`gbm_device`创建出来的BO的生命周期与该`gbm_device`绑定
* `gbm_device`与特定DRI设备绑定

`gbm_create_device`负责从一个DRM设备中创建`gbm_device`：

```c
struct gbm_device* gbm_create_device(int fd);
```

这里的fd自然是打开`/dev/dri/card*`时的文件描述符。

## gbm_bo

`gbm_bo_create`函数用于创建BO，其原型如下：

```c
struct gbm_bo *
gbm_bo_create(struct gbm_device *gbm,
              uint32_t width, uint32_t height,
              uint32_t format, uint32_t flags);
```

函数非常简单，只需要指定BO的长宽，格式以及标志（enum gbm_bo_flags）。GBM提供了一些基本的helper用于获取BO的相关属性。除此之外比较重要的操作就是将BO映射到用户态地址空间了：

```c
void *gbm_bo_map (struct gbm_bo *bo, uint32_t x, uint32_t y, uint32_t width, uint32_t height, uint32_t flags, uint32_t *stride, void **map_data);
```

可以看到该函数提供了将BO中的特定二维区域映射到功能。

# KMS

主要分为用户态接口的使用，内核提供的框架和通用接口。

KMS将整个显示pipeline抽象成以下几个部分：

* framebuffer
* plane
* crtc
* encoder
* connector

其中每一个部分的含义可以参考内核文档，这里不赘述，这里只分析其在内核框架中是如何实现的。

## 对象管理

对于这几个对象，DRM框架将其称作“对象”，有一个公共的基类`struct drm_mode_object`，这个几个对象都由这个基类扩展而来。事实上，这个基类扩展出来的子类并不是只有上面提到的几种。

```c
struct drm_mode_object {
  uint32_t id;
  uint32_t type;
  struct drm_object_properties *properties;
  struct kref refcount;
  void (*free_cb)(struct kref *kref);
};
```

其中id和type分别为这个对象在KMS子系统中的ID和类型（即上面提到的几种）。注意所有的`drm_mode_object`的id共用一个namespace，保存在`drm_device->mode_config.object_idr`中。因此，框架提供了`drm_mode_object_find`函数用于查找对应id的对象。当前DRM框架中存在如下的对象类型：

```c
#define DRM_MODE_OBJECT_CRTC 0xcccccccc
#define DRM_MODE_OBJECT_CONNECTOR 0xc0c0c0c0
#define DRM_MODE_OBJECT_ENCODER 0xe0e0e0e0
#define DRM_MODE_OBJECT_MODE 0xdededede
#define DRM_MODE_OBJECT_PROPERTY 0xb0b0b0b0
#define DRM_MODE_OBJECT_FB 0xfbfbfbfb
#define DRM_MODE_OBJECT_BLOB 0xbbbbbbbb
#define DRM_MODE_OBJECT_PLANE 0xeeeeeeee
#define DRM_MODE_OBJECT_ANY 0
```

从`drm_mode_object`的定义中即可发现其实现了两个比较重要的功能：

* 引用计数及生命周期管理
* 属性管理

属性在DRM中由`struct drm_property`表示，其本质是一个`DRM_MODE_OBJECT_PROPERTY`类型的`drm_mode_object`。一个`drm_mode_object`的所有属性保存在其内部的`drm_object_properties`中，其实现如下：

```c
struct drm_object_properties {
  int count;
  struct drm_property *properties[DRM_OBJECT_MAX_PROPERTY];
  uint64_t values[DRM_OBJECT_MAX_PROPERTY];
};
```

可以看到每一个对象最多可以有24个属性。这里注意一个实现细节，`drm_property`表示一个属性对象，描述属性的类型（如整形，range，浮点数等）、名称和取值范围（约束）。`drm_object_properties`中的properties保存属性的类型，而`values`保存对应类型的值。这是因为同一类型的对象基本上都共有特定名称和类型的属性，独立的属性对象使得我们不需要为在每一个对象中都保存同样的属性名称和类型。对象的属性可以通过`drm_object_property_*`函数操作。

## Atomic Mode Setting

先写一下我的理解，看看到最后读完代码有什么新收获没有。

首先Atomic Mode Setting是DRM子系统最近的一次比较大的改动，其目的是填补当前API的不足。由于原先的API不支持同时更新整个DRM显示pipeline的状态，因此KMS过程中会出现一些中间状态，容易造成开发者不希望看见的结果，影响用户体验。同时，原先的KMS接口也不支持回滚，需要应用程序自己记录原先的配置状态，Atomic Mode Setting也解决了这个问题。

由于Atomic Mode Setting是新出现的API，为了解决用户态程序的兼容性问题，Atomic Mode Setting的接口被隐藏起来，只有用户态程序显式告知DRM层其支持Atomic Mode Setting时它的接口才会暴露出来。目前主流的开源驱动基本都已经迁移到了Atomic Mode Setting接口上了。

Atomic Mode Setting接口在用户态看来，是将原先各个KMS object的状态由隐式的通过API更新，变成了显式的对象属性。用户态程序可以通过通用的属性操作接口读写KMS object上的属性，更改不会立即生效，而是缓存起来。当应用程序更新完其所有想要更新的属性时，可以通过Commit操作告知要求KMS层真正的更新硬件的状态。此时驱动程序需要验证应用程序要求进行的修改是否合法，在合法的情况下，可以一次性完成整个显示状态的修改。Atomic Mode Setting也实现了只用于检查新状态是否合法的接口。

由于Atomic Mode Setting提供的接口的功能是强于原先的KMS接口的，因此原先的KMS接口可以被Atomic Mode Setting接口实现。KMS Core提供了一些helper函数用以帮助驱动程序作者实现原先的Legacy KMS接口[[1]][1]。

由于Legacy接口注定要扔到历史垃圾箱，后续的所有分析都是以`Atomic Mode Setting`的code path作为基准。

## 驱动接口

驱动实现KMS接口的方式如下：

* 在probe函数中调用`drm_mode_config_init`函数初始化KMS core
* 填充mode_config中int min_width, min_height; int max_width, max_height的值，这些值是framebuffer的大小限制
* 设置mode_config中的funcs

下面以virtio-gpu为例分析驱动的实现。对于virtio-gpu上面的操作步骤实现如下：

```c
        drm_mode_config_init(vgdev->ddev);
        vgdev->ddev->mode_config.quirk_addfb_prefer_host_byte_order = true;
        vgdev->ddev->mode_config.funcs = &virtio_gpu_mode_funcs;
        vgdev->ddev->mode_config.helper_private = &virtio_mode_config_helpers;

        /* modes will be validated against the framebuffer size */
        vgdev->ddev->mode_config.min_width = XRES_MIN;
        vgdev->ddev->mode_config.min_height = YRES_MIN;
        vgdev->ddev->mode_config.max_width = XRES_MAX;
        vgdev->ddev->mode_config.max_height = YRES_MAX;
```

funcs中填充的内容如下：

```c
static const struct drm_mode_config_funcs virtio_gpu_mode_funcs = {
        .fb_create = virtio_gpu_user_framebuffer_create,
        .atomic_check = drm_atomic_helper_check,
        .atomic_commit = drm_atomic_helper_commit,
};
```

## helper架构

helper架构是我起的名，知道是指什么东西就好。DRM子系统的API比较难抽象，简单来说就是硬件各有各的不同，很多情况下，驱动可以使用一个共同的实现，而在其它情况下，驱动需要提供自己的实现。因此，DRM驱动核心的接口使用了helper架构，其基本思想是通过一组回调函数抽象特定组件的操作，比如`drm_connector_funcs`，同时又使用另外一组helper函数给出了原先那组回调函数的通用实现，让开发最者实现这组helper函数抽象出的回调函数即可。

这样双层的实现即能保证开发者有足够高的自由度（完全不用helper函数），也能简化开发者的开发（使用helper函数），同时提供给开发者hook特定helper函数的能力。下面以`drm_connector`为例说明helper架构的实现与使用方式。

正常情况下，创建`drm_connector`对象时需要提供`struct drm_connector_funcs`回调函数组，而使用helper函数时，可以直接用helper函数填充对应回调函数：

```c
static const struct drm_connector_funcs vc4_hdmi_connector_funcs = {
        .detect = vc4_hdmi_connector_detect,
        .fill_modes = drm_helper_probe_single_connector_modes,
        .destroy = vc4_hdmi_connector_destroy,
        .reset = drm_atomic_helper_connector_reset,
        .atomic_duplicate_state = drm_atomic_helper_connector_duplicate_state,
        .atomic_destroy_state = drm_atomic_helper_connector_destroy_state,
};
```

事实上helper函数并不万能，只是抽象出了大多数驱动程序应该共享的行为，而特定于硬件的部分，则需要以回调函数的形式提供给helper函数，这个回调函数组由`struct drm_connector_helper_funcs`提供。在创建`drm_connector`时，需要通过`drm_connector_helper_add`函数注册。函数将对应的回调函数对象的地址保存在了`drm_connector`中的`helper_private`指针中，如下：

```c
static inline void drm_connector_helper_add(struct drm_connector *connector,
                                            const struct drm_connector_helper_funcs *funcs)
{
        connector->helper_private = funcs;
}
```

这一套实现位于`include/drm/drm_modeset_helper_vtables.h`中，其他的DRM对象都有类似的实现，可以详细阅读`drm_connector_helper_funcs`的注释，理解其中对应的回调函数的用途。在实现DRM驱动时，helper架构会频繁用到，合理掌握helper函数可以极大简化开发，提升驱动程序的兼容性。

## CRTC

## Framebuffer

[内核文档](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#frame-buffer-abstraction)

framebuffer应该是唯一一个与硬件无关的抽象了。驱动程序需要提供自己的framebuffer实现，其主要入口就是前面提到的`drm_mode_config_funcs->fb_create`回调函数。驱动程序通过扩展`drm_framebuffer`结构体可以向framebuffer中加入自己私有的字段。

```c
struct virtio_gpu_framebuffer {
        struct drm_framebuffer base;
        struct virtio_gpu_fence *fence;
};
```

创建framebuffer时，需要通过`drm_framebuffer_init`函数将framebuffer初始化，并导出到用户空间。`fb_create`函数接受一个`drm_mode_fb_cmd2`类型的参数：

```c
struct drm_mode_fb_cmd2 {
        __u32 fb_id;
        __u32 width;
        __u32 height;
        __u32 pixel_format; /* fourcc code from drm_fourcc.h */
        __u32 flags; /* see above flags */
        __u32 handles[4];
        __u32 pitches[4]; /* pitch for each plane */
        __u32 offsets[4]; /* offset of each plane */
        __u64 modifier[4]; /* ie, tiling, compress */
};
```

其中最重要的就是handle，handle是Buffer Object的指针，该Buffer Object就是被创建framebuffer的存储后端。

TODO framebuffer releated operation

## Plane

[内核文档](https://www.kernel.org/doc/html/latest/gpu/drm-kms.html#plane-abstraction)

plane由`drm_plane`表示，其本质是对显示控制器中scanout硬件的抽象。简单来说，给定一个plane，可以让其与一个framebuffer关联表示进行scanout的数据，同时控制控制scanout时进行的额外操作，比如colorspace的改变，旋转、拉伸等操作。`drm_plane`是与硬件强相关的，显示控制器支持的plane是固定的，其支持的功能也是由硬件决定的。

对于`drm_plane`的分析，我们从其结构体定义入手。首先可以看到，一个plane必须要与一个`drm_deivce`关联，且一个`drm_device`中支持的所有plane都被保存在一个链表中。`drm_plane`中存有一个mask，用以表示该`drm_plane`可以绑定的CRTC。同时`drm_plane`中也保存了一个`format_types`数组，表示该`plane`支持的framebuffer格式。

所有的`drm_plane`必为三种类型之一：

* `Primary` - 主plane，一般控制整个显示器的输出。CRTC必须要有一个这样的plane。
* `Curosr` - 表示鼠标光标，可选。
*  `Overlay` - 叠加plane，可以在主plane上叠加一层输出，可选。

来回顾一点历史：内核向用户态导出的接口实际上不包含`Primary Plane`，对应plane的接口只能操作`Cursor Plane`和`Overlay Plane`，后期提供了一个`Universial Plane`特性，使得用户态API可以直接操作`Primary Plane`。在明白这个历史遗留问题后，对`drm_plane`的实现就好理解了。

## Encoder

## Mode

一般人对mode的理解仅仅是分辨率，这种理解在DRM中是不够的，不足以理解`drm_display_mode`是干什么的。简单来说，mode是一组信号时序，用以驱动显示器正确显示一帧图像。首先能够猜到需要传什么东西给显示器：像素数据。而到底多少个像素就跟显示器的分辨率有关了，如1080p的显示器需要传递`1080 x 1920`个像素。更加具体的形式是一行一行的从左到右发送，由于硬件实现需要，需要额外的步骤对信号进行同步。帧与帧之间被称为vertical，即竖直的，而行与行之间被称为horizontal，即水平的，这直接对应于显示器的横竖方向。

```c
 *               Active                 Front           Sync           Back
 *              Region                 Porch                          Porch
 *     <-----------------------><----------------><-------------><-------------->
 *       //////////////////////|
 *      ////////////////////// |
 *     //////////////////////  |..................               ................
 *                                                _______________
 *     <----- [hv]display ----->
 *     <------------- [hv]sync_start ------------>
 *     <--------------------- [hv]sync_end --------------------->
 *     <-------------------------------- [hv]total ----------------------------->*
```

上面内核注释中的字符画完美的解释了`drm_display_mode`中变量的定义。需要注意的是现实状况中，还有需要其它复杂的显示模式，比如interlaced模式等，所以`drm_display_mode`区分逻辑参数与硬件参数，硬件参数就是真正进行硬件操作时使用的参数，而逻辑参数是为了方便驱动开发人员进行的抽象，`drm_display_mode`根据相应的flag计算出硬件参数。

除了上述直接与硬件相关的参数，`drm_display_mode`还携带了一些DRM相关的属性。比如类型：

```c
        /**
         * @type:
         *
         * A bitmask of flags, mostly about the source of a mode. Possible flags
         * are:
         *
         *  - DRM_MODE_TYPE_PREFERRED: Preferred mode, usually the native
         *    resolution of an LCD panel. There should only be one preferred
         *    mode per connector at any given time.
         *  - DRM_MODE_TYPE_DRIVER: Mode created by the driver, which is all of
         *    them really. Drivers must set this bit for all modes they create
         *    and expose to userspace.
         *  - DRM_MODE_TYPE_USERDEF: Mode defined via kernel command line
         */
		unsigned int type;
```

可以看到mode的两个来源：驱动创建和内核命令行自行定义。而`DRM_MODE_TYPE_PREFERRED`标记的`drm_display_mode`则一般为对应connector的native mode。除此之外一个比较重要的属性就是status：

```c
        /**
         * @status:
         *
         * Status of the mode, used to filter out modes not supported by the
         * hardware. See enum &drm_mode_status.
         */
        enum drm_mode_status status;
```

该属性直接标记该mode是否可以被硬件接受，如果不行，则会标注出具体原因。对应显示器的长宽一般会由`width_mm`和`height_mm`记录，单位是毫米。最后注意`drm_display_mode`一般与`drm_connector`关联，因此`drm_modes.c`中提供了相应的helper函数，比如：

```c
void drm_mode_probed_add(struct drm_connector *connector,
                         struct drm_display_mode *mode)
{
        WARN_ON(!mutex_is_locked(&connector->dev->mode_config.mutex));

        list_add_tail(&mode->head, &connector->probed_modes);
}
```

`drm_mode_probed_add`函数将该mode添加到一个connector的管理中。注意probed_modes列表中可能包含了许多硬件无法使用的mode，对于这样的一个列表，可以使用`drm_mode_prune_invalid`将其中非法的mode清除。

## Connector

首先明确connector抽象了什么东西。从内核文档的描述中可以明白，connector抽象的是一个**能够显示像素的设备**，从流媒体的角度来说，就是一个sink，是最终的图像输出的地方。或者更加具象的理解一下，字面意思就是显卡上面的接头，比如HDMI，DP等接头。connector由`struct drm_connector`进行表示，并定义在`include/drm/drm_connector.h`中，接下来就分析其相关实现。

首先从该结构体的定义下手，可以看到结构体定义开始比较长的，先从常规部分下手：

```c
struct drm_connector {
        /** @dev: parent DRM device */
        struct drm_device *dev;
        /** @kdev: kernel device for sysfs attributes */
        struct device *kdev;
        /** @attr: sysfs attributes */
        struct device_attribute *attr;
		.......
```

很明显，从这里看出，内核认为`struct drm_connector`是sysfs树形结构的一员，翻译一下，就是一个`struct drm_connector`对象会对应`/sys`目录下的某个子文件夹（节点）。有关该文件夹中相关的属性文件可以后续进行分析。

接下来可以看到明白一个`drm_device`中的所有connector都会被保存在一个链表中，进行管理，且`drm_connector`是一个`drm_mode_object`：

```c
        /**
         * @head:
         *
         * List of all connectors on a @dev, linked from
         * &drm_mode_config.connector_list. Protected by
         * &drm_mode_config.connector_list_lock, but please only use
         * &drm_connector_list_iter to walk this list.
         */
        struct list_head head;

        /** @base: base KMS object */
        struct drm_mode_object base;
```

从这里之后，与`drm_connector`相关的分析主要以逻辑功能进行划分，而不应采取线性分析的方式。每一个`drm_connector`都应该定义一个类型，并保存在`drm_connector`中：

```c
        /**
         * @connector_type:
         * one of the DRM_MODE_CONNECTOR_<foo> types from drm_mode.h
         */
        int connector_type;
        /** @connector_type_id: index into connector type enum */
        int connector_type_id;
```

内核支持的`drm_connector`类型是uapi的一部分，定义在`include/uapi/drm/drm_mode.h`中：

```c
#define DRM_MODE_CONNECTOR_Unknown      0
#define DRM_MODE_CONNECTOR_VGA          1
#define DRM_MODE_CONNECTOR_DVII         2
#define DRM_MODE_CONNECTOR_DVID         3
#define DRM_MODE_CONNECTOR_DVIA         4
#define DRM_MODE_CONNECTOR_Composite    5
#define DRM_MODE_CONNECTOR_SVIDEO       6
#define DRM_MODE_CONNECTOR_LVDS         7
#define DRM_MODE_CONNECTOR_Component    8
#define DRM_MODE_CONNECTOR_9PinDIN      9
#define DRM_MODE_CONNECTOR_DisplayPort  10
#define DRM_MODE_CONNECTOR_HDMIA        11
#define DRM_MODE_CONNECTOR_HDMIB        12
#define DRM_MODE_CONNECTOR_TV           13
#define DRM_MODE_CONNECTOR_eDP          14
#define DRM_MODE_CONNECTOR_VIRTUAL      15
#define DRM_MODE_CONNECTOR_DSI          16
#define DRM_MODE_CONNECTOR_DPI          17
#define DRM_MODE_CONNECTOR_WRITEBACK    18
```

很明显，connector驱动在初始化一个connector的时候应该设置connector的类型。与其他的drm对象类似，`drm_connector`的创建者需要提供一组回调函数，由于实现connector需要支持的一组操作：

```c
        /** @funcs: connector control functions */
        const struct drm_connector_funcs *funcs;
```

### drm_helper_probe_single_connector_modes

函数是一个helper，用于提供默认的`drm_connector_funcs->fill_modes`实现。本质上函数实现了对connector支持的`drm_display_mode`的扫描。从函数的注释中，可以看到函数进行的操作大致为：

1. 将connector中现有`modes`列表中的`drm_display_mode`全部标记为`MODE_STALE`状态
2. 从以下三个来源收集`drm_display_mode`，并使用`drm_mode_probed_add`函数添加到`probed_list`中：
   * &drm_connector_helper_funcs.get_modes回调函数
   * 如果`drm_connector`目前已经连接，则加入VESA标准DMT模式`1024 x 768`（这个就是VGA接口没插稳检测不到EDID时分辨率变`1024x768`的原因了吧）
   * 从内核命令行参数`video=`读取并生成`drm_display_mode`
3. 将probed_list中的`drm_display_mode`移动到`modes`列表中，并合并冲突项
4. 验证非STALE状态`drm_display_mode`的合法性
5. 将所有非法的`drm_display_mode`从`modes`列表中删除

### hotplug检测

`drm_connector`支持hotplug且DRM中提供了相应的helper，简化实现。目前主要的helper有：

* drm_kms_helper_poll_init()用于提供轮询检测支持
* drm_helper_hpd_irq_event()用于提供中断检测支持

下面就来分析DRM对于轮询检测的helper实现。可以看到，该helper的实现非常简单，其基本原理是创建一个delayed_work并使能：

```c
void drm_kms_helper_poll_init(struct drm_device *dev)
{
        INIT_DELAYED_WORK(&dev->mode_config.output_poll_work, output_poll_execute);
        dev->mode_config.poll_enabled = true;

        drm_kms_helper_poll_enable(dev);
}
```

而`drm_kms_helper_poll_enable`函数很明显就是用于重置并使能这个delayed_work。注意这个函数的调用参数为`drm_device`，也就是这个机制整个就是应用于一个`drm_device`的。在分析这个函数之前，可以发现一个模块参数`drm.poll`，用于控制轮询的行为：

```c
static bool drm_kms_helper_poll = true;
module_param_named(poll, drm_kms_helper_poll, bool, 0600);
```

`drm_kms_helper_poll_enable`函数首先检查是否能够开启轮询模式，条件如下：

```c
        if (!dev->mode_config.poll_enabled || !drm_kms_helper_poll)
                return;
```

也就是说，`drm.poll`模块参数可以直接影响轮询的行为。随后函数遍历所有的`drm_connector`，然后决定是否需要进行轮询：

```c
        drm_connector_list_iter_begin(dev, &conn_iter);
        drm_for_each_connector_iter(connector, &conn_iter) {
                if (connector->polled & (DRM_CONNECTOR_POLL_CONNECT |
                                         DRM_CONNECTOR_POLL_DISCONNECT))
                        poll = true;
        }
        drm_connector_list_iter_end(&conn_iter);
```

这里注意到`drm_connector.polled`字段，它表示一个`drm_connector`的轮询模式，是一个bitflag，有如下三位：

```c
         * DRM_CONNECTOR_POLL_HPD
         *     The connector generates hotplug events and doesn't need to be
         *     periodically polled. The CONNECT and DISCONNECT flags must not
         *     be set together with the HPD flag.
         *
         * DRM_CONNECTOR_POLL_CONNECT
         *     Periodically poll the connector for connection.
         *
         * DRM_CONNECTOR_POLL_DISCONNECT
         *     Periodically poll the connector for disconnection, without
         *     causing flickering even when the connector is in use. DACs should
         *     rarely do this without a lot of testing.
```

简单来说就是检测所有的`drm_connector`中是否有需要轮询检测状态的，如果有则开启轮询。函数最后根据检测的结果打开轮询：

```c
        if (poll)
                schedule_delayed_work(&dev->mode_config.output_poll_work, delay);
```

默认情况下，第一次进行轮询的delay为1秒，否则为10秒：

```c
#define DRM_OUTPUT_POLL_PERIOD (10*HZ)
```

前面看到delayed_work的回调函数为`output_poll_execute`，函数的实现还是比较简单的。函数遍历`drm_device`所有的`drm_connector`，然后找到需要进行轮询的设备，并调用`drm_helper_probe_detect`检测这个`drm_connector`的状态。而`drm_helper_probe_detect`仅仅是调用了`drm_connector_helper_funcs`中注册的`detect_ctx`和`detect`回调函数。

对于支持中断的`drm_connector`，如果它是粗粒度的，即无法判断哪一个`drm_connector`状态发生了改变，则驱动开发者可以在进程上下文调用`drm_helper_hpd_irq_event`函数，检测所有标记了`DRM_CONNECTOR_POLL_HPD`的`drm_connector`。反之，则开发这可以自行调用`drm_kms_helper_hotplug_event`函数处理该事件。`drm_kms_helper_hotplug_event`的主要行为是发送uevent到用户态，并调用`dev->mode_config.funcs->output_poll_changed`回调函数。

## 用户态调用路径

对于与`drmModeSetCrtc`相关的legacy接口，其最终都调用到了IOCTL上：

```c
        return DRM_IOCTL(fd, DRM_IOCTL_MODE_SETCRTC, &crtc);
```

而所有与drm相关的定义都在`drivers/gpu/drm/drm_ioctl.c`中：

```c
        DRM_IOCTL_DEF(DRM_IOCTL_MODE_SETCRTC, drm_mode_setcrtc, DRM_MASTER),
```

可以知道它的处理函数是`drm_mode_setcrtc`。函数首先检查DRM设备的feature：

```c
        if (!drm_core_check_feature(dev, DRIVER_MODESET))
                return -EOPNOTSUPP;
```

忽略到中间的处理可以看到：

```c
        if (drm_drv_uses_atomic_modeset(dev))
                ret = crtc->funcs->set_config(&set, &ctx);
        else
                ret = __drm_mode_set_config_internal(&set, &ctx);
```

对于支持A-KMS的驱动来说，我们最终调用的就是`drm_crtc_funcs->set_config`回调函数，也就是`drm_atomic_helper_set_config`函数。

```c
static inline bool drm_drv_uses_atomic_modeset(struct drm_device *dev)
{
        return drm_core_check_feature(dev, DRIVER_ATOMIC) ||
                (dev->mode_config.funcs && dev->mode_config.funcs->atomic
_commit != NULL);
}
```

用户态A-KMS调用的入口函数`drmModeAtomicCommit`内部使用了不同的IOCTL调用：

```c
        ret = DRM_IOCTL(fd, DRM_IOCTL_MODE_ATOMIC, &atomic);
```

对应到内核态：

```c
DRM_IOCTL_DEF(DRM_IOCTL_MODE_ATOMIC, drm_mode_atomic_ioctl, DRM_MASTER),
```

该函数就是A-KMS在内核对应的处理函数，主要进行如下的操作：

1. 检查DRM设备是否设置`DRIVER_ATOMIC`标志，没有设置报错退出
2. 检查用户态是否使能了A-KMS相关的API，没有使能报错退出
3. 处理用户态传入的flags如PAGE_FLIP_ASYNC，ATOMIC_TEST_ONLY，PAGE_FLIP_EVENT等
4. 申请一个新的atomic_mode_state，将用户态传入的property拷贝并设置到新的state上
5. 最后根据flags中是否允许阻塞调用`drm_atomic_commit`或者`drm_atomic_nonblocking_commit`函数

## State对象

state是什么？这里的state是DRM框架用来追踪显示pipeline各个组件状态的状态集合。一个DRM显示pipeline的整体状态由`struct drm_atomic_state`表示：

```c
struct drm_atomic_state {
  struct kref ref;
  struct drm_device *dev;
  bool allow_modeset : 1;
  bool legacy_cursor_update : 1;
  bool async_update : 1;
  bool duplicated : 1;
  struct __drm_planes_state *planes;
  struct __drm_crtcs_state *crtcs;
  int num_connector;
  struct __drm_connnectors_state *connectors;
  int num_private_objs;
  struct __drm_private_objs_state *private_objs;
  struct drm_modeset_acquire_ctx *acquire_ctx;
  struct drm_crtc_commit *fake_commit;
  struct work_struct commit_work;
};
```

可以看到他由每个独立的组件（即drm object）的状态对象组成。

### State的创建

`drm_atomic_state`的创建由`drm_atomic_state_alloc`实现。函数中可以看到，`drm_mode_config_funcs`中提供了名为`atomic_state_alloc`的hook，允许我们自己实现state对象的创建。在默认情况下，函数会调用简单分配内存，然后使用`drm_atomic_state_init`进行初始化。初始化函数仅仅是简单分配分配`drm_atomic_state`中几个指针指向的内存区域。

对于各个drm object对应的state，其创建操作由其对应的`drm_{object}_funcs->atomic_duplicate_state`实现，在驱动程序没有扩展`drm_atomic_state`的情况下，这个回调函数一般填写为`drm_atomic_helper_{object}_duplicate_state`。而在commit过程中，是由`drm_atomic_get_{object}_state`函数触发这个创建操作的。该函数触发复制state操作后，还会将复制后的state及原本的state填入`drm_atomic_state`中对应的`__drm_{object}_state`中。

```c
struct __drm_{object}_state {
        struct drm_{object} *ptr;
        struct drm_{object}_state *state, *old_state, *new_state;
    
        /* extra fields may exist */
};
```

这里的`old_state`保存`drm_{object}`现有的state，而`state`及`new_state`就保存我们复制后的state。

最后描述一下commit时创建state的简单流程：

* drm_mode_atomic_ioctl函数中会将用户态传入的property更新依次调用drm_atomic_set_property写入前面创建的`drm_atomic_state`
* drm_atomic_set_property函数会根据传入object的类型调用对应的`drm_atomic_get_{object}_state`函数，得到对应于该object类型的`drm_{object}_state`。在这个调用中，如果`drm_atomic_mode`中对应的`__drm_{object}_state`不存在，则复制原有的state并填入其中
* 随后`drm_atomic_set_property`会调用`drm_atomic_{object}_set_property`将属性更新写入到新的state当中
* 最后drm_mode_atomic_ioctl调用对应函数（`drm_atomic_commit`及其非阻塞版本）进行commit操作（该操作前提是没有设置TEST_ONLY的标志）

### state更新

state更新由`drm_atomic_{object}_set_property`函数实现。目前我们看到的state更新是作为一个整体出现的，即通过用户态的commit操作触发。事实上DRM还支持partial update，支持单独对某个object进行更新操作，后面会分析清楚。

### state的commit

上面看到真正的commit操作由`drm_atomic_commit`函数实现。该函数的实现也比较简单：

```c
int drm_atomic_commit(struct drm_atomic_state *state)
{
        struct drm_mode_config *config = &state->dev->mode_config;
        int ret;

        ret = drm_atomic_check_only(state);
        if (ret)
                return ret;

        DRM_DEBUG_ATOMIC("committing %p\n", state);

        return config->funcs->atomic_commit(state->dev, state, false);
}
```

主要分为检查state合法性和调用`drm_mode_config_funcs->atomic_commit`函数进行commit操作。默认情况下，atomic_commit回调函数的功能是由`drm_atomic_helper_commit`实现的。函数内部有两个code path：阻塞和非阻塞，我们主要讨论阻塞情况下的实现。在阻塞情况下，函数会直接调用`drm_mode_config_helpers->atomic_commit_tail`函数。A-KMS中实现了一个标准的helper：`drm_atomic_helper_commit_tail`。而该helper又由更多的helper组成，因此想要真正理解A-KMS中commit操作的大致流程，需要分析这些helper实现的功能及调用的约定。

```c
void drm_atomic_helper_commit_tail(struct drm_atomic_state *old_state)
{
        struct drm_device *dev = old_state->dev;

        drm_atomic_helper_commit_modeset_disables(dev, old_state);

        drm_atomic_helper_commit_planes(dev, old_state, 0);

        drm_atomic_helper_commit_modeset_enables(dev, old_state);

        drm_atomic_helper_fake_vblank(old_state);

        drm_atomic_helper_commit_hw_done(old_state);

        drm_atomic_helper_wait_for_vblanks(dev, old_state);

        drm_atomic_helper_cleanup_planes(dev, old_state);
}
```

### drm_atomic_helper_commit_modeset_disables

该helper的作用是关闭所有的

TODO: check_only



## Atomic Modeset Helper函数分析

### 架构

1. 去libdrm里找找看A-KMS的IOCTL接口与legacy到底有什么不同没有
2. 假设有不同，那么IOCTL就是有两套接口。对于legacy接口，走原先legacy那套，其对应callback由Atomic Modeset Helper函数实现。对于A-KMS接口，其对应接口也由对应Helper实现。也就是说，Helper是框架中的一部分。
3. 现在已经都是用新的A-KMS接口了，我认为legacy不用花大功夫去分析。

整体架构为：

* 原先legacy的callback保留，但是基本由A-KMS提供的公共helper实现
* 公共helper依赖与对应KMS object中保存的private_helper实现功能
* 驱动程序在注册KMS object时必须初始化legacy callback和private_helper，否则无法正常工作

以CRTC举例，如下：

```c
static const struct drm_crtc_funcs virtio_gpu_crtc_funcs = {
        .set_config             = drm_atomic_helper_set_config,
        .destroy                = drm_crtc_cleanup,

        .page_flip              = drm_atomic_helper_page_flip,
        .reset                  = drm_atomic_helper_crtc_reset,
        .atomic_duplicate_state = drm_atomic_helper_crtc_duplicate_state,
        .atomic_destroy_state   = drm_atomic_helper_crtc_destroy_state,
};

static const struct drm_crtc_helper_funcs virtio_gpu_crtc_helper_funcs = {
        .mode_set_nofb = virtio_gpu_crtc_mode_set_nofb,
        .atomic_check  = virtio_gpu_crtc_atomic_check,
        .atomic_flush  = virtio_gpu_crtc_atomic_flush,
        .atomic_enable = virtio_gpu_crtc_atomic_enable,
        .atomic_disable = virtio_gpu_crtc_atomic_disable,
};

static int vgdev_output_init(struct virtio_gpu_device *vgdev, int index)
{
        // ......
        drm_crtc_init_with_planes(dev, crtc, primary, cursor,
                                  &virtio_gpu_crtc_funcs, NULL);
        drm_crtc_helper_add(crtc, &virtio_gpu_crtc_helper_funcs);
        // ......
}

```



### drm_atomic_helper_check

从前面我们看到，A-KMS的主要操作主要分为两个：

* 检查显示mode的合法性，确认硬件确实在该mode下正常工作
* commit操作，将硬件完整的设置成对应的状态

而`drm_atomic_helper_check`就是一般情况下`drm_mode_config_funcs->atomic_check`内的回调函数。其主要包含两个大的功能点：

* drm_atomic_helper_check_modeset
* drm_atomic_helper_check_planes

前者逐级调用CRTC下面组件的`atomic_check`回调函数，确认modeset是否合法。



[1]: https://www.kernel.org/doc/html/latest/gpu/drm-kms-helpers.html#atomic-modeset-helper-functions-reference	"Atomic Modeset Helper Functions Reference"
[2]: https://lwn.net/Articles/653071/	"Atomic mode setting design overview, part 1"
[3]: https://lwn.net/Articles/653466/	"Atomic mode setting design overview, part 2"
