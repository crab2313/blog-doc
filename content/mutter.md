+++
title = "Mutter窗口管理器实现分析"
date = 2020-09-27
draft = true

[taxonomies]
tags = ["gnome", "mutter", "drm"]

+++

本文是我对理解Mutter运行机制以及Linux的DRM子系统的一次尝试。之前的尝试似乎由于种种原因没有进行下去，而我最近深受GNOME下各种HIDPI问题的困扰，所以决定坚持下去，达到可以为GNOME开发特性的水准。在以前的尝试中，总结出比较重要的几点列举如下，做为之后的指导方向：

* 对于GPU的功能一定要准确的划分清楚。对于Mutter使用了GPU哪部分的功能也要有清晰的认识。Mutter本质上是利用了Linux内核抽象出来的DRM接口，控制Display Controller将图输出到屏幕上。而输出的内容则由各个窗口的内容合成出来，故而被称作Compositor，合成的过程则利用了GPU的图形pipeline，但也仅仅是非常简单的利用，并不是什么复杂的3D程序。因此分析Mutter代码时，一定要重点关注窗口合成的逻辑，而非具体GPU的实现。
* Mutter事实上实现了两个后端，一个被称为native，即DRM+wayland那一套，另一个为x11。X11是没有分析的必要的，后续也会淘汰，未来是Wayland的天下。

由于本文中夹杂着以前遗留下来的文档，故而逻辑不是很清晰，阅读时需要顺着自己的逻辑。以前的多次尝试都是直接从Mutter入手开始阅读，但在最后都由于卡在了自己没有涉猎的知识点上，导致流产，因此本次直接从Mutter的底层依赖库入手进行学习。因此，这次的顺序应该为：

* Clutter。Clutter是gtk3时代的产物，本质上是因为gtk3缺乏GPU加速能力（即无法利用GPU渲染）而被gnome开发者创造出来的一套替代方案，是与gtk3平行的toolkit。因此clutter被gnome开发者拿来写compositor，本质上是因为其具备基本的3D渲染能力。
* EGL。EGL是通用的OpenGL系渲染API的Context创建接口，Mutter利用其创建基本的渲染Context。
* libinput。与wayland同期开发出来的用户输入处理库。

## 问题

希望本文档结束时，我可以回答以下问题：

* MetaWindow是如何绘制的，如何重新绘制
* clutter的长宽坐标是如何传入的
* 整个mutter的显示架构是什么样的
* HIDPI是如何实现的

## 感兴趣的特性

* void GPU实现。本质上允许无mutter运行在无GPU模式下，这使得mutter可以动态卸载唯一的GPU，让内核卸载其驱动，将设备用以GPU穿透。实现单GPU不重启进行GPU穿透。
* 根据EDID自动决定默认缩放比例。
* Atomic Modesetting

## Clutter

看了一部分Clutter的文档，感觉对于Mutter来说，并不需要理解许多多余的概念。首先是ClutterActor，即一个2D元素，可以在3D空间中被变换，即移动，拉伸，旋转。而ClutterActor可以组成树型结构，即构成绘制树，child在parent之上进行绘制。ClutterStage是总的ClutterActor也是一个ClutterCroup，可以放多个ClutterActor。

## Mutter

从入口`core/mutter.c`看起。首先明确一点，mutter是一个支持插件的设计，gnome-shell实际上是mutter的一个插件。gnome3的桌面环境并没有mutter进程，只有gnome-shell进程。`core/mutter.c`实际上是一个独立的可执行程序，这里我直接从这里入手，可以先避开gnome-shell的代码。

可以看到这个文件很短，除掉命令行参数处理就只剩下几行：

```c
  if (plugin)
    meta_plugin_manager_load (plugin);

  meta_init ();
  meta_register_with_session ();
  return meta_run ();
```

其中plugin默认为libdefault，这里先不理会。所以入手点为`meta_init`函数，这个函数几乎是所有组件初始化的入口，目前只需要知道它初始化backend和事件循环。

