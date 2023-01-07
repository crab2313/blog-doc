+++
title = "MESA源码分析：GBM"
date = 2021-11-15


tags = ["mesa", "drm", "gbm"]

+++

这次我感觉我能看懂GBM了，本文对GBM的实现和用法进行分析。

我们先从原理上对GBM进行推导。GBM的本质：硬件无关的显存申请与管理。为什么需要这么一个库：DRM自带的DUMB缓冲区不能申请GPU显存，不能使用3D加速。因此，GBM的基本操作就是显存的申请与释放。但是这个管理必有一个上下文，很明显不可能是全局的，因为哪怕是作为用户的我们，也肯定希望申请出的GPU显存是与某个GPU绑定的。这个上下文由`struct gbm_device`进行抽象，且必定是与DRM设备绑定的。由此，我们开始我们第一个API的分析：`gbm_create_device`。

## gbm_create_device

```c
struct gbm_device *gbm_create_device(int fd);
```

从[这里](https://www.khronos.org/registry/EGL/extensions/MESA/EGL_MESA_platform_gbm.txt)可以看到一段示例代码，`gbm_create_device`的参数fd实际上就是打开一个DRM设备后得到的文件描述符。注意实际上文件描述符并不一定必须是DRM文件描述符，这是因为GBM是支持多种后端的，`gbm_create_device`函数可以在`/usr/lib/gbm/`文件夹下找到形如`<driver_name>_gbm.so`的后端，然后装载使用（这个主要是给NVIDIA使用的）。由于我们是分析MESA的代码，这里假定只有一个dri后端。

经过层层dispatch后，最终调用的是`dri_device_create`函数。这里我们先看一下`struct gbm_device`的实现：

```c
struct gbm_device {
   /* Hack to make a gbm_device detectable by its first element. */
   struct gbm_device *(*dummy)(int);
   struct gbm_device_v0 v0;
};
```

这里提一嘴，第一个dummy元素必定会被设置成`gbm_create_device`函数的地址，这么做的意义就是后面EGL GBM扩展时，使用eglGetDisplay并传入gbm_device指针时，函数可以直接通过第一个字段是否为gbm_create_device的地址来判断传入的是否是gbm_device，从而决定是否使用GBM后端。而`struct gbm_device_v0`则是一个常见的trick，来保证ABI不会改变。除此之外，`struct gbm_device_v0`本质上就保存了后端信息，文件描述符，以及一大堆函数指针，用于GBM库其他API的dispatch。

回到`dri_create_device`，函数除了设置上面提到的函数指针之外，唯一做的工作就是如下：

```c
   force_sw = env_var_as_boolean("GBM_ALWAYS_SOFTWARE", false);
   if (!force_sw) {
      ret = dri_screen_create(dri);
      if (ret)
         ret = dri_screen_create_sw(dri);
   } else {
      ret = dri_screen_create_sw(dri);
   }
```

很明显，dri后端需要对`struct gbm_device`进行扩展，得到`struct gbm_dri_device`。与EGL中实现一样，该结构体对DRI驱动装载后得到的各类扩展进行了存储。经过分析，可以对`dri_screen_create`进行的操作总结如下：

* 从DRM文件描述符中得到DRM驱动的名称
* 根据驱动名称装载对应的用户态DRI驱动
* 绑定loader，core等扩展，得到函数指针
* 调用DRI_DRI2上的`createNewScreen2`方法，得到`__DRIScreen`并保存

## gbm_bo_create

```c
GBM_EXPORT struct gbm_bo *
gbm_bo_create_with_modifiers2(struct gbm_device *gbm,
                              uint32_t width, uint32_t height,
                              uint32_t format,
                              const uint64_t *modifiers,
                              const unsigned int count,
                              uint32_t flags);
```

该函数有多个变体，如`gbm_bo_create_with_modifiers2`，本质上这几个变体最后都dispatch到`gbm_dri_bo_create`函数。先来逐步理清这个函数的意义及用法：

* BO是什么？Buffer Object的缩写，在内核驱动中，这样一个缓冲区被称作Buffer Object，这是因为围绕着这个缓冲区对象，有相当多的method。其次，Buffer Object的位置实际上并不固定，可以在GPU独立显存，内存中移动，所以需要这么一个object的描述符来引用它。`gbm_bo_create`本质上就是GBM的核心操作，请求显存的分配。
* 为什么这个函数这么复杂？本质上还是硬件本身就复杂。现代GPU对显存的管理相当的精细，有GPU端的MMU负责管理GPU独立显存，系统内存，同时也管理显存类型，如普通显存，tile显存等等。同时，每一段显存可以执行的操作也不同，比如用于渲染的显存，用于scanout的显存。除此之外，还存在显存共享的需求，虽然现在已经有DMA-BUF接口，但是一个新的问题就是不同GPU对于显存格式的定义是不同的，需要更加精细的描述显存的格式。这也就是modifier出现的意义。
* 所以函数的这几个参数本质上就是：format =>格式，modifier => 显存modifier，flags => 显存用途。

在明白用法之后，我们来看`gbm_dri_bo_create`的实现。首先函数开头可以看到一个实现上的细节，如果我们向GBM请求一个`GBM_BO_USE_WRITE`用途的BO，但是后端没有`DRI_IMAGE`扩展时，则GBM的DRI后端会自动fallback回DUMB Buffer：

```c
   if (usage & GBM_BO_USE_WRITE || dri->image == NULL)
      return create_dumb(gbm, width, height, format, usage);
```

后面进行的一些操作简单就是将一些GBM认识的flags与格式转换成DRI认识的格式，这基本是个一一映射的状态。随后调用`loader_dri_create_image`：

```c
   bo->image = loader_dri_create_image(dri->screen, dri->image, width, height,
                                       dri_format, dri_use, modifiers, count,
                                       bo);
```

这个函数的本质就是调用DRI_IMAGE扩展上的createImageWithModifiers族函数创建一个`__DRIimage`并保存在BO对象中。也就是说，对于DRI后端，BO对象实质上就是`__DRIimage`对象的wrapper，且GBM本身借助`DRI_IMAGE`扩展实现显存的通用分配，所以GBM就是一个壳子而已。最后函数通过`queryImage`方法获取该`__DRIimage`的stride和handle，并缓存起来。

总结一下，GBM的DRI后端对于BO的管理，本质上就是依靠于DRI驱动的`DRI_IMAGE`扩展。常见的操作比如BO创建，map，释放等都是直接调扩展中提供的方法实现的。而`gbm_dri_bo`本身，其实也就是`__DRIimage`的wrapper。

## gbm_surface_create

我第一次看GBM的时候，一直对`struct gbm_surface`没有太深刻的理解，现在算是能看懂了。要理解`struct gbm_surface`相关的API到底是干什么的，必须要理解compositor的工作原理。首先我们要明白为什么要直接申请显存，直接用opengl等API不行么？事实上我们直接使用GBM进行显存管理的场景就是实现compositor，或者offscreen渲染。这个使用场景想达成的目标非常简单，即创建一个opengl上下文，使得我们可以直接使用OpenGL往一个framebuffer中渲染，然后将这个framebuffer作为scanout buffer输出到屏幕上。

在wayland协议早期，这类操作是直接使用GBM加上EGL的surfaceless扩展完成的。但是缺点很明显，开发者必须手动申请BO然后导入EGLImage，然后将其提交给surfaceless扩展。后面，MESA开发者提出了一个新的扩展：`EGL_KHR_platform_gbm`，算是解决了这个问题。本质上，这个扩展允许你直接将DRM设备当作EGLDisplay，并将一个申请好的`gbm_surface`当作EGLSurface，使得我们可以直接使用eglSwapBuffer等API完成我们想要的工作。这里，`gbm_surface`本质上就是个stub，记录了这个EGLSurface的基本状态。

```c
struct gbm_dri_surface {
   struct gbm_surface base;

   void *dri_private;
};

struct gbm_surface_v0 {
   uint32_t width;
   uint32_t height;
   uint32_t format;
   uint32_t flags;
   struct {
      uint64_t *modifiers;
      unsigned count;
   };
};
```

其中，dri_private实质上保存的是MESA EGL实现中的dri2_egl_surface，也就是EGLSurface的backend。实际上，`gbm_surface`在创建后仅仅就是个stub，只有在`eglCreatePlatformWindowSurface`调用后，才会完成真正的初始化。
