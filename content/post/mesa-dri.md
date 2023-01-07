+++
title = "MESA源码分析：DRI驱动"
date = 2021-01-17
draft = true


tags = ["mesa", "dri"]
+++

DRI驱动是MESA后端的入口，从这里与硬件紧密相关，而MESA前端则没有那么绑定硬件，大致是共用的。从概念上将，`libGL.so`等MESA前端接口会通过MESA loader装载对应DRI驱动，通过DRI驱动实现相应的功能。从这里可以看出，DRI驱动对MESA前端提供的接口应该是比较统一的，有成型的API。这也是使用DRI驱动作为切入点的原因。

## DRI驱动形式

从前面看到，`libGL`（在MESA称作loader）会根据系统环境动态的加载DRI驱动，但是实际上我们对DRI驱动到底是个什么形式是没有概念的。用户态的DRI驱动位于系统`/usr/lib/dri`文件夹下，具有类似`<驱动名称>_dri.so`的名称。本质上是一个具有特定ABI的共享库文件。使用md5sum工具我们可以看到：

```
e50f4d8b96fd8e3976d283b64ab72efb  crocus_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  d3d12_dri.so
c99bcd49197c5e9990b5548e8c39d72b  i830_dri.so
c99bcd49197c5e9990b5548e8c39d72b  i915_dri.so
c99bcd49197c5e9990b5548e8c39d72b  i965_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  iris_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  kms_swrast_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  nouveau_dri.so
593c399171676d555e6613d3dfbdb530  nouveau_drv_video.so
c99bcd49197c5e9990b5548e8c39d72b  nouveau_vieux_dri.so
c99bcd49197c5e9990b5548e8c39d72b  r200_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  r300_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  r600_dri.so
593c399171676d555e6613d3dfbdb530  r600_drv_video.so
c99bcd49197c5e9990b5548e8c39d72b  radeon_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  radeonsi_dri.so
593c399171676d555e6613d3dfbdb530  radeonsi_drv_video.so
e50f4d8b96fd8e3976d283b64ab72efb  swrast_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  virtio_gpu_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  vmwgfx_dri.so
e50f4d8b96fd8e3976d283b64ab72efb  zink_dri.so
```

可以看到很多驱动的md5值实际上是一样的，这是因为这些驱动实际上都是同一个文件的硬链接。MESA中的DRI驱动大致分成两组：legacy驱动和gallium驱动，这两种驱动后续的文档会进行分析。使用nm工具，可以看到这几个so实际上仅仅定义了几个类似`__driDriverGetExtensions_<驱动名称>`的符号。也就是说，loader在装载驱动的时候，其主要入口就是这个函数，loader实际上可以根据so文件的名称计算出这个入口函数的名称。这样也就实现同一个so文件充当多个DRI用户态驱动的机制。

从这个入口函数的名称，我们看到了一个新的概念：扩展（Extension）。这个概念是DRI驱动向外提供接口的主要机制。

## DRI扩展

使用一种比较向下兼容的方式定义自己支持的API，称为extensions（扩展）。MESA的loader装载DRI驱动时，首先通过驱动名称打开对应的.so文件，然后调用`__driDriverGetExtensions_<driver_name>`接口，获取DRI驱动支持的扩展。该函数返回一个数组，每个数组元素是一个指向扩展的指针。DRI驱动使用一种称为扩展（Extension）的方式定义其接口，DRI驱动对外的接口实际上就是一堆DRI扩展的集合。先来看一下扩展是如何工作的。

```c
struct __DRIextensionRec {
    const char *name;
    int version;
};
```

每一个扩展都是基于`__DRIextensionRec`扩展而来，也就是要将这个结构体嵌到扩展开头。该结构体记录扩展的名称和版本，后面可以看到，一个扩展本身就是一组定义好的函数指针，而为了向下兼容，扩展不允许更改既有函数指针的顺序和定义，只允许在尾部新增加函数指针，并增加版本号。

列举一下常见的扩展：

* DRI_Core
* DRI_IMAGE_DRIVER
* DRI_DRI2
* DRI_ConfigOptions

扩展实际上是上层与DRI用户态驱动之间的接口，分析DRI用户态驱动，实际上就是分析相关的扩展实现。目前，所有的扩展都定义在`include/GL/internal/dri_interface.h`文件中。对于每个扩展，文件中都定义三个基本元素：

```c
#define __DRI_CORE "DRI_Core"
#define __DRI_CORE_VERSION 2

struct __DRIcoreExtensionRec {
    __DRIextension base;
    __DRIscreen *(*createNewScreen)(int screen, int fd,
				    unsigned int sarea_handle,
				    const __DRIextension **extensions,
				    const __DRIconfig ***driverConfigs,
				    void *loaderPrivate);
    /* ... */
};
```

即使用两个宏定义扩展名称与版本，并定义一个新的结构体，定义相关的函数指针原型。由于缺乏文档，下文零碎的穿插目前看到的相关实现。

## __DRIscreen

从原先的分析中可以看到，`createNewScreen2`这个函数事实上是与用户态DRI驱动交互时被调用的第一个函数。类似的函数指针实际上在多个扩展中存在（在分析EGL的时候看到的，EGL中有一个调用该函数的优先级）。从函数的名称上看，函数的用途相当明显，即创建`__DRIscreen`对象。这里就引出一个问题，screen是什么？很明显screen不是狭义上的屏幕。从开发人员遗留下来的少量文档可以发现，screen实际就是GPU或者显示设备的意思。从这里可以看到，screen实际指代整个图形硬件。这个操作实际上是必要的，这是因为实际上任何进程都可以使用dlopen打开这个DRI驱动，这个对象实际上就是client端的handle。注意这个对象的细节是没有暴露在外面的。

## DRI2 vs DRI3

熟悉xorg的人应该熟悉这两个概念，Xorg后期在原有DRI机制（即DRI2）上出现了DRI3。DRI2与DRI3的主要区别是：DRI3下，X11的client需要自行申请Buffer，而不是从X11那里申请。这个机制和wayland的比较像了，同时也是EGLImage使用的机制。因此，可以明显看到与DRI3相关的实现的扩展很多都是`EGL_IMAGE`打头的，比如`EGL_IMAGE`和`EGL_IMAGE_DRIVER`。

注意DRI2和DRI3都是X11的概念，实际上wayland协议的机制类似于DRI3。且，xwayland只支持DRI3而不支持DRI2，这几点要注意。

## DRI Common

gallium直接借用了DRI通用代码中的一部分扩展的实现，这部分实现位置在`/mesa/drivers/dri/dri_util.h`中。简而言之，这部分代码是DRI驱动实现的时的简单共用代码，直接通过扩展的形式向外提供：

```c
/** DRI2 interface */
const __DRIdri2Extension driDRI2Extension = {
    .base = { __DRI_DRI2, 4 },

    .createNewScreen            = dri2CreateNewScreen,
    .createNewDrawable          = driCreateNewDrawable,
    .createNewContext           = driCreateNewContext,
    .getAPIMask                 = driGetAPIMask,
    .createNewContextForAPI     = driCreateNewContextForAPI,
    .allocateBuffer             = dri2AllocateBuffer,
    .releaseBuffer              = dri2ReleaseBuffer,
    .createContextAttribs       = driCreateContextAttribs,
    .createNewScreen2           = driCreateNewScreen2,
};
```

