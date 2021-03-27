+++
title = "Linux设备模型：bus与driver"
date = 2018-04-28


tags = ["kernel", "driver"]

+++

# bus & driver

bus core的实现在`driver/base/bus.c`中，初始化过程由buses_init函数实现，函数中创建了两个kset：bus_kset（name为bus）和system_kset（name为system）。初始化过程这么简单的话bus注册的工作应该就会比较复杂了。

## subsys_private

为了隐藏bus core内部的状态，每个bus_type都保存一个指向subsys_private结构体的指针，这个数据结构只能由bus core中进行操作。其中subsys即为表示该bus的kset，subsys中还将放置另外两个kset，即结构体中的devices_kset和drivers_kset，很明显这两个kset用于存放驱动和设备。

```c
struct subsys_private {
        struct kset subsys;
        struct kset *devices_kset;
        struct list_head interfaces;
        struct mutex mutex;

        struct kset *drivers_kset;
        struct klist klist_devices;
        struct klist klist_drivers;
        struct blocking_notifier_head bus_notifier;
        unsigned int drivers_autoprobe:1;
        struct bus_type *bus;

        struct kset glue_dirs;
        struct class *class;
};
```

## bus_register

这个函数应该是bus core最广为人知的入口了，可以将一个bus_type注册到内核的bus子系统中。抛开大部分内存申请等常规初始化过程不谈，函数中首先创建了一个subsys_private对象，并将其的指针保存到bus->p中。随即将bus->p->kobj的name设置为bus->name，并将这个kset加入到前面提到的bus_kset中。顺便一提表示一个bus的kset使用bus_ktype作为其类型，后面详细说明。

接下来函数调用kset_create_and_add创建devices_kset和drivers_kset，并将其加入表示bus的kset。在做完以上工作后，函数向表示该bus的kset中加入了三个属性：

* uevent
* drivers_autoprobe
* drivers_probe

而传入bus_register的bus_type->bus_group也会被当作attribue_group注册到bus对应的sysfs文件夹中。

## bus的三个属性

前面提到表示bus的kset中加入了三个属性，这里列出他们的定义：

```c
static BUS_ATTR(uevent, S_IWUSR, NULL, bus_uevent_store);
static BUS_ATTR(drivers_probe, S_IWUSR, NULL, store_drivers_probe);
static BUS_ATTR(drivers_autoprobe, S_IWUSR | S_IRUGO,
                show_drivers_autoprobe, store_drivers_autoprobe);
```

这三个属性会作为文件存在于sysfs中。首先drivers_autoprobe是比较好理解的，subsystem_private中保存了一个drivers_autoprobe值，如果为true则进行驱动与设备的自动匹配。向sysfs中一个bus对应文件夹下的drivers_autoprobe写入1或者0则可以改变它的值。

uevent则作为一个调试uevent的接口，通过向该文件写入对应的信息，内核可以调用kobject_synth_uevent函数生成对应的uevent事件。可以直接使用echo进行该操作，而写入字符串的格式如下：

```
ACTION [UUID] [KEY1=VALUE1] [KEY2=VALUE2] ...
```

这里写入的环境变量会以SYNTH_ARG_KEY=VALUE的形式出现在uevent信息中。最后一个drivers_probe属性可以让总线重新为一个没有匹配到驱动的设备进行驱动匹配。将想要进行匹配的设备名写入到drivers_probe即可进行该操作。

## subsys_system_register

文档里说这个接口不要在新代码中用，接口本身只做兼容性用途。前面提到了一个system_ket，这里需要提一下它是如何初始化的：

```c
        system_kset = kset_create_and_add("system", NULL, &devices_kset->kobj);
```

可以猜到devices_kset应该`/sys/devices`文件夹，也就是说system_ket的路径为`/sys/devices/system`。首先函数注册了一个假设备，名字与传入的bus_type一致，然后将这个设备的parent设置成system_kset.kobj，并将bus_type->dev_root设置刚才注册的那个假设备。总线新注册设备时会将其parent设置成dev_root，因此使用subsys_system_register函数注册的总线（子系统）会被放到`/sys/devices/system/<bus-name>/`底下。我们常见的子系统有clocksource，cpu，memory等等。

## bus通知机制

subsys_private中有一个bus_notifier，他是一个blocking_notifier。内核各个子系统之间为了高效进行信息传递使用注册监听机制，称作notifier。这里使用的是blocking_notifier，其特点是发送事件是可以阻塞。当然，bus core负责事件的发送，所以我们最多进行事件的监听。

## bus_add_device

一个bus从逻辑上来讲是要管理设备和驱动的，所以应该提供对应的接口将总线或者驱动加入到bus中来。但是事实上，bus_add_device并没有被export出来，也就是说内核模块是不能调用这个函数的。

每个设备在创建时都应将其bus指针设置成其对应的bus_type，所以函数第一步做的就是将其bus_type取出来。接下来所作大部分是sysfs相关的处理：

* 调用device_add_groups将bus->dev_groups加入到设备中去，翻译成白话就是在设备对应的sysfs文件夹中加入总线自己定义的一些属性。
* 调用sysfs_create_link创建从设备到bus->p->devices_kset的符号连接，也就是在sysfs中建立从设备文件夹到bus/devices文件夹下的符号连接。
* 调用sysfs_create_link在设备文件夹内创建一个名为subsystem的符号连接，该链接指向bus文件夹。

最后将设备挂入bus维护的设备列表中。

## bus_add_driver

类似于device，driver也有一个bus指针指向它应该属于的bus_type。传进来的device_driver本身没有kobject，bus_add_driver会创建一个嵌入了kobject的driver_private结构体并将其保存在device_driver中。很明显，这个被创建的kobject的kset指针需要被设置成bus->p->drivers_kset，即其注册入sysfs时会被放入`/sys/bus/<bus-name>/drivers/`文件夹中。

如果bus的drivers_autoprobe为true，则bus_add_driver会尝试进行设备匹配。与bus设备一样，driver也被加入了uevent属性，也就是driver对应的文件夹下也有uevent文件，向其写人对应的命令也能触发uevent事件。类似于device，bus->drv_groups保存的attribute_group向该bus中的driver追加了bus预先设置好的属性。

如果device_driver里的suppress_bind_attrs不为true，则bus_add_driver应该向其追加两个属性：bind和unbind。

```c
static DRIVER_ATTR_WO(unbind);
static DRIVER_ATTR_WO(bind);
```

这个文件的作用是相反的：将设备名写入在一个driver文件夹下的unbind文件时，会使driver释放掉该设备，bind文件则会使driver试图匹配该设备。最后可以看一下这一行：

```c
        module_add_driver(drv->owner, drv);
```

该函数的主要作用为更新引用计数，并创建两个符号链接：driver文件夹下的module，`<module>/drivers/`文件夹下的driver。

## driver_register

这个函数其实没有做什么事情，大部分工作都由bus承担了。函数首先调用driver_find查找bus上是否已经有了同名驱动，如果有则报错退出。接下来调用bus_add_driver将其注册到bus中去。如果device_driver->groups不为NULL，则将其添加到自身的属性中。最后触发一个ADD类型的uevent事件。