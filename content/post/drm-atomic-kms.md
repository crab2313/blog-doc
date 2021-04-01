+++
title = "DRM框架分析（四）：Atomic KMS"
date = 2021-03-31
tags = ["kernel", "drm"]

+++

首先Atomic Mode Setting（后续简称`A-KMS`）是DRM子系统最近的一次比较大的改动，其目的是填补当前API的不足。由于原先的API不支持同时更新整个DRM显示pipeline的状态，因此KMS过程中会出现一些中间状态，容易造成开发者不希望看见的结果，影响用户体验。同时，原先的KMS接口也不支持回滚，需要应用程序自己记录原先的配置状态，Atomic Mode Setting也解决了这个问题。

由于Atomic Mode Setting是新出现的API，为了解决用户态程序的兼容性问题，Atomic Mode Setting的接口被隐藏起来，只有用户态程序显式告知DRM层其支持Atomic Mode Setting时它的接口才会暴露出来。目前主流的开源驱动基本都已经迁移到了A-KMS接口上了。

Atomic Mode Setting接口在用户态看来，是将原先各个KMS object的状态由隐式的通过API更新，变成了显式的对象属性。用户态程序可以通过通用的属性操作接口读写KMS object上的属性，更改不会立即生效，而是缓存起来。当应用程序更新完其所有想要更新的属性时，可以通过Commit操作告知要求KMS层真正的更新硬件的状态。此时驱动程序需要验证应用程序要求进行的修改是否合法，在合法的情况下，可以一次性完成整个显示状态的修改。A-KMS也实现了只用于检查新状态是否合法的接口。

由于Legacy接口注定要扔到历史垃圾箱，后续的所有分析都是以A-KMS的code path作为基准。毕竟目前主流驱动都实现了基于A-KMS的接口，并利用DRM框架提供的helper实现了原有的legacy接口。

## 实现形式

