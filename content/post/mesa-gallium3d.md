+++
title = "MESA源码分析：Gallium3D"
date = 2022-04-28
draft = true


tags = ["mesa", "gallium3d", "drm"]

+++

前面可以看到DRI驱动的接口是比较灵活的，以至于大部分驱动程序在独立实现DRI驱动时很有可能出现对特定特性的重复实现。而Gallium3D则是由开发者提出的DRI驱动框架，其目的是简化DRI驱动程序的开发，使得代码得以复用。也就是说，gallium3D提供了一个框架，上接DRI驱动程序接口，下接内核DRI设备接口。

## 整体架构

从我的理解上来看，一个完整的gallium3d驱动程序，本质上分成三个部分：

* state tracker，也叫gallium frontend。本质上就是API前端，与上层接驳的地方。
* device driver。这个是与设备硬件相关的驱动层，从我的理解上是尽可能抽象出硬件的特性，使得所有的硬件可以提供统一的结构完成工作。
* winsys，这部分代码负责抽象操作系统层面的东西。

这里首先明确一点，gallium3d不仅仅是用来实现DRI驱动的，也可以实现别的驱动程序，比如vdpau或者vaapi，但是我们这里只讨论DRI驱动的实现。对于DRI驱动程序，frontend是确定的，就是DRI，位于`mesa/src/gallium/frontends/dri`文件夹下。而`device driver`与`winsys`则与硬件紧密相关。

最后提到一个target的概念，target实际上就是最终输出`.so`库的逻辑，位于`mesa/src/gallium/targets`文件夹下。比如vdpau驱动需要在`/usr/lib/vdpau`下生成`libvdpau_<driver-name>.so`等一系列库文件，则相应的文件夹下`meson.build`文件就会有对应逻辑。

## DRI驱动接口

一个非常简单的问题，如何基于gallium3D生成DRI驱动？Mesa的开发者采取了一个比较非常规的方式。所有基于gallium3D框架的DRI驱动会被编译在一起，形成一个动态库.so文件。通过文件硬链接的方式创建出多个_dri.so后缀的DRI驱动。

可以在`src/gallium/targets/dri/meson.build`中找到相应操作。且在同目录的target.c中定义了DRI驱动接口实现，dri.sym则定义了向外导出的符号。从前面的EGL分析可以看出，DRI驱动都提供一个`__driDriverGetExtensions_<driver_name>`接口，这里target.c中也是这么定义的：

```c
#define DEFINE_LOADER_DRM_ENTRYPOINT(drivername)                          \
const __DRIextension **__driDriverGetExtensions_##drivername(void);       \
PUBLIC const __DRIextension **__driDriverGetExtensions_##drivername(void) \
{                                                                         \
   globalDriverAPI = &galliumdrm_driver_api;                              \
   return galliumdrm_driver_extensions;                                   \
}
```

进一步查找可以看到：

```c
/* This is the table of extensions that the loader will dlsym() for. */
const __DRIextension *galliumdrm_driver_extensions[] = {
    &driCoreExtension.base,
    &driImageDriverExtension.base,
    &driDRI2Extension.base,
    &gallium_config_options.base,
    NULL
};
```

这是所有基于gallium3D框架的DRI驱动的入口。从这里我们并没有发现有和驱动名相关的东西，那gallium3d是如何正确配置打开对应的设备的呢？从这里推断gallium3d一定实现某种自动配置机制。

从上面看到，使用gallium3d编写的DRI驱动程序的主要入口`__driDriverGetExtensions_##drivername`在大部分情况下都会返回`galliumdrm_driver_extensions`，我们也主要分析这个扩展集合上的接口。可以看到，其实现了以下几种扩展：

* DRI_CORE
* DRI_IMAGE_DRIVER
* DRI_DRI2
* DRI_ConfigOptions
* DRI_DriverVTable

### driCreateNewScreen2

该函数是`createNewScreen2`的实现。在gallium3d的dri frontend中，使用`__DRIscreen`结构体表示一个screen。其中有一个比较中要的字段就是`driver`，这个driver字段是从传入的driver_extensions中的`DRI_DriverVTable`扩展中取得的，在我们分析的上下文中就是`galliumdrm_driver_api`。

```c
const struct __DriverAPIRec galliumdrm_driver_api = {
   .InitScreen = dri2_init_screen,
   .DestroyScreen = dri_destroy_screen,
   .CreateBuffer = dri2_create_buffer,
   .DestroyBuffer = dri_destroy_buffer,

   .AllocateBuffer = dri2_allocate_buffer,
   .ReleaseBuffer  = dri2_release_buffer,
};
```

忽略掉一些细节，函数调用`driver->InitScreen`，实际上就是`dri2_init_screen`。



## pipe_loader

先明确一下这个pipe指什么，个人认为pipe泛指整个gallium3d，即由frontend -> driver -> winsys组成的pipeline。而这个pipeloader整个就是指将各个pipeline组件装载并串起来的一个loader。pipe-loader的代码比较少，我们可以逐一分析。首先pipe-loader将所有的设备分成三种类型：

```c
enum pipe_loader_device_type {
   PIPE_LOADER_DEVICE_SOFTWARE,
   PIPE_LOADER_DEVICE_PCI,
   PIPE_LOADER_DEVICE_PLATFORM,
   NUM_PIPE_LOADER_DEVICE_TYPES
};
```