来看`meta_run`，可以看到mutter是一个事件驱动的软件，其核心操作即为初始化一个事件循环，然后进行事件循环直至退出。

```c
  if (!meta_display_open ())
    meta_exit (META_EXIT_ERROR);

  g_main_loop_run (meta_main_loop);
```

至此，我们进入了最主要的入口`meta_display_open`函数。该函数目的是初始化一个`MetaDisplay`，该对象是mutter最核心的对象，表示整个mutter管理的显示，注意mutter支持多屏幕（Screen）。作为一个GObject，MetaDisplay的`_init`函数里是空的，也就是说我们只需要关注`meta_display_open`。

## MetaDisplay

`MetaDisplay`原先是对XDisplay的抽象，从`Wayland`支持后被赋予了更加抽象的意义。`MetaDisplay`表示整个`Mutter`管理的显示会话。我们从其创建函数`meta_display_opne`开始分析。

### meta_display_open

函数首先获取`MetaBackend`，该对象是单例对象，无论是哪里设置的，我们只考虑`Native`后端，而不考虑`X11`。函数首先通过`g_object_new`创建一个`MetaDisplay`然后对其进行初始化。`MetaDisplay`中记录的大量的配置，用以决定其行为，而默认的配置我们就不进行进一步分析了。

TODO

## MetaCompositor

这是一个抽象类，我们只关心其`MetaComositorWayland`子类。从名字上看来这个类是Mutter对其Compositor功能的抽象，虽然Wayland下并没有独立的Compositor组件。其中MetaCompositor管理的窗口实质上都是`MetaWindowActor`，这里的`Actor`实际上就是`ClutterActor`一个很直接的概念上的迁移。`MetaWindowActor`的结构如下：

```
 MetaWindowActor
  ↳ MetaSurfaceActor (surface)
     ↳ MetaShapedTexture
     ↳ MetaSurfaceActor (subsurface)
        ↳ MetaShapedTexture
        ↳ MetaSurfaceActor (sub-subsurface)
           ↳ MetaShapedTexture
     ↳ MetaSurfaceActor (subsurface)
        ↳ MetaShapedTextur
```

注意Wayland的`Sub Surface`特性，这里的`MetaSurfaceActor`抽象了这一特性。这里的`MetaShapedTexture`实质上是Surface的内容，Mutter中将Surface的内容与其形状区分了开来。所以`Compositor`的任务实质上就是管理这样的一个树形结构，并将其合成，并渲染成一张位图。

### meta_compositor_add_window

该函数如其名字所述，就是将一个`MetaWindow`加入到`Compositor`的管理中来。函数首先根据传入的`MetaWindow`的类型将其分类为X11或者Wayland，后续根据其类型创建`MetaWindowActorX11`或者`MetaWindowActorWayland`。

```c
  window_actor = g_object_new (window_actor_type,
                               "meta-window", window,
                               "show-on-set-parent", FALSE,
                               NULL);
```

作为一个`stack-based`的窗口管理器，Mutter中存在layer的概念，这很好理解，想想我们常见的置顶窗口功能，这就是将窗口设置到`top layer`的行为。Mutter中的layer定义如下：

```c
/**
 * MetaStackLayer:
 * @META_LAYER_DESKTOP: Desktop layer
 * @META_LAYER_BOTTOM: Bottom layer
 * @META_LAYER_NORMAL: Normal layer
 * @META_LAYER_TOP: Top layer
 * @META_LAYER_DOCK: Dock layer
 * @META_LAYER_OVERRIDE_REDIRECT: Override-redirect layer
 * @META_LAYER_LAST: Marks the end of the #MetaStackLayer enumeration
 *
 * Layers a window can be in.
 * These MUST be in the order of stacking.
 */
```

但是`Compositor`只会将`META_LAYER_OVERRIDE_REDIRECT`进行特殊对待，为其分配一个单独的`Group`。

