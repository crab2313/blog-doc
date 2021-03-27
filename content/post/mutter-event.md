+++
title = "Mutter事件处理框架分析"
date = 2020-11-25
draft = true


tags = ["mutter", "gnome"]

+++

本文处理对mutter事件处理的分析，其主要目的是调我机器上一个bug。简而言之，当我新插上一个显示器时，原先显示器上的全屏窗口的大小会发生改变，把原来gnome-shell的位置也占用了。为了调这个bug，我就来分析mutter事件处理框架了，主要涉及：

* wayland服务器如何向一个窗口发送调整大小的请求。
* mutter如何处理桌面输入，并将事件转发给对应的目的地。

## Clutter事件处理初始化

`MetaBackend`在实例化的时候会调用`init_clutter`函数，随后该函数会初始化clutter。clutter设置好后端之后，`init_clutter`函数会向主事件循环中加入一个GSource：

```c
static GSourceFuncs clutter_source_funcs = {
  clutter_source_prepare,
  clutter_source_check,
  clutter_source_dispatch
};
```

`prepare`和`check`回调函数，简单检查位于`ClutterMainContext`上的`events_queue`队列。而`disaptch`回调函数简单从这个队列中取下一个`ClutterEvent`，然后调用`clutter_do_event`函数处理这个事件。

从上面的线索可以看出来：

* ClutterEvent是Clutter中表示事件的基本单元
* ClutterMainContext存在一个队列events_queue，保存所有的事件
* GMainContext中存在event source，用于处理事件

### ClutterEVent

`ClutterEvent`的定义如下，非常直观：

```c
union _ClutterEvent
{
  /*< private >*/
  ClutterEventType type;

  ClutterAnyEvent any;
  ClutterButtonEvent button;
  ClutterKeyEvent key;
  ClutterMotionEvent motion;
  ClutterScrollEvent scroll;
  ClutterStageStateEvent stage_state;
  ClutterCrossingEvent crossing;
  ClutterTouchEvent touch;
  ClutterTouchpadPinchEvent touchpad_pinch;
  ClutterTouchpadSwipeEvent touchpad_swipe;
  ClutterProximityEvent proximity;
  ClutterPadButtonEvent pad_button;
  ClutterPadStripEvent pad_strip;
  ClutterPadRingEvent pad_ring;
  ClutterDeviceEvent device;
  ClutterIMEvent im;
};
```

这是C语言的常见写法，简单来说，通过固定字段`type`可以决定事件的类型，然后每一个个事件类型的事件携带自己特有的数据。`ClutterEventType`的定义上的注释基本写明了所有的Clutter事件。所有类型的`ClutterEvent`的开头都存在如下字段：

```c
struct _ClutterAnyEvent
{
  ClutterEventType type;
  guint32 time;
  ClutterEventFlags flags;
  ClutterStage *stage;
  ClutterActor *source;
};
```

简单来书，所有的Clutter事件都存在一个时间戳、标志位、stage以及actor。

### clutter_do_event

函数从前面看到就是处理排在`ClutterEvent`队列中的元素的处理函数。函数首先检查`ClutterEvent`是否设置好了`stage`字段，没有设置的事件一律不处理：

```c
  if (event->any.stage == NULL)
    {
      g_warning ("%s: Event does not have a stage: discarding.", G_STRFUNC);
      return;
    }
```

如果`ClutterStage`正被解构，则该事件同样不处理：

```c
  /* stages in destruction do not process events */
  if (CLUTTER_ACTOR_IN_DESTRUCTION (event->any.stage))
    return;
```

最后函数根据其目标`ClutterStage`将其压入`ClutterStage`对应的事件队列：

```c
  /* Instead of processing events when received, we queue them up to
   * handle per-frame before animations, layout, and drawing.
   *
   * This gives us the chance to reliably compress motion events
   * because we've "looked ahead" and know all motion events that
   * will occur before drawing the frame.
   */
  _clutter_stage_queue_event (event->any.stage, event, TRUE);
```

进而，Clutter的事件处理从`ClutterMainContext`转到了`ClutterStage`进行处理。

### _clutter_stage_queue_event



## Mutter事件处理框架

`MetaDisplay`被创建时，其构造函数调用了`meta_display_init_events`函数对事件处理进行了初始化。函数简单调用了`clutter_event_add_filter`函数，向Clutter注册了一个钩子函数，日后Clutter处理所有的`ClutterEvent`之前，都要提前调用这个钩子函数。这个注册的钩子函数实际上就是`meta_display_handle_event`。



## Mutter窗口最大化

Mutter窗口最大化由MutterWindow控制，对应的接口函数为`meta_window_maximize`。函数除了进行操作的窗口之外，只有一个参数`MetaMaximizeFlags`，本质上是两个标志位，每一位分别对应水平、竖直最大化。

```c
typedef enum
{
  META_MAXIMIZE_HORIZONTAL = 1 << 0,
  META_MAXIMIZE_VERTICAL   = 1 << 1,
  META_MAXIMIZE_BOTH       = (1 << 0 | 1 << 1),
} MetaMaximizeFlags;
```

首先检测窗口是否需要竖直最大化，如果需要，则关闭窗口的阴影效果。随后检测窗口是否已经被放置在屏幕上，如果没有，则将该操作缓存起来，等到进行放置的时候才进行最大化操作。

接下来的操作分成两步：最大化与更改大小。