即纯软件设备，PCI设备与platform设备，通过对类型的区分，后续可以方便的获取一些信息。随后可以看到，pipe-loader对于设备的表示：

```c
struct pipe_loader_device {
   enum pipe_loader_device_type type;

   union {
      struct {
         int vendor_id;
         int chip_id;
      } pci;
   } u; /**< Discriminated by \a type */

   char *driver_name;
   const struct pipe_loader_ops *ops;

   driOptionCache option_cache;
   driOptionCache option_info;
};
```

其中，我们比较关心的就是这个ops。对于整个模块的分析，我们逐一列举所有相关的API：

* `pipe_loader_probe`，这个函数本质上是连续调用两个`probe`函数，即`pipe_loader_drm_probe`与`pipe_loader_sw_probe`并合并结果
* `pipe_loader_create_screen`，这个函数实际上是调用ops->create_screen函数
* `pipe_loader_drm_probe`，这个函数简单打开所有的DRM render设备并得到文件描述符，并依次调用`pipe_loader_drm_probe_fd_nodup`函数
* `pipe_loader_drm_probe_fd_nodup`，函数首先尝试检测DRM设备是否是PCI设备，反之则是platform设备。随后函数将ops设置成`pipe_loader_drm_ops`，并通过loader从DRM文件描述符获取对应驱动的名称，并得到gallium内部对应于这个驱动名称的驱动描述符。后续我们来解释一个ops和驱动描述符。

ops，其类型是：

```c
struct pipe_loader_ops {
   struct pipe_screen *(*create_screen)(struct pipe_loader_device *dev,
                                        const struct pipe_screen_config *config, bool sw_vk);

   const struct driOptionDescription *(*get_driconf)(struct pipe_loader_device *dev,
                                                     unsigned *count);

   void (*release)(struct pipe_loader_device **dev);
};
```

我们最关心它的是`create_screen`回调函数。而对于DRI，`pipe_loader_drm_ops`如下：

```c
static const struct pipe_loader_ops pipe_loader_drm_ops = {
   .create_screen = pipe_loader_drm_create_screen,
   .get_driconf = pipe_loader_drm_get_driconf,
   .release = pipe_loader_drm_release
};
```

对于drivers_descriptors，他是放在元素类型为`struct drm_driver_descriptor`的同名数组中的。在`target-helpers`模块中，有macro用于定义这个数组的元素，如下：

```c
#define DEFINE_DRM_DRIVER_DESCRIPTOR(descriptor_name, driver, _driconf, _driconf_count, func) \
const struct drm_driver_descriptor descriptor_name = {         \
   .driver_name = #driver,                                     \
   .driconf = _driconf,                                        \
   .driconf_count = _driconf_count,                            \
   .create_screen = func,                                      \
};

#define DRM_DRIVER_DESCRIPTOR(driver, driconf, driconf_count)                          \
   DEFINE_DRM_DRIVER_DESCRIPTOR(driver##_driver_descriptor, driver, driconf, driconf_count, pipe_##driver##_create_screen)
```

从这里我们可以看到，所有的device driver都要在这里定义形如`pipe_<driver>_create_screen`的函数，这个函数实质上是前面看到的`pipe_loader_create_screen`函数最终调用到的回调函数，因为前面看到的`pipe_loader_drm_create_screen`函数实际上是这样的：

```c
static struct pipe_screen *
pipe_loader_drm_create_screen(struct pipe_loader_device *dev,
                              const struct pipe_screen_config *config, bool sw_vk)
{
   struct pipe_loader_drm_device *ddev = pipe_loader_drm_device(dev);

   return ddev->dd->create_screen(ddev->fd, config);
}
```

其中dd为填充入`pipe_loader_drm_device`的driver_descriptor。



现在对gallium3d的一些主要抽象进行分析。首先要明白几个概念：

* screen。显式的对象，用于管理共享的资源：textures，surfaces，buffers。还需管理的一个比较重要的对象是context。
* context。接近于opengl rendering context的概念，基本上就是如rasterization vertex state，shaders，还有drawing commands之类的。

## pipe_screen

在gallium3d中，screen由`struct pipe_screen`进行抽象。从注释中可以看到，开发者对这个对象的描述为：Adaptor or GPU，context无关的驱动函数。也就是说，一个`struct pipe_screen`从概念上是对应一个GPU的，描述了所有GPU中与context无关的部分。而所有与context相关的部分则另行定义了`struct pipe_context`进行描述。从逻辑关系上来看，一个screen必定包含多个context，事实上我们也能够看到`pipe_screen`有创建context的相关接口。

仔细观察，可以发现`pipe_screen`是一个巨型函数指针集合，亦即`pipe_screen`实际上是一个接口，需要由驱动进行实现。接下来我们对接口进行简单的分析，进一步理解这个对象。`struct pipe_screen`中定义了大量的函数指针，可以简单的将这些函数分成如下几类：

* query接口，主要是设备厂商以及capabilities，可以通过类似`get_param`这样的接口对特性的属性进行请求。很明显这是驱动与上层state tracker进行通信用的。
* context管理函数。一个context是隶属于一个screen的，所以需要相关的context创建、销毁函数。
* resource管理函数。管理例如texture，buffer等资源的接口。
* winsys相关接口。与winsys层进行通信，比如frontbuffer，swap region之类的函数。

## pipe_resource





## pipe_context



### context创建





