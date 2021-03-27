+++
title = "USB Hub驱动分析"
date = 2018-02-27


tags = ["kernel", "usb", "driver"]
+++

# USB hub框架分析

USB hub框架是usbcore模块里的一个大头5000行代码（5分之一）。同时，USB hub本身在USB协议中又是一个非常重要的组成部分，直接关系到USB的核心功能。理解USB hub框架对于理解usbcore模块至关重要。

## 初始化

USB hub框架随着USB子系统一起初始化，usb_init函数中直接调用usb_hub_init函数。usb_hub_init函数中只进行了两件事：

* 注册hub_driver
* 申请一个名为hub_wq的workqueue

与usb_hub_init函数对应的usb_hub_cleanup函数做了相反的事情。在USB协议中，USB hub也是一个USB设备，也提供了所有USB设备都会提供的通用请求，除此之外也提供了Hub特定的请求。因此，自然而然应该将Hub看作普通USB设备，并提供对应的USB驱动，这样做可以简化实现。可以看到内核中也是这么做的：

```c
static struct usb_driver hub_driver = {
        .name =         "hub",
        .probe =        hub_probe,
        .disconnect =   hub_disconnect,
        .suspend =      hub_suspend,
        .resume =       hub_resume,
        .reset_resume = hub_reset_resume,
        .pre_reset =    hub_pre_reset,
        .post_reset =   hub_post_reset,
        .unlocked_ioctl = hub_ioctl,
        .id_table =     hub_id_table,
        .supports_autosuspend = 1,
};
```

hub_id_table中匹配了bDeviceClass或者bInterfaceClass为USB_CLASS_HUB的设备或者接口。

## hub_irq

前面提到Hub初始化之后会激活对Status Change Endpoint的轮询操作，为此usb_hub->urb指针保存了一个Interrupt类型的URB，并将其处理函数设置为hub_irq。也就是说，hub_irq函数的作用是处理该轮询结果。从协议中可以得知，每次轮询成功时返回的数据是一个记录着Hub及Hub上的所有Port状态改变的Bitmap。hub_irq的实现比较简单，主要流程如下：

1. 每次出现错误状态时增加错误计数器usb_hub->nerror。计数器超过10之后，设置usb_hub->error，清零错误计数器，并调用kick_hub_wq。
2. 如果没有错误，则清零错误计数器，然后调用kick_hub_wq
3. 如果usb_hub->quiescing不为真，则重新提交该URB

事实上一旦usb_hub->error不为0，则kick_hub_wq后续触发的hub_event函数中会对该Hub进行重置操作。

## hub_event

旧的内核中使用一个内核线程轮询读取Hub的状态，而新的内核中换成了前面提到的名为hub_wq的workqueue，该workqueue在USB子系统初始化的时候创建。内核使用hub_event函数轮询Hub的状态，为了向该函数传入参数，每个usb_hub结构体内都嵌入了work_struct（名为events），通过container_of获取对应的usb_hub。

该函数在做完简单的错误处理之后开始枚举PORT状态，枚举机制可以参考USB2.0协议文档中的11.12.3章节。简而言之，驱动程序在轮询Hub的Status Change Endpoint过程中，如果获取到有状态更新事件，那么就会调用kick_hub_wq函数将usb_hub->events加入到hub_wq中。同一时刻只有一个usb_hub的events能放到hub_wq中，这跟Hub设备的事件通知机制有关，驱动在获取到Hub的事件更新之后，必须手动清除掉一个事件的状态更改位，已使Hub设备不再报告相同的事件。也就是说，没有处理完的事件在下次轮询的时候依然存在。

很显然根据Hub的规范，hub_event函数需要做的就是从Default Control Pipe中调用对应的请求获取Hub或者Port的状态信息，并清除对应的状态位。函数中当然不可能轮询所有的Port和Hub状态，这样做效率太低。事实上，每次轮询Status Change的时候，返回的数据是一个Bitmap，其中第0位置1表示Hub本生状态发生了改变，而后续的第n位置1表示第n个Port的状态发生了改变。如果你阅读Hub规范，则会发现hub_event函数承担的责任并仅仅是处理硬件状态的改变，还包括处理软件状态的改变（即软件控制信息的处理）。也就是说，如果内核需要更改Hub的状态，则会修改usb_hub中的状态，并调用kick_hub_wq函数。

从hub_irq函数中可以看到，内核将前面提到的Bitmap保存在usb_hub->event_bits中，并调用kick_hub_wq。因此，hub_event函数可以通过这个bitmap来判断是否有硬件事件的发生。hub_event中调用port_event处理Port事件：如果一个Port的change_bits、event_bits、wakeup_bits中的对应位任意一位置一，则需要进行处理。port_event函数开头先清除了event_bits和wakeup_bits，看样子这两个标志仅仅起通知的作用。change_bits则被保存了下来，内核通过这一标志表示逻辑连接状态的改变。

```c
        connect_change = test_bit(port1, hub->change_bits);
        clear_bit(port1, hub->event_bits);
        clear_bit(port1, hub->wakeup_bits);
```

随后函数从Default Control Pipe读取Port相关的信息并进行处理。可以发现内核对大部分信息仅仅是作为调试信息记录下来，然后将其清除掉，已使下一次轮询时不再获取该信息，真正进行处理的信息不多。首先处理的是USB_PORT_STAT_C_CONNECTION，从名字上看得出是连接状态的改变，因此如果这个标志被发现的话意味着物理连接状态的改变，需要将connect_change直接置一。

```c
        if (portchange & USB_PORT_STAT_C_CONNECTION) {
                usb_clear_port_feature(hdev, port1, USB_PORT_FEAT_C_CONNECTION);
                connect_change = 1;
        }
```

其次就是处理远程唤醒和USB3.0的重置操作。最后如果connect_change为1的话（即Port的逻辑或者物理连接状态改变）则调用hub_port_connect_change函数进行处理。