```c
  if (window->layer == META_LAYER_OVERRIDE_REDIRECT)
    window_group = priv->top_window_group;
  else
    window_group = priv->window_group;
```

```c
  // struct _MetaCompositorPrivate
  ClutterActor *window_group;
  ClutterActor *top_window_group;
  ClutterActor *feedback_group;
```

注意这里的`Group`是`Clutter`中的`ClutterGroup`概念，本质上就是可以放`ClutterActor`的容器，最常见的就是`ClutterStage`。随后所有创建的`MetaWindowActor`会被放到`Compositor`内维护的`windows`列表中，最后调用`sync_actor_stacking`。`sync_actor_stacking`函数本质上是维护真个`Compositor`的stack窗口管理的性质，后续在分析`meta_compositor_sync_stack`函数时仔细分析。

### meta_compositor_sync_stack

现在来看`sync_actor_stacking`函数。本质上，`windows`是一个链表，保存着所有的`MetaWindowsActor`，也就是所有的窗口。我们知道`Stack-based`的窗口管理器有着深度这一概念，也就是窗口在Stack中的位置，而`windows`链表本质上就是反映了这个位置信息，`windows`的第一个元素就是Stack中最底下的窗口。而事实上合成整个窗口要利用`Clutter`实现，因此每个窗口对应一个`ClutterActor`，同时背景也要一同绘制，因此`MetaBackGroundActor`也算一个`ClutterActor`。这些`ClutterActor`都放置在`window_group`这个`ClutterGroup`中。`sync_actor_stacking`函数的功能实质上就是同步`windows`链表中窗口的深浅顺序与`window_group`中`ClutterActor`的深浅顺序。函数首先检测`windows`和`window_group`的顺序是否同步，一旦不同步，则调用`clutter_actor_set_child_below_sibling`一个一个地重建整个`window_group`中`ClutterActor`的顺序。

### meta_compositor_{manage,unmange}

`manage`函数的目的是让对应的Display被`Compositor`接管，内部调用了`meta_compositor_do_manage`函数。函数首先连接了`ClutterStage`上的`presented`信号：

```c
  priv->stage_presented_id =
    g_signal_connect (stage, "presented",
                      G_CALLBACK (on_presented),
                      compositor);
```

随后创建三个`MetaWindowGroup`，并将其加入到`ClutterStage`中：

```c
  priv->window_group = meta_window_group_new (display);
  priv->top_window_group = meta_window_group_new (display);
  priv->feedback_group = meta_window_group_new (display);
  clutter_actor_add_child (stage, priv->window_group);
  clutter_actor_add_child (stage, priv->top_window_group);
  clutter_actor_add_child (stage, priv->feedback_group);
```

最后调用子类实现的`manage`虚函数。`MetaCompositoative`没有实现这个虚函数。

### meta_compositor_queue_frame_draw

函数的直接参数是一个`MetaWindow`，但是前面提到了一个`MetaWindow`是直接与其对应的`MetaWindowActor`绑定的。因此函数获取对应的`MetaWindowActor`后直接调用`meta_window_actor_queue_frame_draw`。对应于`MetaWindowActorWayland`那就是什么都不做，也就是说这个函数在`Wayland`窗口上是空的。

## MetaStage

`MetaStage`实质上是`ClutterStage`的子类。本质上代表了多组需要显示在屏幕上的Actor。`meta_state_new`负责创建一个`MetaStage`，创建时，监听`MetaMonitorManager`上的`power-save-mode-changed`信号。除此之外，实现了`ClutterState`类提供的多个虚函数：

```c
  actor_class->paint = meta_stage_paint;

  stage_class->activate = meta_stage_activate;
  stage_class->deactivate = meta_stage_deactivate;
  stage_class->before_paint = meta_stage_before_paint;
  stage_class->paint_view = meta_stage_paint_view
```

### meta_stage_paint

