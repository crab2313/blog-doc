+++
title = "DRM框架分析（三）：显存管理"
date = 2021-03-28
tags = ["kernel", "drm"]
+++

## GEM

### GEM与TTM之间的关系

TTM是内核最初的DRM显存管理器，其设计思想是试图为所有的显卡驱动提供一个公共的API。TTM后面被认为是失败的，其API与实现复杂不可控，没有人愿意用他。后来intel吸取教训，设计了GEM，其设计较为灵活，只提供基本的实现，部分功能需要驱动程序通过驱动自定的接口进行扩展。GEM与TTM在特性上的主要区别为GEM不支持管理独立显存，只支持UMA，而TTM两种都支持。目前的DRM驱动程序中，基本都使用GEM作为前端为用户态提供接口，而涉及到管理独立显存的时候，则借助TTM作为后端实现管理功能。

### 框架初始化

驱动程序可以自己选择是否使用GEM框架，如果选择使用，则需要在注册`drm_driver`到DRM core中时，在`driver_features`里设置DRIVER_GEM标志。在驱动加载时，DRM框架会自动初始化GEM。

### handle与name

handle是gem_object的引用，其作用域被限制在drm文件描述符内，用于用户态程序引用内核态的drm_gem_object。当drm文件描述符被关闭时，所有的handle都会被关闭。而name则是drm_gem_object的名称，为一个32-bit整数。name的作用域是全局的，因此被直接保存在drm_gem_object内。默认情况下drm_gem_object的name都为0，表示其是未命名的。用户态可以通过FLINK为drm_gem_object命名，之后系统内的其他进程可以通过对应的name对该drm_gem_object进行访问。

handle的创建是创建GEM对象的一个步骤。目前GEM对象的创建一般是通过设备相关的API进行实现的，驱动程序可以通过`drm_gem_handle_create`函数从对应drm文件中创建一个handle返回给用于态。关于handle，gem core中还提供了`drm_gem_handle_delete`和`drm_gem_handle_lookup`函数。事实上，handle的管理是通过idr实现的。

### drm_gem_object

该对象是GEM内存管理的核心。GEM目前提供的功能是不完全的，部分空缺需要驱动自行填补，因此GEM框架要求驱动在`drm_gem_object`的基础上实现自己的GEM对象。这个操作实际上就是将`drm_gem_object`嵌入到驱动自己定义的`{driver}_gem_object`中。`drm_gem_object`定义在`include/drm/drm_gem.h`中，且有详细的注释，这里不再赘述，重点分析一些字段。

`filp`是一个指向`struct file`的指针，要理解它的作用就必须理解真正的内存是如何分配的。前面提到了GEM只支持UMA，即显卡使用RAM做为显存，这就又涉及到了两种情况。在PC中由于IOMMU的存在，显卡并不强制要求连续的物理内存，因此GEM可以使用SHMFS做为存储后端，此时filp指针就指向对应的文件描述符。而在嵌入式应用场景下，大部分设备没有IOMMU，因此必须要求连续的物理内存，此时filp为NULL，驱动通过CMA申请到连续的物理内存用作存储后端。注意，GEM框架并不负责管理存储后端，只提供了一些基本的helper，而存储的分配与释放完全由驱动程序控制。

TODO： 研究一下SHMFS

`vma_node`简单来说是保存了这个object的mmap偏移量。这里又需要提及`drm_gem_object`向用户态的映射问题。GEM目前基本上是通过两种方式实现MAP gem_object到用户态的：设备相关IOCTL和基于drm文件描述符的mmap。后者通过一个虚拟的偏移量确定mmap究竟在映射哪一个`drm_gem_object`。该机制的实现细节需要单独讨论。

`dma_buf`字段是一个`struct dma_buf`类型的字段。这里又设计drm-prime框架的细节了。简单来说，通过name来实现进程间共享`drm_gem_object`存在显而易见的安全性问题，毕竟name是全局的。攻击者可以通过特定的pattern推测或者枚举`drm_gem_object`的name，达到窃取数据的目的。后续GEM集成了dma_buf，可以通过dma_buf实现共享。

### 用户态映射

用户态映射实质上就是将`drm_gem_object`描述的存储空间映射到用户态进程的虚拟地址空间，让用户态进程可以随机读写。正如前面提到的，这里只分析GEM提供的方式，即基于drm文件描述符的mmap。我们知道mmap系统调用的原型如下：

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset); 
```

GEM框架可以为每一个`drm_gem_object`绑定一个特定虚拟offset，通过offset辨别需要映射到用户态的`drm_gem_object`。该机制需要drm文件描述符与`drm_gem_object`的共同协作，前面看到`drm_gem_object`中提供了vma_node用于保存该信息，驱动程序需要自行调用`drm_gem_create_mmap_offset`为一个`drm_gem_object`注册一个虚拟offset。虚拟offset需要通过设备相关的IOCTL传递给用户态。`drm_device`中存在vma_offset_manager用于统一管理mmap的虚拟offset，其后端基本上为drm_mm与红黑树缓存的组合。

## DUMB

(TODO)

## GBM

前面看到基于GEM的驱动对外是没有提供统一的内存管理接口的，至少Buffer Object创建销毁等操作是需要自行提供设备相关的即口进行实现的。用户态没有统一的接口对缓冲区进行管理，这导致某些特定用户态程序的开发的困难，如wayland compositor。MESA项目提供了libgbm，抽象并实现了一个通用的buffer管理API。这里记录对该API进行的探讨。

### Note

经过代码分析，gbm实际上来源于MESA内存OpenGL实现的`internal/drm_interface.h`，也就是Mesa OpenGL实现的一个私有兼容层。

### gbm_device

`gbm_device`是DRM设备的抽象，管理所有的分配出来的BO（Buffer Object）。自然而然：

* 一个`gbm_device`创建出来的BO的生命周期与该`gbm_device`绑定
* `gbm_device`与特定DRI设备绑定

`gbm_create_device`负责从一个DRM设备中创建`gbm_device`：

```c
struct gbm_device* gbm_create_device(int fd);
```

这里的fd自然是打开`/dev/dri/card*`时的文件描述符。

### gbm_bo

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