由于Atomic Mode Setting提供的接口的功能是强于原先的KMS接口的，因此原先的KMS接口可以被Atomic Mode Setting接口实现。KMS框架提供了一整套[helper函数](https://www.kernel.org/doc/html/latest/gpu/drm-kms-helpers.html#atomic-modeset-helper-functions-reference)用以帮助驱动程序作者实现原先的Legacy KMS接口。

本质上，就是原先的legacy相关的接口都通过`A-KMS`兼容层实现的helper函数实现，具体到驱动实现细节，类似下面的写法：

```c
static const struct drm_crtc_funcs vkms_crtc_funcs = {
        .set_config             = drm_atomic_helper_set_config,
        .destroy                = drm_crtc_cleanup,
        .page_flip              = drm_atomic_helper_page_flip,
        .reset                  = drm_atomic_helper_crtc_reset,
        .atomic_duplicate_state = drm_atomic_helper_crtc_duplicate_state,
        .atomic_destroy_state   = drm_atomic_helper_crtc_destroy_state,
        .enable_vblank          = vkms_enable_vblank,
        .disable_vblank         = vkms_disable_vblank,
};
```

实质上就是使用带有`drm_atomic_helper`前缀的helper函数实现原有的legacy接口。为了分析`A-KMS`的实现，对于原有的legacy接口和A-KMS相关的分析都是必不可少的。也就是说，我们的分析可以分成两部分：Native实现和Legacy helper。从实用的角度考虑，可以直接分析Native实现，而Legacy helper则可以在时间充裕的情况下进行分析。

## 用户态接口

可以现在来看看用户态接口的相关情况，以libdrm为例：

```c
typedef struct _drmModeAtomicReq drmModeAtomicReq, *drmModeAtomicReqPtr;

extern drmModeAtomicReqPtr drmModeAtomicAlloc(void);
extern drmModeAtomicReqPtr drmModeAtomicDuplicate(drmModeAtomicReqPtr req);
extern int drmModeAtomicMerge(drmModeAtomicReqPtr base,
                              drmModeAtomicReqPtr augment);
extern void drmModeAtomicFree(drmModeAtomicReqPtr req);
extern int drmModeAtomicGetCursor(drmModeAtomicReqPtr req);
extern void drmModeAtomicSetCursor(drmModeAtomicReqPtr req, int cursor);
extern int drmModeAtomicAddProperty(drmModeAtomicReqPtr req,
                                    uint32_t object_id,
                                    uint32_t property_id,
                                    uint64_t value);
extern int drmModeAtomicCommit(int fd,
                               drmModeAtomicReqPtr req,
                               uint32_t flags,
                               void *user_data);
```

用户态接口的本质就是扩展原有的属性接口，允许用户描述一个状态集合，然后通过`drmModeAtomicCommit`函数进行commit操做。从`libdrm`的代码中可以看到，commit的操作最后实质上调用了`DRM_IOCTL_MODE_ATOMIC`的`ioctl`，这就是Native接口唯一的入口。从内核代码中可以看到，该`ioctl`的处理函数为`drm_mode_atomic_ioctl`。以`drm_mode_atomic_ioctl`为线索，可以发现许多相关的实现。

## 状态对象

A-KMS的核心是整个显示控制器的状态集合，由一个独立的状态对象表示。一个DRM显示pipeline的整体状态由`struct drm_atomic_state`表示：

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

可以看到他由每个独立组件（即`drm_mode_object`）的状态对象组成。

#### state的创建

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

#### state更新

state更新由`drm_atomic_{object}_set_property`函数实现，前面已经分析了整体流程。目前我们看到的state更新是作为一个整体出现的，即通过用户态的commit操作触发。事实上DRM还支持partial update，支持单独对某个object进行更新操作，后面会分析清楚。

简单来说，**`atomic_duplicate_state`和`atomic_state_alloc`等hook的存在目的是允许驱动程序开发者在原有的state中加入自己的状态，通常情况下用既有helper即可。**

## 状态检验

### drm_atomic_check_only

前面提到整个接口是`atomic`的，这要求实现这套接口的驱动程序能够检测一个特定的显示pipeline（mode）状态是否合法（即能被硬件接受且正确运行）。`drm_atomic_check_only`函数即为汇总驱动这项功能的的入口。DRM的用户态API提供了`DRM_MODE_ATOMIC_TEST_ONLY`标志位，其目的是允许用户态直接要求驱动检测配置的合法性而不commit配置。

函数主要操作如下：

1. 对所有的CRTC，Connector和Plane，分别调用`drm_atomic_{crtc,connector,plane}_check`对其进行基本的合法性检查，注意这个检查并不涉及驱动实现的回调，完全是DRM框架自己的检查。
2. 如果`mode_config->funcs->atomic_check`回调函数存在，则调用其进行检查。注意这个函数一般情况下为`drm_atomic_helper_check`，或者是驱动自行实现的该函数的wrapper。
3. 如果`state->allow_modeset`为false，即要求不进行modeset操作，则对所有的`CRTC`调用`drm_atomic_crtc_needs_modeset`函数进行检查。

### drm_atomic_helper_check

从前面我们看到，A-KMS的主要操作主要分为两个：

* 检查显示mode的合法性，确认硬件确实在该mode下正常工作
* commit操作，将硬件完整的设置成对应的状态

而`drm_atomic_helper_check`就是一般情况下`drm_mode_config_funcs->atomic_check`内的回调函数。其主要包含两个大的功能点：

* drm_atomic_helper_check_modeset
* drm_atomic_helper_check_planes

前者逐级调用CRTC下面组件的`atomic_check`回调函数，确认modeset是否合法。

## Commit操作

### drm_crtc_commit

commit操作从感念上来看是基于每一个CRTC的，因此每个commit操作由`drm_crtc_commit`进行抽象。整个结构体的定义非常简单，位于`include/drm/drm_atomic.h`中，注释非常详细：

```c
struct drm_crtc_commit {
    struct drm_crtc *crtc;
    struct kref ref;
    struct completion flip_done;
    struct completion hw_done;
    struct completion cleanup_done;
    struct list_head commit_entry;
    struct drm_pending_vblank_event *event;
    bool abort_completion;
};
```

从中可以知道所有的`drm_crtc_commit`会被放入`drm_crtc->commit_list`中，且**`drm_crtc_commit`实质上仅仅起到一个同步的作用**，分别对应三个事件：

* flip_down
* hw_done
* cleanup_done

后续的分析中点位于这三个信号的应用场景。

### drm_atomic_commit

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

主要分为检查state合法性和调用`drm_mode_config_funcs->atomic_commit`函数进行commit操作。默认情况下，atomic_commit回调函数的功能是由`drm_atomic_helper_commit`实现的。函数内部有两个code path：阻塞和非阻塞，我们主要讨论阻塞情况下的实现，因为上面看到，`drm_atomic_commit`调用的是非阻塞的实现。

### drm_atomic_helper_commit

大多数情况下，驱动程序会使用DRM框架中提供的默认实现。而DRM框架为`atomic_commit`回调函数提供个的默认实现为`drm_atomic_helper_commit`，接下来就对该函数进行分析。函数的实现很明显被`drm_atomic_state`参数中的`async_update`分成两段，如下：

```c
        if (state->async_update) {
                ret = drm_atomic_helper_prepare_planes(dev, state);
                if (ret)
                        return ret;

                drm_atomic_helper_async_commit(dev, state);
                drm_atomic_helper_cleanup_planes(dev, state);

                return 0;
        }
```

在非异步模式下，函数首先调用`drm_atomic_helper_setup_commit`。

### drm_atomic_helper_setup_commit

TODO: 基本看懂，但是我需要了解drm event机制，看完再回来写

在阻塞情况下，函数会直接调用`drm_mode_config_helpers->atomic_commit_tail`函数。A-KMS中实现了一个标准的helper：`drm_atomic_helper_commit_tail`。而该helper又由更多的helper组成，因此想要真正理解A-KMS中commit操作的大致流程，需要分析这些helper实现的功能及调用的约定。

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