`MetaStage`作为`ClutterActor`的子类，其`paint`虚函数是被重写过的，即`meta_stage_paint`函数。这个函数实质上是给`MetaStage`的绘制操作后面添加了信号处理与鼠标绘制。可以看到每当`paint`回调函数调用完毕之后，就会触发`MetaStage`上的`ACTORS_PAINTED`信号，然后检查`paint_contexts`中的标志位，如果发现没有`NO_CURSOR`则会重新绘制所有相关联的`MetaOverlay`。

```c
  if (!(clutter_paint_context_get_paint_flags (paint_context) &
        CLUTTER_PAINT_FLAG_NO_CURSORS))
    g_list_foreach (stage->overlays, (GFunc) meta_overlay_paint, paint_context);
```

注意这里默认所有`MetaOverlay`都表示鼠标了。

## MetaOverlay

`MetaOverlay`从概念上讲应该是一个`Overlay`区域，这个名词一般是指一个plane上的叠加层。`MetaOverlay`与一个`MetaStage`相关联，表示这个`MetaStage`显示区域上的一个叠加层，其定义如下：

```c
struct _MetaOverlay
{
  MetaStage *stage;

  gboolean is_visible;

  CoglPipeline *pipeline;
  CoglTexture *texture;

  graphene_rect_t current_rect;
  graphene_rect_t previous_rect;
  gboolean previous_is_valid;
};
```

各个字段的意思都比较直观，用户可以通过`meta_overlay_set`设置填充这个Overlay使用的材质，并设置其显示位置。`meta_overlay_paint`函数用于绘制`Overlay`到`MetaStage`上来。

### 光标Overlay绘制

熟悉DRM架构的人一定知道DRM提供了三种plane，其中一种就是`Cursor Plane`，用于支持硬件光标。这个功能本质上是实现了一个额外的图层绘制光标。`Mutter`中将其抽象为一个Overlay，可以看到几个用于操作这个Overlay的接口：

```c
MetaOverlay      *meta_stage_create_cursor_overlay   (MetaStage   *stage);
void              meta_stage_remove_cursor_overlay   (MetaStage   *stage,
						      MetaOverlay *overlay);

void              meta_stage_update_cursor_overlay   (MetaStage       *stage,
                                                      MetaOverlay     *overlay,
                                                      CoglTexture     *texture,
                                                      graphene_rect_t *rect)
```



## MetaWindowActor

`MetaWindowActor`是`ClutterActor`的子类，其基本目的是将一个`MetaWindow`与`ClutterActor`绑定起来，如前面所述。这里只分析其对应`Wayland`的子类`MetaWindowActorWayland`。

## MetaSurfaceActor

`MetaSurfaceActor`也是`ClutterActor`的子类，其目的是表示一个`surface`，类似于`Wayland`的`subsurface`的概念，是一个嵌套的树形结构，允许在`Surface`上添加子`Surface`。这是由于`Surface`可能有着不同的像素格式，比如父`Surface`是RGB格式的，而子`Surface`是`YUV`格式的，这个问题在硬件解码时比较常见。有了`Sub Surface`特性，就不需要应用程序自己对像素格式进行转换，也少了一层拷贝，只需要将硬件解码出来的位图作为一个`Sub Surface`的内容即可将其显示在窗口上。

该类的`Private`数据定义如下：

```c
typedef struct _MetaSurfaceActorPrivate
{
  MetaShapedTexture *texture;

  cairo_region_t *input_region;

  /* MetaCullable regions, see that documentation for more details */
  cairo_region_t *unobscured_region;

  /* Freeze/thaw accounting */
  cairo_region_t *pending_damage;
  guint frozen : 1;
} MetaSurfaceActorPrivate;
```

### meta_surface_actor_update_area

TODO

### MetaShapedTexture

`MetaShapedTexture`实现了`ClutterContent`接口，其目的是为一个`ClutterActor`绘制内容。`ClutterContent`接口提供的回调函数如下：

