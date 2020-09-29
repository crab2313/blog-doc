+++
title = "Mutter实现分析：Atomic Modesetting"
date = 2020-09-29

[taxonomies]
tags = ["gnome", "mutter", "drm"]
+++

首先注意一个名字上的区分，一开始看代码的人可能会对命名有疑惑。在Jonas的设计中，整个KMS相关代码分为两个上下文：main和impl。main上下文就是mutter运行的上下文，即GMainLoop所在线程，而impl上下文虽然目前也运行在GMainLoop所在线程，但是从设计上就是KMS操作运行的上下文，后期可能迁移到独立线程上。因此，所有命名为Impl的类都是注册在`Impl`上下文的。

## MetaKms

该对象为简单容器，是Jonas为了实现transactional modesetting对KMS进行的抽象。transactional KMS最终的目的有两个：

* 使mutter可以利用Atomic Modesetting API，更加充分高效的利用硬件特性，消除modesetting的中间状态
* 使KMS API的调用主体可以为独立线程，本质上是允许KMS API的异步调用，即调用时

首先明确其抽象的目标，即`transactional modesetting`，`transactional`这个词与数据库中的意义一致。即对于KMS设备进行的modesetting是原子性的，没有中间状态，要么成功要么失败，失败时不会进行任何状态更新。因此`MetaKms`使用`MetaKmsUpdate`抽象一个`transaction`。这套抽象目前使用普通的KMS API实现，但是只要接口移植完毕，那么即可直接调用Atomic KMS实现对应的操作。因此`MetaKms`作为容器也有选择后端实现的功能，但是当前的实现都为普通KMS。

创建`MetaKms`对象时，需要传入一个`MetaBackend`，`meta_kms_new`函数会自行创建一个`MetaKmsImpl`对象：

```c
  kms = g_object_new (META_TYPE_KMS, NULL);
  kms->backend = backend;
  kms->impl = META_KMS_IMPL (meta_kms_impl_simple_new (kms, error))
```

随后会从后端中取出`MetaUdev`对象，然后注册两个信号的处理函数，可以看出这是为了处理GPU热插拔事件：

```c
  kms->hotplug_handler_id =
    g_signal_connect (udev, "hotplug", G_CALLBACK (on_udev_hotplug), kms);
  kms->removed_handler_id =
    g_signal_connect (udev, "device-removed",
                      G_CALLBACK (on_udev_device_removed), kms);
```

## MetaKmsUpdate

这个结构体是transactional KMS API设计的核心。其核心思想是：

* 对显示控制器的操作（即modesetting）transaction化
* 每个`MetaKmsUpdate`即代表一个transaction，操作时将所有操作缓存到`MetaKmsUpdate`中
* 只有commit操作时，原先cache的操作才会真正应用到显示控制器中

因此，`MetaKms`会保存一个当前的`MetaKmsUpdate`，所有需要进行的KMS操作都需要缓存到其中，然后在合适的时机进行commit。

```c
struct _MetaKmsUpdate
{
  gboolean is_sealed;

  MetaPowerSave power_save;
  GList *mode_sets;
  GList *plane_assignments;
  GList *page_flips;
  GList *connector_properties;
  GList *crtc_gammas;
};
```

在明白上述原理后，这个对象的表示也就比较容易理解了。首先`is_sealed`表示这个对象是否已经被封装好，即是否能够继续向其中添加操作。随后紧跟的是多个List，分别可以保存特定的一类操作。

`MetaKms`提供了多个接口，用于创建、获取，并commit一个`MetaKmsUpdate`：

```c
MetaKmsUpdate * meta_kms_ensure_pending_update (MetaKms *kms);
MetaKmsUpdate * meta_kms_get_pending_update (MetaKms *kms);
MetaKmsFeedback * meta_kms_post_pending_update_sync (MetaKms *kms)
```

因此，`MetaKmsImpl`的实现的功能就显而易见了，即负责真正执行一个`MetaKmsUpdate`。

## MetaKmsImplDevice

为了实现`Atomic Modesetting`，首先需要抽象一个`KMS`支持的操作。`MetaKmsImplDevice`抽象出了这个接口：

```c
struct _MetaKmsImplDeviceClass
{
  GObjectClass parent_class;

  void (* setup_drm_event_context) (MetaKmsImplDevice *impl,
                                    drmEventContext   *drm_event_context);
  MetaKmsFeedback * (* process_update) (MetaKmsImplDevice *impl,
                                        MetaKmsUpdate     *update);
  void (* handle_page_flip_callback) (MetaKmsImplDevice   *impl,
                                      MetaKmsPageFlipData *page_flip_data);
  void (* discard_pending_page_flips) (MetaKmsImplDevice *impl);
};
```

并由`MetaKmsImplDeviceSimple`和`MetaKmsImplDeviceAtomic`实现。显然`handle_page_flip_callback`和`discard_pending_page_flips`与`process`被直接当作虚函数导出。另一个非常重要的操作是`dispatch`，实现如下：

