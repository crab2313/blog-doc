+++
title = "MESA源码分析：EGL"
date = 2022-01-17


tags = ["mesa", "egl"]

+++

本文尝试分析EGL与MESA相关的实现，这也是我第一次尝试涉及MESA相关的代码。

本文的目的是记录我对我感兴趣的点的分析：

* MESA是如何实现EGL的
* EGL的GBM后端
* EGL的GBM后端是如何用于实现wayland compositor的

分析的一切基础是EGL标准，可以在[这里](https://www.khronos.org/registry/EGL/specs/eglspec.1.5.pdf)找到。文档本身很短，这是因为它只是定义了非常基础的一部分标准，剩下的需要由扩展进行实现。MESA对于EGL的实现是比较独立的，位于`/src/egl`文件夹下，它的实现基于EGL标准，采取driver的方式进行区分。一个driver可以实现多个platform，目前可以看到的三个driver分别为dri2，haiku与wgl，分别对应与linux的DRI平台，haiku平台与windows上的WGL平台。我们代码分析的重点是dri2。

## DRI2与Platform

MESA的设计上，driver同时只能有一个，对于linux平台肯定是dri2了，我们简单解读一下dispatch流程。

Displatch的起点肯定是EGL的API了，EGL的API在`src/egl/eglapi.c`文件中定义，并通过`EGLAPIENTRY`标记，很好区分。整个`libEGL`库也使用symbols文件限定了向外导出的符号。每一个driver都会定义自己的`struct _egl_driver`全局变量，在`eglInitialize`初始化时，会将该对象的指针写入`EGLDisplay`中。这个全局变量类型定义了驱动应该实现的相关函数，后续调用相关EGL API时，一般情况下，做完合法性检查，即会调用相应的函数指针。`struct _egl_driver`一般情况下是与EGL API一一对应的。

在dri2驱动内部，又根据platform定义了一组函数指针，类型为`struct dri2_egl_display_vtbl`。对于不同的platform，dri2会调用对应的函数指针，根据platform的不同进行第二次dispatch。承接我们对gbm的分析，我认为本文应该分析EGL的gbm平台。

## eglGetPlatformDisplay

对于EGL实现的分析自然要以EGL相关API作为入手点，我们从eglGetPlatformDisplay开始。该函数的目的是初始化EGL并根据特定的platform获取EGLDisplay。很明显该函数仅仅应该是一个入口，根据传入的platform的值来进行分发。前面聊到，我们关注的重点应该是GBM平台，如下：

```c
#ifdef HAVE_DRM_PLATFORM
   case EGL_PLATFORM_GBM_MESA:
      disp = _eglGetGbmDisplay((struct gbm_device*) native_display,
                              attrib_list);
      break;
#endif
```

可以看到`_eglGetGbmDisplay`函数仍然是一个简单入口，其目的是根据platform扩展的要求检查attrib_list的值合不合法。之后真正的工作由`_eglFindDisplay`完成，如下：

```c
   return _eglFindDisplay(_EGL_PLATFORM_DRM, native_display, attrib_list);
```

EGL 1.5中明确说道，相同参数的eglGetPlatformDisplay应该返回相同的EGLDisplay句柄，因此MESA的EGL实现应该将所有的EGLDisplay句柄保存起来，在eglGetPlatformDisplay被使用相同参数调用时返回同一个句柄。这个句柄列表被保存在一个`_eglGlobal.DisplayList`全局变量中。函数首先在该表中进行查找，没有找到时则会进行重新创建，并将该元素插入到链表中，仅此而已。函数最终即返回这个EGLDisplay句柄。

## eglGetDisplay

这个函数就比较有意思的，EGL中定义这个函数为eglGetPlatformDisplay的替代品。但是这个函数仅仅接收一个EGLNativeDisplayType参数，比较神奇。它是如何知道我们想要的platform类型的呢？

函数使用`_eglGetNativePlatform`获取这个display指针对应的platform，其他的与eglGetPlatformDisplay无差别。这个函数使用多个方法进行检测：

* 使用EGL_PLATFORM或者EGL_DISPLAY环境变量进行决定
* 将传入的display指针cast成对应的结构体，根据结构体中的特殊字段来检测。比如wayland的display一般会有一个字段保存wl_display_interface指针
* 如果上面的都没检测匹配，则将其当作编译时默认的平台`_EGL_NATIVE_PLATFORM`，一般是X11

## eglInitialize

前面看到，获取EGLDisplay时实际上并没有进行任何有意义的操作，仅仅是添加了一个链表元素而已。对于EGL真正的初始化由eglInitialize函数实现。函数进行如下操作：

* 根据`LIBGL_ALWAYS_SOFTWARE`环境变量是否设置，来决定这个EGLDisplay是否强制软件渲染
* 调用`_eglDriver`注册的Initialize回调函数对EGLDisplay进行初始化操作。在失败的情况下，强制使用软件渲染，然后再次尝试初始化
* 创建相关的字符串，计算EGL版本号

这里能够看出`_eglDriver`实际上是每个driver实现了一个的，但是这个变量是全局的，实际上就是一个平台只能使用一个driver，不存在同时支持多个driver的情况。我们集中分析dri2即可。从`/src/egl/drivers/dri2/egl_dri2.c`文件中可以看到，该回调函数实际上为`dri2_initialize`。

同样，`dri2_initialize`函数也仅仅是个dispatcher，对于GBM后端，最终会分发到`dri2_initialize_drm`函数中。这里能够看到，如果前面传入的native display为NULL，那么初始化时会尝试打开`/dev/dri/card0`设备然后创建gbm device。随后函数创建`_EGLDevice`并加入到`_eglGlobal`上的一个链表中。接下来可以看到进行了`dri2_load_driver`操作，然后就是填充了一些列函数指针。

对于load_driver操作，其工作逻辑也是比较简单的，本质上是一个动态库装载操作。我们知道MESA是由一系列DRI驱动组成的，可以看到在MESA安装目下有对应硬件的DRI驱动：

```
mesa /usr/lib/dri/crocus_dri.so
mesa /usr/lib/dri/i830_dri.so
mesa /usr/lib/dri/i915_dri.so
mesa /usr/lib/dri/i965_dri.so
mesa /usr/lib/dri/iris_dri.so
mesa /usr/lib/dri/kms_swrast_dri.so
mesa /usr/lib/dri/nouveau_dri.so
mesa /usr/lib/dri/nouveau_vieux_dri.so
mesa /usr/lib/dri/r200_dri.so
mesa /usr/lib/dri/r300_dri.so
mesa /usr/lib/dri/r600_dri.so
mesa /usr/lib/dri/radeon_dri.so
mesa /usr/lib/dri/radeonsi_dri.so
mesa /usr/lib/dri/swrast_dri.so
mesa /usr/lib/dri/virtio_gpu_dri.so
mesa /usr/lib/dri/vmwgfx_dri.so
mesa /usr/lib/dri/zink_dri.so
```

装载操作的本质就是根据gbm对应的驱动名称，装载对应的dri动态库文件，然后从定义好的接口中得到驱动特定的扩展。对于dri2，为：

```c
static const struct dri2_extension_match dri2_driver_extensions[] = {
   { __DRI_CORE, 1, offsetof(struct dri2_egl_display, core) },
   { __DRI_DRI2, 2, offsetof(struct dri2_egl_display, dri2) },
   { NULL, 0, 0 }
};
```

找到特定的extesnion后，将其保存在EGLDisplay指针中。

## eglCreatePlatformWindowSurface

这个函数是用于创建`EGLSurface`的，本质上是一个在屏幕上的区域。后续我们可以使用任何一个根据兼容的`EGLConfig`创建出来的`EGLContext`对其进行渲染。

## eglBindAPI && eglQueryAPI

这个函数简单设置当前的渲染API，由于它和后端与驱动的关联性不强，所以这个信息可以简单的保存在EGL的线程结构体中。也就是，调用这个API时，EGL查找当前线程的线程结构体，然后将这个信息写入。至于为什么是这个行为，因为EGL标准规定渲染上下文是每个线程独立的。

## EGLDisplay

在分析API的过程中，需要对一些基本的对象有一些理解。EGLDisplay类型对外实际上是一个`void *`，而对内（MESA中）则是`_EGLDisplay *`即`struct _egl_display *`。首先明确，MESA不会将所有传入的EGLDisplay指针解释成`EGLDisplay`，而是需要将其进行查找，确认传入的句柄是由MESA自己创建的，才会继续使用它。这一过程由`_eglLookupDisplay`函数实现。

```c
static inline _EGLDisplay *
_eglLookupDisplay(EGLDisplay dpy)
{
   _EGLDisplay *disp = (_EGLDisplay *) dpy;
   if (!_eglCheckDisplayHandle(dpy))
      disp = NULL;
   return disp;
}
```

前面看到所有由MESA创建的EGLDisplay都会保存在一个链表上，`_eglCheckDisplayHandle`实际上就是遍历这个链表，检查指针是否在这个链表中。

还有一点要注意的就是线程安全性，对一个EGLDisplay进行相应操作的时候，需要确保线程安全性。所以MESA设计了一个`_eglLockDisplay`函数，并在`struct _egl_display`结构体中加入了一个mutex，并要求对EGLDisplay进行修改的函数都需要通过`_eglLockDisplay`获取这个mutex。

## eglCreateContext

层层dispatch之后，这个函数本质上就是`dri2_create_context`的wrapper。这个函数的目的是用于创建一个EGL中非常重要的对象EGLContext，这个context保存整个图形API的state。从函数的实现上来看，`dri2_create_context`函数主要做如下工作：

* 调用`_eglInitContext`初始化刚刚申请的`dri2_egl_context`，该类型本质上就是一个扩展了的`_EGLContext`，添加了一个`__DRIcontext`类型的指针。初始化过程无非是根据传入的参数和attr_list填充相应的字段，这一阶段实际上平台无关，仅仅是创建填充。
* 进行一些合法性检查，目前可以忽略。
* 根据解析到的字段确定API类型，比如OpenGL，OpenGL ES1， OpenGL ES 2等。
* 根据如下优先度调用API创建context：
  * DRI_IMAGE_DRIVER扩展的createContextAttribs
  * DRI_DRI2扩展的createContextAttribs或者createNewContextForAPI（取决于扩展版本）
  * DRI_SWRast扩展的createContextAttribs或者createNewContextForAPI（取决于扩展版本）,这里是fallback成软件实现了

至此，Context就创建完成了，可以看到，有意义的工作还是委托DRI驱动实现的。

## EGLContext

首先明确这个context是什么。OpenGL和OpenGL ES的API设计实际上就是巨型的隐式状态机，这个状态的状态集合就称作context。也就是说OpenGL正常工作时，有一个状态机，根据状态机的状态确定执行结果。EGL一个很明确的用途就是管理这个context，我们可以根据需要创建对应的context，也可以同时使用多个context，很明显，OpenGL工作时，需要绑定一个context。context是绑定到线程的，所以保存在一个线程独立的状态中。

```c
struct _egl_thread_info
{
   EGLint LastError;
   _EGLContext *CurrentContext;
   EGLenum CurrentAPI;
   EGLLabelKHR Label;

   /**
    * The name of the EGL function that's being called at the moment. This is
    * used to report the function name to the EGL_KHR_debug callback.
    */
   const char *CurrentFuncName;
   EGLLabelKHR CurrentObjectLabel;
};
```

context并没有保存大量的信息：

```c
struct _egl_context
{
   /* A context is a display resource */
   _EGLResource Resource;

   /* The bound status of the context */
   _EGLThreadInfo *Binding;
   _EGLSurface *DrawSurface;
   _EGLSurface *ReadSurface;

   _EGLConfig *Config;

   EGLint ClientAPI; /**< EGL_OPENGL_ES_API, EGL_OPENGL_API, EGL_OPENVG_API */
   EGLint ClientMajorVersion;
   EGLint ClientMinorVersion;
   EGLint Flags;
   EGLint Profile;
   EGLint ResetNotificationStrategy;
   EGLint ContextPriority;
   EGLBoolean NoError;
   EGLint ReleaseBehavior;
};
```