```c
struct ClutterContentIface {
  gboolean      (* get_preferred_size)  (ClutterContent   *content,
                                         gfloat           *width,
                                         gfloat           *height);
  void          (* paint_content)       (ClutterContent   *content,
                                         ClutterActor     *actor,
                                         ClutterPaintNode *node);

  void          (* attached)            (ClutterContent   *content,
                                         ClutterActor     *actor);
  void          (* detached)            (ClutterContent   *content,
                                         ClutterActor     *actor);

  void          (* invalidate)          (ClutterContent   *content);
};
```

具体到`MetaShapedTexture`，则是将一个`CoglTexture`表示的材质绘制到`ClutterActor`上。

### meta_shaped_texture_update_area



## MetaMonitor

这个类型很明显是对显示器的抽象。事实上，Mutter中的`MetaMonitor`分为两个子类：`MetaMonitorTiled`和`MetaMonitorNormal`，这里只分析`MetaMonitorNormal`。Mutter中定义了一系列描述显示器相关信息与设置的类型：

```c
typedef struct _MetaMonitorModeSpec
{
  int width;
  int height;
  float refresh_rate;
  MetaCrtcModeFlag flags;
} MetaMonitorModeSpec;

typedef struct _MetaMonitorCrtcMode
{
  MetaOutput *output;
  MetaCrtcMode *crtc_mode;
} MetaMonitorCrtcMode;
```

`MetaMonitor`从设计上就需要其子类实现一系列虚函数：

```c
static void
meta_monitor_normal_class_init (MetaMonitorNormalClass *klass)
{
  MetaMonitorClass *monitor_class = META_MONITOR_CLASS (klass);

  monitor_class->get_main_output = meta_monitor_normal_get_main_output;
  monitor_class->derive_layout = meta_monitor_normal_derive_layout;
  monitor_class->calculate_crtc_pos = meta_monitor_normal_calculate_crtc_pos;
  monitor_class->get_suggested_position = meta_monitor_normal_get_suggested_position;
}
```

从代码中来看，`MetaMonitor`记录了整个显示器的状态。包括：

* 这个显示器属于哪一个GPU
* 这个显示的包含的output，这里的`MetaOutput`是抽象DRM输出口的概念，这样可以实现复制屏
* 显示器的mode，优先的mode，当前的mode
* 显示器参数（复制于第一个output）
* 窗口系统ID

## MetaMonitorManager

这个类本质上就是你在`gnome-control-center`里配置显示器选项时的直接接口。事实上，`gnome-control-center`是通过`dbus`接口与系统组件进行通信的，为了让系统中的进程可以配置显示器属性，`MetaMonitorManager`中实现了`org.gnome.Mutter.DisplayConfig`接口。

由于存在外部接口，对这个类型的分析就从dbus接口开始。在创建对象之时，会创建dbus接口：

```c
  manager->dbus_name_id = g_bus_own_name (G_BUS_TYPE_SESSION,
                                          "org.gnome.Mutter.DisplayConfig",
                                          G_BUS_NAME_OWNER_FLAGS_ALLOW_REPLACEMENT |
                                          (meta_get_replace_current_wm () ?
                                           G_BUS_NAME_OWNER_FLAGS_REPLACE : 0),
                                          on_bus_acquired,
                                          on_name_acquired,
                                          on_name_lost,
                                          g_object_ref (manager),
                                          g_object_unref);
```

接口相关代码是由`gdbus-codegen`生成的，我们只需要知道其基本模型是`gdbus-codegen`帮助生成一个GObject，然后我们监听其上特定的信号，即可知道其他人调用了特定的method。作为典型案例，分析`handle-apply-monitors-config`信号的处理，即`ApplyMonitorsConfig`方法的处理函数。

### ApplyMonitorsConfig