```c
  drm_event_context = (drmEventContext) { 0 };
  klass->setup_drm_event_context (impl_device, &drm_event_context);

  while (TRUE)
    {
      if (drmHandleEvent (priv->fd, &drm_event_context) != 0)
        {
```

可以看出，首先使用`setup_drm_event_context`回调函数设置好`drmEventContext`，本质上就是`page flip`的处理函数，在随后的`drmHandleEvent`中，如果出现`PageFlip`事件，则会调用`MetaKmsImplDeviceXXX`中设置好的`Page flip`处理函数进行处理。

除此之外比较有意思的是这个类保存了所有的DRM设备属性，也就是`connector`、`CRTC`等KMS接口导出的对象：

```c
typedef struct _MetaKmsImplDevicePrivate
{
  MetaKmsDevice *device;
  MetaKmsImpl *impl;

  int fd;
  GSource *fd_source;
  char *path;

  GList *crtcs;
  GList *connectors;
  GList *planes;

  MetaKmsDeviceCaps caps;

  GList *fallback_modes;
} MetaKmsImplDevicePrivate;
```

可以在该类型的初始化函数中得到对应的初始化方式，且该类型的初始化函数中还隐藏了比较重要的信息：一个`GSource`，如下：

```c
  priv->fd_source =
    meta_kms_register_fd_in_impl (meta_kms_impl_get_kms (priv->impl), priv->fd,
                                  kms_event_dispatch_in_impl,
                                  impl_device);
```

本质上是在GMainContext中注册了一个`GSource`，以DRM的文件描述符的可读状态为事件源，以传入的`kms_event_dispatch_in_impl`函数作为`dispatch`回调函数。

```c
static gpointer
kms_event_dispatch_in_impl (MetaKmsImpl  *impl,
                            gpointer      user_data,
                            GError      **error)
{
  MetaKmsImplDevice *impl_device = user_data;
  gboolean ret;

  ret = meta_kms_impl_device_dispatch (impl_device, error);
  return GINT_TO_POINTER (ret);
}
```

而在该函数中调用了`meta_kms_impl_device_dispatch`函数，除此之外别无它地。上述即为`Page Flip`的处理机制，从这里看，`MetaKmsImplDevice`一旦创建，即会注册`Page Flip`的处理函数，并进行处理。

## MetaKmsImplDeviceSimple

~~这个是目前mutter唯一实现的`MetaKmsImpl`，~~实质上是用普通KMS（即非atomic）操作实现transacational KMS的接口。可以预见，后续mutter项目会使用Atomic KMS接口实现一个`MetaKmsImplAtomic`，实现事实意义上的atomic KMS支持。

这里分析`MetaKmsImplSimple`几个比较复杂的操作。首先明确入口为`meta_kms_impl_simple_process_update`函数，其内部连续处理缓存在`MetaKmsUpdate`中的KMS操作。简单的操作只是简单的wrap一个`drmMode*`函数，可以略过，直接分析最复杂的PageFlip处理。先看其缓存在`MetaKmsUpdate`中的形式：

```c
typedef struct _MetaKmsPageFlip
{
  MetaKmsCrtc *crtc;
  const MetaKmsPageFlipFeedback *feedback;
  gpointer user_data;
  MetaKmsCustomPageFlipFunc custom_page_flip_func;
  gpointer custom_page_flip_user_data;
} MetaKmsPageFlip;
```

先理解`drmModePageFlip`函数，该函数与`drmModeSetCrtc`函数非常相似，只不过其只有在VBlank事件来了时才生效，也就是告诉DRM当VBlank事件到来时，更新framebuffer。我们可以将函数的flags参数传入`DRM_MODE_PAGE_FLIP_EVENT`，此时每当PageFlip发生时，都会产生一个PageFlip事件。事件通过drm文件描述符变为可读告知用户态，且我们可以使用`drmHandleEvent`处理事件。注意`drmModePageFlip`的最后一个参数为`user_data`，一个用户态指针，每当PageFlip事件生成时，对应的指针就会跟着传入`drmHandleEvent`函数，所以我们可以通过该指针辨别PageFlip。

接下来看`process_page_flip`函数，该函数是PageFlip处理的核心。首先可以知道`MetaKmsPageFlip`中提供了一个让调用方自己写PageFlip处理函数的机制，即对应于`custom_page_flip_func`函数指针和其对应的数据。当他们存在时，就通过它们进行PageFlip的处理，反之则使用标准的处理流程，即调用`drmModePageFlip`函数：

```c
      ret = drmModePageFlip (fd,
                             meta_kms_crtc_get_id (crtc),
                             plane_assignment->fb_id,
                             DRM_MODE_PAGE_FLIP_EVENT,
                             meta_kms_page_flip_data_ref (page_flip_data));
```

`page_flip_data`究竟记录了什么，目前不需要知道，到时候在分析mutter frame调度器的时候进行分析，目前只看到：

```c
  page_flip_data = meta_kms_page_flip_data_new (impl,
                                                crtc,
                                                page_flip->feedback,
                                                page_flip->user_data);
```

