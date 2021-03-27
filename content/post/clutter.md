+++
title = "Clutter代码分析"
date = 2020-10-19
draft = true


tags = ["gnome", "clutter", "mutter"]

+++

本文尝试分析Clutter代码实现，其目的是获取调试Mutter代码bug的能力。Clutter的基本知识可以从网络获取，后面会贴一些基础文档。Clutter是mutter的基本组成部件，如果不能正确理解clutter，那么就无法理解mutter对屏幕进行的绘制。

首先clutter绘制时要考虑多显示器的支持，ClutterStage是一个大的坐标系，或者说绘制区域，而我们需要一块区域来表示显示器输出的区域，这就是`ClutterStageView`。`ClutterActor`作为一个舞台上的"演员"，是一个2D的材质，被放到了`ClutterStage`所表示的3D坐标系中。其中camera位于坐标原点，且方向为(0,0,-1)，即向下看。

[参考资料 - Mutter Wiki](https://gitlab.gnome.org/GNOME/mutter/-/wikis/Clutter%20Rendering%20Model)

## ClutterStage

从抽象的概念上看，`ClutterStage`是最高一层的可见对象，这句话是指上是说整个渲染节点是组织成树状的，而`ClutterStage`就是它的根节点。从这个意义上来说，很明显`ClutterStage`是一个特殊的`ClutterActor`，因此在`Clutter`中被实现为`ClutterActor`的一个子类。

### clutter_stage_paint_view

### clutter_stage_init

`ClutterStage`并没有`_new`函数，说明`Mutter`中并不直接使用这个类，而是使用它的子类。从`_init`函数中可以看到，`ClutterStage`中保存了一个后端`ClutterStageWindow`，该对象由`ClutterBackend`创建。viewport也是从`ClutterStageWindow`中获取的，但是很明显`viewport`就是整个渲染区域（单屏幕下应该为屏幕大小）。



### _clutter_stage_queue_actor_redraw

分析`ClutterActor`的redraw操作时见到了该函数，其本质就是将`ClutterActor`的更新请求排列到redraw队列上。这个redraw队列的元素如下：

```
struct _ClutterStageQueueRedrawEntry
{
  ClutterActor *actor;
  gboolean has_clip;
  ClutterPaintVolume clip;
};
```

函数所作的第一件事就是检查`pending_finish_queue_redraws`标志，如果为false，则会对当前`ClutterStage`上外挂的所有`ClutterStageView`执行schedule_update操作，然后将标志置为true。该操作的潜在逻辑就是一旦`pending_queue_redraws`队列上有元素加入，说明整个`ClutterStageView`需要更新，所以要调度起一个update操作（具体到Native后端，即一次屏幕的刷新）。这里注意如果有两个显示器，那么两个显示器的刷新是会互相干涉的，后面的VRR实现应该要改变这一行为，然显示器不受其他显示器上才显示的`ClutterActor`的影响。

后面的操作则简单，首先明确一个`ClutterActor`在这个queue里只能有一个entry，该entry被保存在`ClutterActor`中。当`ClutterActor`调用该函数时，需要将entry传入，即函数的`entry`参数。如果`entry`为`NULL`，则该`ClutterActor`不在`redraw`队列中，需要重新创建`ClutterStageQueueRedrawEntry`。反之，则需要对entry进行更新。接下来的分析应该围绕这个队列`stage->priv->pending_queue_redraws`进行。

### clutter_stage_maybe_finish_queue_redraw

进一步搜索之后，发现`stage->priv->pending_queue_redraws`只有这个函数用到了，这个函数就是对所有`ClutterActor`进行重绘制的函数。函数开头与`_clutter_stage_queue_actor_redraw`对应，一旦发现`pending_finish_queue_redraws`为`FALSE`，说明没有`ClutterActor`需要重新绘制，则直接返回。反之，则将其置为`FALSE`，然后进行进一步的处理。

函数末尾本质就是将`pending_queue_redraws`队列中的元素一个一个取下，然后依次调用`_clutter_actor_finish_queue_redraw`函数。也就是说，当`ClutterActor`需要进行更新的时候，`Clutter`并不是直接调用后端对其进行更新，而是先将该请求缓存起来，最后在`ClutterStageView`需要更新的时候一同进行更新，这样可以避免一些重复操作。可以看到，调用`clutter_stage_maybe_finish_queue_redraw`函数的直接位置就是`ClutterStageView`中`ClutterFrameClock`的`frame`回调函数。

## ClutterFrameClock

注意这个对象是为了实现多显示器使用独立刷新率时实现的对象，本质表示一个显示器的FrameClock。`ClutterFrameClock`直接继承GObject，与其一同定义的还有一组接口：

```c
typedef struct _ClutterFrameListenerIface
{
  void (* before_frame) (ClutterFrameClock *frame_clock,
                         int64_t            frame_count,
                         gpointer           user_data);
  ClutterFrameResult (* frame) (ClutterFrameClock *frame_clock,
                                int64_t            frame_count,
                                int64_t            time_us,
                                gpointer           user_data);
} ClutterFrameListenerIface;
```

创建`ClutterFrameClock`时，需要传入该对象：

```c
CLUTTER_EXPORT
ClutterFrameClock * clutter_frame_clock_new (float                            refresh_rate,
                                             const ClutterFrameListenerIface *iface,
                                             gpointer                         user_data);
```

### clutter_frame_clock_new

该函数创建一个`ClutterFrameClock`对象。`ClutterFrameClock`的init函数非常简单，如下：

```c
static void
clutter_frame_clock_init (ClutterFrameClock *frame_clock)
{
  frame_clock->state = CLUTTER_FRAME_CLOCK_STATE_INIT;
}
```

`clutter_frame_clock_new`函数会调用`g_object_new`创建对象，然后将参数保存在对象中，最后调用`init_frame_clock_source`进行初始化操作。事件驱动编程的核心就是事件源与对应的处理，和明显`ClutterFrameClock`可以作为一个事件源，`init_frame_clock_source`函数的核心就是创建这样一个事件源(GSource)，然后加入到当前线程的事件循环中。可以看到，这个`GSource`的`GSourceFuncs`为`frame_clock_source_funcs`：

```c
static GSourceFuncs frame_clock_source_funcs = {
  NULL,
  NULL,
  frame_clock_source_dispatch,
  NULL
};
```

看到`prepare`和`check`都为NULL，这说明`prepare`和`check`都默认为NULL。事实上这个`GSource`使用`Ready Time`触发dispatch操作。dispatch函数最终会调用`clutter_frame_clock_dispatch`函数，后面进行分析。

### clutter_frame_clock_schedule_update

该函数用于调度起一次更新操作，需要注意`ClutterFrameClock`本身是一个状态机，有如下状态：

```c
typedef enum _ClutterFrameClockState
{
  CLUTTER_FRAME_CLOCK_STATE_INIT,
  CLUTTER_FRAME_CLOCK_STATE_IDLE,
  CLUTTER_FRAME_CLOCK_STATE_SCHEDULED,
  CLUTTER_FRAME_CLOCK_STATE_DISPATCHING,
  CLUTTER_FRAME_CLOCK_STATE_PENDING_PRESENTED,
} ClutterFrameClockState;
```

在不同状态下，调用`clutter_frame_clock_schedule_update`所在成的效果是不同的。`ClutterFrameClock`刚被创建时，其状态为`INIT`，则调用该函数时如下：

```c
  switch (frame_clock->state)
    {
    case CLUTTER_FRAME_CLOCK_STATE_INIT:
      next_update_time_us = g_get_monotonic_time ();
      break;
```

其中`next_update_time_us`为后面设置`GSource`的`Ready Time`时的事件戳，把其设置为当前时间，也就是立即进行`dispatch`操作的意思。而`IDLE`状态时，会计算该时间戳：

```c
    case CLUTTER_FRAME_CLOCK_STATE_IDLE:
      calculate_next_update_time_us (frame_clock,
                                     &next_update_time_us,
                                     &frame_clock->next_presentation_time_us);
      frame_clock->is_next_presentation_time_valid = TRUE;
```

计算方式比较复杂，最后进行分析。如果当前为`SCHEDULED`状态，即`GSource`已经设置好了`Ready Time`，正在等待dispatch，则直接返回。对于`DISPATCHING`和`PENDING_PRESENTED`状态，则设置`pending_reschedule`标志，然后返回。对于`IDLE`和`INIT`，计算完`next_update_time_us`后将会设置对应的`Ready Time`，然后将状态设置为`SCHEDULED`。

### clutter_frame_clock_dispatch

前面提到了这个函数是对应的`dispatch`函数。函数第一件做的事情就是清空`Ready Time`，停止`dispatch`后续的触发，然后将`ClutterFrameClock`的状态设置为`DISPATCHING`。随后函数自增记录`frame`产生个数的计数器，然后调用创建`ClutterFrameClock`时传入的`before_frame`回掉函数。接下来更新`ClutterFrameClock`中注册的`ClutterTimeline`对象，然后调用`frame`回调函数。函数末尾会根据`frame`回调函数返回的值决定`ClutterFrameClock`的状态：

```c
    case CLUTTER_FRAME_CLOCK_STATE_DISPATCHING:
      switch (result)
        {
        case CLUTTER_FRAME_RESULT_PENDING_PRESENTED:
          frame_clock->state = CLUTTER_FRAME_CLOCK_STATE_PENDING_PRESENTED;
          break;
        case CLUTTER_FRAME_RESULT_IDLE:
          frame_clock->state = CLUTTER_FRAME_CLOCK_STATE_IDLE;
          maybe_reschedule_update (frame_clock);
          break;
        }
```

## ClutterStageView

只有`Mutter`中的`Clutter`有该类。从`Mutter`的角度来看，`ClutterStageView`将一个`ClutterStage`的特定区域渲染到一个屏幕区域，其实质上承担的任务为多显示器的输出工作。除此之外，`ClutterStageView`还持有前后端的FrameBuffer，并且在最近的改动中管理FrameClock。

`ClutterStageView`是一个GObject，作为基类被具体实现的子类继承。

### Frame Clock回调函数

前面提到了`ClutterStageView`内部包含了一个`ClutterFrameClock`，我们可以在该对象的创建函数中看到：

```c
  priv->frame_clock = clutter_frame_clock_new (priv->refresh_rate,
                                               &frame_clock_listener_iface,
                                               view);
```

现在来分析其注册的两个回调函数：

```c
static const ClutterFrameListenerIface frame_clock_listener_iface = {
  .before_frame = handle_frame_clock_before_frame,
  .frame = handle_frame_clock_frame,
};
```

其中`before_frame`函数做的事情比较单一，就是处理当前`ClutterStageView`等待处理的事件：

```c
static void
handle_frame_clock_before_frame (ClutterFrameClock *frame_clock,
                                 int64_t            frame_count,
                                 gpointer           user_data)
{
  ClutterStageView *view = user_data;
  ClutterStageViewPrivate *priv =
    clutter_stage_view_get_instance_private (view);

  _clutter_stage_process_queued_events (priv->stage);
}
```

然后`frame`回调函数的处理就相当复杂了，毕竟是整个Mutter渲染的核心驱动，涉及了大量的Clutter实现细节。

TODO

## ClutterStageViewCogl

这里只关注`redraw_view`回调函数。可以看到`ClutterStageViewCogl`是实现了`ClutterStageWindow`接口，且前面发现在`ClutterStageView`的`frame`回调函数中调用了`_clutter_stage_window_redraw_view`函数，这里就来分析这个函数。`ClutterStageViewCogl`实现的接口如下：

```c
static void
clutter_stage_window_iface_init (ClutterStageWindowInterface *iface)
{
  iface->realize = clutter_stage_cogl_realize;
  iface->unrealize = clutter_stage_cogl_unrealize;
  iface->get_wrapper = clutter_stage_cogl_get_wrapper;
  iface->resize = clutter_stage_cogl_resize;
  iface->show = clutter_stage_cogl_show;
  iface->hide = clutter_stage_cogl_hide;
  iface->get_frame_counter = clutter_stage_cogl_get_frame_counter;
  iface->redraw_view = clutter_stage_cogl_redraw_view;
}
```

`clutter_stage_cogl_redraw_view`进行了scanout操作：

```c
  scanout = clutter_stage_view_take_scanout(view);
  if (scanout)
  {
    g_autoptr(GError) error = NULL;

    if (clutter_stage_cogl_scanout_view(stage_cogl, view, scanout, &error))
      return;

    g_warning("Failed to scan out client buffer: %s", error->message);
  }
```

该操作直接调用`cogl_onscreen_direct_scanout`，进而调用原先在Native后端从winsys注册的`onscreen_direct_scanout`函数指针~~，进而调用Native后端的相关代码，进行pageflip~~。这里是`direct_scanout`，目测是unredirection模式，即直接将一个buffer的内容scanout到显示器上。之后，调用`clutter_stage_cogl_redraw_view_primary`函数。

### clutter_stage_cogl_redraw_view_primary

该函数应该就是主要的redraw函数。负责重新绘制整个显示器的framebuffer。函数传入的参数为`ClutterStageCogl`和`ClutterStageView`，也就是所有与更新相关的信息都记录在了`ClutterStageView`中。函数首先获取framebuffer信息：

```c
  clutter_stage_view_get_layout (view, &view_rect);
  fb_scale = clutter_stage_view_get_scale (view);
  fb_width = cogl_framebuffer_get_width (fb);
  fb_height = cogl_framebuffer_get_height (fb);
```

`ClutterStageView`中保存了一个`redraw_clip`：

```c
  redraw_clip = clutter_stage_view_take_redraw_clip (view);
```

如果redraw_clip为空，则会进行full_redraw也就是全屏重新绘制。

## ClutterActor

### clutter_actor_queue_redraw

分析这个函数的目的是想搞明白一个`ClutterActor`是如何绘制到屏幕上的。从函数名称看该函数将对应的`ClutterActor`排到redraw队列，等待下一次绘制。这里就需要搞明白：

* 这个绘制队列究竟是什么
* 什么时候进行绘制
* 绘制的原理是什么样子的

函数本质直接调用`_clutter_actor_queue_redraw_full`，且提供的`ClutterPaintVolume`和`ClutterEffect`都是`NULL`。我们知道每一个`ClutterActor`都是有parent的，最顶层的parrent就是`ClutterStage`，函数获取到`ClutterStage`之后，调用了`_clutter_stage_queue_actor_redraw`：

```c
  self->priv->queue_redraw_entry =
    _clutter_stage_queue_actor_redraw (CLUTTER_STAGE (stage),
                                       priv->queue_redraw_entry,
                                       self,
                                       volume);
```

而该函数的操作可以在`ClutterStage`的分析中看到。由于前面看到`CluterEffect`为`NULL`，则函数直接将自己的`is_dirty`属性设置为`TRUE`。至此，函数结束，可以看到函数最大的工作还是委托给了`ClutterStage`进行。

### _clutter_actor_finish_queue_redraw

从`ClutterStage`的分析中可以看到，该函数与`clutter_actor_queue_redraw`不同，是直接进行`ClutterActor`重绘制的函数。函数首先清空`priv->queue_redraw_entry`，毕竟函数被调用时，`pending_queue_redraws`队列正在进行清空操作。