函数首先检查`serial`的值是否与当前`MetaMonitorManager`保存的`serial`是否一致，如果不一致，则认为调用方设置的参数不合法。这个操作是因为显示器的属性是动态改变的，如果配置发生变化，则`MetaMonitorManager`会自增自身保存的`serial`，与此同时通过`GetResources`获取显示器属性时，也会拿到一个`serial`。通过`serial`的比对，可以确定调用方手中保存的属性是否合法。忽略掉中间复杂的参数处理，函数最后生成一个`MetaMonitorsConfig`对象，并调用`meta_monitor_manager_apply_monitors_config`，最后调用到子类实现的`apply_monitors_config`虚函数。由于这里分析的是Native后端，则对应子类为`MetaMonitorManagerKms`。

## MetaMonitorManagerKms

### LayoutMode

```c
/* Equivalent to the 'layout-mode' enum in org.gnome.Mutter.DisplayConfig */
typedef enum _MetaLogicalMonitorLayoutMode
{
  META_LOGICAL_MONITOR_LAYOUT_MODE_LOGICAL = 1,
  META_LOGICAL_MONITOR_LAYOUT_MODE_PHYSICAL = 2
} MetaLogicalMonitorLayoutMode;
```

只要系统中开启的`fractional scaling`的特性，默认使用`Logical`模式，否则为`Physical`模式。在`Physical`模式下，一个`MetaMonitor`的维度使用显示器分辨率，而`Logical`模式下使用显示器分辨率除以缩放系数作为维度。详见`org.gnome.Mutter.DisplayConfig.xml`文件中相关的描述。

### calculate_supported_scales

该函数计算一个显示器分辨率支持的缩放系数。这个函数实现的基本思路是枚举缩放系数，然后进行修正。从代码中可以看到枚举步长，最大值与最小值：

```c
#define SCALE_FACTORS_PER_INTEGER 4
#define SCALE_FACTORS_STEPS (1.0 / (float) SCALE_FACTORS_PER_INTEGER)
#define MINIMUM_SCALE_FACTOR 1.0f
#define MAXIMUM_SCALE_FACTOR 4.0f
#define MINIMUM_LOGICAL_AREA (800 * 480)
#define MAXIMUM_REFRESH_RATE_DIFF 0.001
```

即函数以0.25以步长，从1.0开始，依次1.0, 1.25, 1.5, 1.75 .... 直到4.0。经过枚举之后，枚举出来的缩放系数并不一定能够工作正常，需要进一步修正。首先明确终极目标，或者说什么情况下缩放系数才合法：存在分辨率`w x h`使得：$ a \times w = mode.w \and a \times h = mode.h $，其中w与h为整数。所以函数使用了一个算法，首先计算`floor(mode.w / a)`，这是一个整数，然后以这个整数为中心依次向两边推开，然后验证`mode.h / a`是否也是一个整数。经过这样的枚举，最终得到对一个分辨率可以使用的缩放系数。

### calculate_monitor_mode_scale

本质上调用`MetaMonitor`上的`calculate_mode_scale`计算一个`MetaMonitorMode`应该使用的缩放系数。实现方式比较naive，没有照顾到HiDPI屏幕，后面我可能在这里开刀，增加计算HiDPI屏幕缩放系数的相关代码。函数首先检查有没有设置全局缩放系数，如果设置了，就使用它。否则根据从`MetaMonitor`中读出显示器的分辨率和物理尺寸（毫米），然后计算出屏幕DPI，如果超过一定限度，则将缩放系数设置成2.0。

如果我要实现HiDPI屏幕的支持，计算出对应于`HiDPI`屏幕正确的缩放系数，那么我感觉需要进行如下步骤：

* 首先判断是否支持`fractional scaling`，不支持的话直接跳过
* 随后计算屏幕DPI
* 枚举所有支持的缩放系数，然后检查经过缩放后的DPI是否会落在一个合理范围内
* 第一个使DPI落在合理范围内的缩放系数即使我们想要的缩放系数