注意`drmModePageFlip`函数在一个VBlank期间只能调用一次，在已经调用过一次的情况下再继续调用的话会返回`-EBUSY`。可以看到`process_page_flip`函数在该情况下实现了一个缓存机制，将返回`-EBUSY`的调用重新调度到下一次VBlank。这里只需要看到它是将多余的PageFlip计算出一个时间间隔，并缓存到了一张表内，其余细节在分析frame调度时再分析。

## MetaKmsDevice

~~没意思。~~

有意思了，原先没有看到PageFlip事件的处理在`MetaKmsDevice`里，单纯以为其是drm设备文件描述符的容器。漏掉了最重要的函数`meta_kms_device_dispatch_sync`。先分析通用实现，然后再看`MetaKmsImplDevice`。

可以看到函数首先调用`MetaKmsImpl`的`meta_kms_impl_dispatch_idle`函数，后续调用了`MetaKmsImplDevice`实现的`dispatch`函数，中间穿插了`meta_kms_flush_callback`的调用。

## MetaCursorRenderer

来看个比较独立也看起来比较简单的东西吧：鼠标光标绘制。首先看通用的抽象即`MetaCursorRender`。首先明确光标绘制的两种方式，硬件光标和软件光标。首先说软件光标，这个比较好理解，即直接在屏幕上进行光标的绘制，与绘制窗口无异。硬件光标则不同，目前的显示控制器一般都实现了cursor图层（plane），即屏幕的主图层和cursor图层是相互独立的，只有在scanout的时候才由硬件进行叠加操作。cursor图层一般支持绘制一块比较小的bitmap到屏幕上，而该bimtap可以在屏幕上快速移动。但是硬件实现的cursor图层是有局限性的，特定情况下并不能启用，所以mutter需要管理硬件cursor的启用。

事实上通过简单思考即抽象出Cursor管理需要向外提供的接口：

* 光标（相对屏幕）位置的设置和获取
* 硬件光标的启用与停用管理
* 光标位图设置
* 光标更新（绘制）

mutter使用`MetaCursorRender`抽象cursor的绘制器，而使用`MetaCursorSprite`描述光标图片（注意，这个可以是动画）。为了管理硬件cursor的启用条件，定义了`MetaHwCursorInhibitor`接口，实现了该接口的对象可以提供一个函数用于确定是否可以启用硬件cursor。

可以看到`MetaCursorRender`向外提供的接口如下：

* set_curosr/get_cursor，更换和获取`MetaCursorSprite（并绘制）
* set_position/get_position，更该和获取光标位置（并绘制）
* force_update，强制重新绘制光标
* {is，add，remove}_hw_cursor_inhibitor，向其注册删除`MetaHwCursorInhibitor`

先来看一下绘制光标的抽象接口：

```c
  if (cursor_sprite)
    meta_cursor_sprite_prepare_at (cursor_sprite,
                                   (int) priv->current_x,
                                   (int) priv->current_y);

  handled_by_backend =
    META_CURSOR_RENDERER_GET_CLASS (renderer)->update_cursor (renderer,
                                                              cursor_sprite);
  if (handled_by_backend != priv->handled_by_backend)
    {
      priv->handled_by_backend = handled_by_backend;
      should_redraw = TRUE;
    }
```

这是`meta_cursor_renderer_update_cursor`中的片段。其大致逻辑就是，如果子类实现的`update_cursor`函数没有处理光标绘制，则交由`MetaCursorRenderer`实现的软件绘制器进行光标绘制。

## MetaDrmBuffer

Buffer管理应该是transacational KMS最后一个还没有做的部分了。很明显一个`MetaDrmBuffer`对应一个DRM Buffer，且其只提供一个虚函数：`get_fb_id`。现在看到这个类有三个实现：

* DUMB，这个就是那个几乎所有DRM驱动都支持的DUMB Buffer，不支持GPU加速，不看
* GBM，这个应该是本地GPU通过libgbm分配出来的Buffer
* Import，这个是其他GPU通过prime接口导出的Buffer，这个也暂时不看

`MetaDrmBuffer`是一个抽象类，由于上面的`DUMB`和`Import`类型的Buffer都没有进一步分析的必要，所以这里主要分析`MetaDrmBufferGbm`。首先明确这个Buffer对应的是DRM中的Buffer概念，且是使用libgbm分配而出的Buffer，所以这一套API与EGL和DRM高度相关。`MetaDrmBufferGbm`提供了两个创建方法：

* new_lock_front。该方法对应EGL中的`lock front buffer`操作，本质是锁定一个gbm_surface中的一个后端Buffer然后将其返回供用户操作。一旦完成操作则需要将该Buffer返还gbm_surface并解除锁定。
* new_take。直接将一个`gbm_bo`包装成一个`MetaDrmBufferGbm`。

无论使用哪个方法进行创建，都需要使用`meta_gpu_kms_add_fb`将整个buffer注册到DRM中，并得到一个`framebuffer id`并保存起来。

