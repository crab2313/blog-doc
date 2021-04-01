+++
title = "DRM框架分析（二）：KMS"
date = 2021-03-26
tags = ["kernel", "drm"]

+++

KMS全称是`Kernel Mode Setting`，这里的mode是指显示控制器的mode，详见下面对`drm_mode`的分析。与`KMS`相对应的是`User Mode Setting`，早期Unix的Xorg几乎完整实现了一套图形栈，此时`Mode Setting`这项功能主要是由用户态的DDX（Device Depedent Driver）实现的。`UMS`由于存在各种各样的问题，已经被放弃，目前主流驱动已经在多年以前完成了`KMS`接口的迁移，并将`Mode Setting`相关的实现从用户态移动到了内核态。本文着重分析内核KMS相关功能的框架实现。

事实上，显示控制器的设计从最初（CRT显示器时代）到现在（LCD显示器时代）并没有根本性的变化。KMS将整个显示控制器的显示pipeline抽象成以下几个部分：

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

## 驱动入口

我们知道`drm_device`用于抽象一个完整的DRM设备，而其中与`Mode Setting`相关的部分则由`drm_mode_config`进行管理。为了让一个`drm_device`支持`KMS`相关的API，DRM框架要求驱动：

* 注册`drm_driver`时，`driver_features`标志位中需要存在`DRIVER_MODESET`
* 在probe函数中调用`drm_mode_config_init`函数初始化KMS框架，本质上是初始化`drm_device`中的`mode_config`结构体
* 填充mode_config中int min_width, min_height; int max_width, max_height的值，这些值是framebuffer的大小限制
* 设置mode_config->funcs指针，本质上是一组由驱动实现的回调函数，涵盖`KMS`中一些相当基本的操作
* 最后初始化`drm_device`中包含的`drm_connector`，`drm_crtc`等对象

我们知道注册一个支持`KMS`的DRM设备时，会在`/dev/drm/`下创建一个`card%d`文件，用户态可以通过打开该文件，并对文件描述符做相应的操作实现相应的功能。该文件描述符对应的文件操作回调函数（`filesystem_operations`）位于`drm_driver`中，并由驱动程序填充。典型如下：

```c
static const struct file_operations vkms_driver_fops = {
        .owner          = THIS_MODULE,
        .open           = drm_open,
        .mmap           = drm_gem_mmap,
        .unlocked_ioctl = drm_ioctl,
        .compat_ioctl   = drm_compat_ioctl,
        .poll           = drm_poll,
        .read           = drm_read,
        .llseek         = no_llseek,
        .release        = drm_release,
};
```

基本都为DRM框架预先提供好的helper函数，可以根据驱动需要灵活改变。

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

### 用户态调用路径

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

1. 检查DRM设备是否设置`DRIVER_ATOMIC`标志，没有设置则报错退出
2. 检查用户态是否使能了A-KMS相关的API，没有使能报错退出
3. 处理用户态传入的flags如PAGE_FLIP_ASYNC，ATOMIC_TEST_ONLY，PAGE_FLIP_EVENT等
4. 申请一个新的atomic_mode_state，将用户态传入的property拷贝并设置到新的state上
5. 最后根据flags中是否允许阻塞调用`drm_atomic_commit`或者`drm_atomic_nonblocking_commit`函数

## VBlank处理

前面分析`drm_mode`时大致理解了vsync等相关信号，从原理上讲vblank是指上一帧画面scanout完毕后到下一帧画面开始scanout这两个时间节点之间的“空档期（blank）”。一般情况下，可以在这个空档期进行一些常规的操作，比如控制硬件更换scanout的framebuffer，在比较老的不支持page flip的硬件上，会要求在vblank时间段内完成framebuffer的重新绘制。目前的显示控制器一般会实现vblank中断，即显示控制器进入vblank时，会触发相应的中断，而DRM框架则提供了相应的helper帮助驱动程序对vblank机制进行处理。

从`drm_vblank.c`开头的注释可以知道，DRM框架对于驱动支持vblank的最小要求就是通过`drm_vblank_init`函数初始化vblank处理代码，然后在`drm_crtc_funcs`中实现`enable_vblank`和`disable_vblank`回调函数。最后在vblank的中断处理函数中调用`drm_crtc_handle_vblank`函数完成vblank的处理。

从`drm_vblank_init`函数中可以看到，DRM框架以CRTC为单位处理vblank，且为每一个CRTC创建了一个对应的`drm_vblank_crtc`对象，保存在`drm_device->vblank`数组中。DRM框架实现了一个比较精巧的机制，可以根据系统中是否有vblank事件的用户动态地开启和关闭vblank中端。而前面的`enable_vblank`和`disable_vblank`回调函数即为硬件填充的中断使能开关。为了追踪系统中vblank事件的用户，DRM框架采用引用计数的方式追踪当前vblank事件的用户数量。如果一个用户要求接收vblank事件，则可以调用`drm_crtc_vblank_get`，没有需要后，则可调用`drm_crtc_vblank_put`函数，降低引用计数。DRM框架根据引用计数是否为0决定是否使能vblank中断。注意引用计数为0时，DRM框架不会立即关闭中断，而是设置一个定时器，默认为5秒，超时后关闭vblank中断，这个等待事件可以通过DRM模块的模块参数进行配置。

用户态可以通过`drmWaitVBlank`函数等待特定的vblank事件，这个操作通过一个wait_queue实现，进程将自己注册到`wait_queue`上等待唤醒。而`drm_crtc_handle_vblank`函数则会唤醒这个`wait_queue`。最后提一句，`drmWaitVBlank`函数可以通过特定flag要求发生vblank事件后直接返回特定的DRM event给用户态的drm文件描述符。该行为通过`drm_device->vblank_event_list`实现。

## 事件传递

DRM可以异步的向用户态发送事件，最为常见的是Page Flip完成事件和vblank事件，但是也可以是其他通用的事件。由于涉及到用户态，那么相关定义肯定位于`uapi`中，即`include/uapi/drm/drm.h`。

事件本身通过文件描述符的read操作传递，简单来说，用户态通过对drm文件描述符进行读取操作得到一个事件。所有的事件都有一个公共的header，定义如下：

```c
struct drm_event {
        __u32 type;
        __u32 length;
};
```

0-0x7fffffff的事件类型为通用的DRM事件，目前只看到三个定义，而超过0x80000000的事件则为设备特定的，这里不进行分析。

```c
#define DRM_EVENT_VBLANK 0x01
#define DRM_EVENT_FLIP_COMPLETE 0x02
#define DRM_EVENT_CRTC_SEQUENCE 0x03
```

后面以以下几点为线索进行分析：

* DRM文件描述符的read回调函数
* 事件排队与分发的流程
* 常见事件的产生及用途

### drm_read

DRM框架要求驱动将read回调函数填充为`drm_read`。与功能相关的所有的字段都位于DRM文件描述符的private_data，即`drm_file`对象中：

```c
struct drm_file {
		......
        wait_queue_head_t event_wait;
        struct list_head event_list;
        int event_space;
        struct mutex event_read_lock;
		......
};
```

简单来说，所有要被发送给用户态的`drm_event`会被相应的`drm_pending_event`引用，而所有的`drm_pending_event`则会保存在`event_list`链表中。一旦链表为空，可以通过排队等待`event_wait`直到`event_list`不为空。为了防止多个线程对DRM文件描述符的读取，需要使用`event_read_lock`进行互斥。`event_space`记录着“概念上的”`drm_event`缓冲区剩余大小，默认情况下该缓冲区初始为4KB大。

接着回到`drm_read`，函数仅仅是依次取下`event_list`的元素，并将其写入读取缓冲区中。如果`event_list`为空，则根据是否为非阻塞状态决定等待`event_wait`或者直接返回`-EAGAIN`。如果写入 缓冲区已经写满，则将已经取下`drm_pending_event`放回原处。
