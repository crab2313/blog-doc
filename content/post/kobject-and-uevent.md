+++
title = "Linux设备模型：kobject与uevent"
date = 2018-04-28


tags = ["kernel", "driver"]
+++

# kobject

```c
struct kobj_type {
        void (*release)(struct kobject *kobj);
        const struct sysfs_ops *sysfs_ops;
        struct attribute **default_attrs;
        const struct kobj_ns_type_operations *(*child_ns_type)(struct kobject *kobj);
        const void *(*namespace)(struct kobject *kobj);
};
```

```c
struct kobj_ns_type_operations {
        enum kobj_ns_type type;
        bool (*current_may_mount)(void);
        void *(*grab_current_ns)(void);
        const void *(*netlink_ns)(struct sock *sk);
        const void *(*initial_ns)(void);
        void (*drop_ns)(void *);
};
```

```c
struct kobject {
        const char              *name;
        struct list_head        entry;
        struct kobject          *parent;
        struct kset             *kset;
        struct kobj_type        *ktype;
        struct kernfs_node      *sd; /* sysfs directory entry */
        struct kref             kref;
#ifdef CONFIG_DEBUG_KOBJECT_RELEASE
        struct delayed_work     release;
#endif
        unsigned int state_initialized:1;
        unsigned int state_in_sysfs:1;
        unsigned int state_add_uevent_sent:1;
        unsigned int state_remove_uevent_sent:1;
        unsigned int uevent_suppress:1;
};
```

## kobject_init

初始化时必须传入一个ktype参数，否则无法成功初始化，这个是必然的，每个嵌入kobject的结构体必须在引用计数变为0的时候必须使用ktype中保存的release函数指针进行销毁。kobj中保存了一个state_initialized字段，用以标记该kobject是初始化过的，从这里也能看出申请完kobject结构体之后必须将其全部写零。接下来初始化kobj->kref作为引用计数，初始化kobj->entry（这个list_head用于将kobject保存到一个kset中）。最后可以看到kobject还有三个字段用于标记其状态：

* state_in_sysfs
* state_add_uevent_sent
* state_remove_uevent_sent

从名字中即可看出其表示的含义，后两个状态表示对应的uevent事件是否已经发送给用户态。

## kobject_add

函数首先检查state_initialized字段，即kobject是否已经通过kobject_init进行初始化。随后设置kobject的name和parent，并确认name不为空。该函数用于将kobject注册到内核中，如果kobject的kset字段已经被设置，则将这个kobject加入到那个kset中。函数最后调用create_dir函数创建kobject对应于sysfs中的文件夹，如果创建成功，则将state_in_sysfs设置为1。

随后来看create_dir函数，首先可以看到函数开头调用了kobject_namespace函数，从这里立即可以看出kobject是namespace-aware的。kboject_namespace的实现如下：

```c
const void *kobject_namespace(struct kobject *kobj)
{
        const struct kobj_ns_type_operations *ns_ops = kobj_ns_ops(kobj);

        if (!ns_ops || ns_ops->type == KOBJ_NS_TYPE_NONE)
                return NULL;

        return kobj->ktype->namespace(kobj);
}
```

也就是说，如果kset的kobj_type中的kobj_ns_type_operations->type不为KOBJ_NS_TYPE_NONE（目前应该只有一个KOB_NS_TYPE_NET）的话就用kobject->kobj_type中的namespace回调函数给出的namespace标签创建sysfs中对应的文件夹：

```c
        error = sysfs_create_dir_ns(kobj, kobject_namespace(kobj));
```

并利用kobject->kobj_type->attributes中保存的属性创建文件夹中的文件，这里具体的实现需要参考sysfs文件系统的实现。

## kobject_get_path

很明显该函数返回该kobject的path，从代码中可以看出path是指在该kobject所在的树形结构中，从该kobject到树的根节点所经过的路径。函数实现比较简单，首先是遍历一遍树到根结点，计算出整个字符串所占用的空间。然后申请这个字符串，然后再次向上遍历树至根结点，依次从后向前填充每个kobject节点的名字，并以斜杠（“/”）作为间隔。

## kobject_release

该函数将kobject所占用的系统资源删除。需要注意的是，这个函数并不仅仅释放kboject占用的内存，而是释放嵌入kobject的结构体占用的内存和系统资源。每个kobject的kobj_type->release不能为NULL，否则kobject的无法在其以用计数为0时释放内存。然后函数首先检查state_add_uevent_sent是否为true，如果是则调用kobject_uevent发送其对应的REMOVE事件到用户态。如果state_in_sysfs为true，则表示该kobject还在sysfs中显示，需要调用kobject_del函数将其从树形结构中删除。最后调用release函数指针将被嵌入这个kobject的结构体删除，当然，不能忘记释放name占用的内存。

## kobject_create

该函数创建并初始化一个“动态”的kobject结构体对象。之所以说是动态的，是因为它们没有被嵌入到别的结构体中，而时独立存在的。所以kobject_create在创建kobject之后，给其了一个默认的release函数，即释放kobject本身所占用的内存。kobject_object创建的kobject对象是匿名的，且使用如下的kobj_type：

```c
static struct kobj_type dynamic_kobj_ktype = {
        .release        = dynamic_kobj_release,
        .sysfs_ops      = &kobj_sysfs_ops,
};

static void dynamic_kobj_release(struct kobject *kobj)
{
        pr_debug("kobject: (%p): %s\n", kobj, __func__);
        kfree(kobj);
}
```

# uevent

uevent的实现可以在`lib/kobject_uevent.c`文件中找到。我们首先注意到的是uevent依赖于网络子系统，如果内核没有开启CONFIG_NET选项，则uevent的很多功能都会受到影响。同时也能猜到uevent是namepsace-aware的，由于uevent依赖于netlink套接字向用户态发送uevent事件，所以每个net namespace都可以有自己独立的uevent事件。

## uevent与netlink

uevent使用netlink套接字向用户态发送信息，而netlink是namespace-aware的，所以uevent的初始化必须对每个net namespace区分对待。

```c
static struct pernet_operations uevent_net_ops = {
        .init   = uevent_net_init,
        .exit   = uevent_net_exit,
};

static int __init kobject_uevent_init(void)
{
        return register_pernet_subsys(&uevent_net_ops);
}
```

register_pernet_subsys是注册net namespace初始化函数的标准方式，每当新的net namespace创建或者销毁的时候，这注册的函数指针就会执行。由于每个net namespace的netlink套接子都是相互独立的，uevent_net_init负责在每个net namepsace中注册对应的netlink套接字，并将其保存在一个列表中。

```c
struct uevent_sock {
        struct list_head list;
        struct sock *sk;
};
static LIST_HEAD(uevent_sock_list);
```

## uevent事件类型

```c
static const char *kobject_actions[] = {
        [KOBJ_ADD] =            "add",
        [KOBJ_REMOVE] =         "remove",
        [KOBJ_CHANGE] =         "change",
        [KOBJ_MOVE] =           "move",
        [KOBJ_ONLINE] =         "online",
        [KOBJ_OFFLINE] =        "offline",
        [KOBJ_BIND] =           "bind",
        [KOBJ_UNBIND] =         "unbind",
};
```

TODO: 从内核设备框架中找到以上事件类型的具体含义和触发方式

## uevent事件的创建

uevent事件包括以下三个信息：

* DEVPATH，即设备路径。该设备路径采用前面提到的kobject_get_path函数创建。
* ACTION，即uevent事件表示的行为，如设备增加，设备删除等等。
* 环境变量，类似于进程的环境变量，用于携带其他额外的信息。

uevent事件的创建由kobject_uevent_env的前半段完成，这里所说的创建是指收集向用户态发送uevent事件所需的全部信息这一过程。前面提到DEVPATH是由kobject_get_path函数从kobject树中创建，而显然ACTION应该是触发uevent事件代码段传入的参数，所以我们只需要看看环境变量是如何生成的。

从代码中直接可以看到每个uevent事件默认携带四个环境变量：ACTION，DEVPATH，SEQNUM和SUBSYSTEM。前面三个顾名思义，SUBSYSTEM比较费解。简而言之，SUBSYSTEM变量是事件目标kobject从kobject树向根结点方向找到的第一个kset的名字。

```c
        top_kobj = kobj;
        while (!top_kobj->kset && top_kobj->parent)
                top_kobj = top_kobj->parent;

        kset = top_kobj->kset;
        uevent_ops = kset->uevent_ops;

        if (uevent_ops && uevent_ops->name)
                subsystem = uevent_ops->name(kset, kobj);
        else
                subsystem = kobject_name(&kset->kobj);
```

也就是说内核默认将kset当作一个子系统中看待，kset中可以包含kset，也就是子系统中可以包含另一个子系统。还有一点比较有意思的是内核使用一个单独的uevent_seqnum作为uevent事件的序列号，使用uevent_sock_mutex互斥锁保护这一变量，这告诉我们以下两点：

1. 从用户态的视角来看，uevent事件的序列号并不是每次都加一的。这是因为uevent是namespace-aware的，并不是所有namespace都会收到。很明显这里说的是net namespace，一个net namespace下的进程并不能看到其他net namespace下的网卡，更不应该收到对应的uevent事件。
2. kobject_uevent和kobject_uevent_env函数必须在进程上下文调用，因为其内部使用了mutex。

除了标准的环境变量之外，kobject_uevent_env还接收一个env_ext参数作为其他额外的环境变量。除此之外，kset（也就是kobject所属的SUBSYSTEM）中注册的uevent_ops->uevent回调函数也可以添加自己的环境变量。函数会去掉UNBIND事件中的MODALIAS变量。

## uevent事件的发送

kobject_uevent_env前半段收集完所有所需信息之后，将发送工作交由kobject_uevent_net_broadcast进行。这里首先列举以下kobject_uevent_env中会阻止uevent事件发送的几种情况：

* 前面提到的SUBSYSTEM变量没找到，也就是触发uevent事件的kobject没有包含在任何一个kset中，这种情况视为错误，并返回EINVAL。
* kobject->suppress为true，表示这个kobject不应该触发任何uevent事件。
* kset中注册的uevent_ops->filter函数过滤掉了该kobject。

kobject_uevent_net_broadcast正如你想的一样，向除了kobject所在的net namespace发送uevent事件，如果kobject没有关联的net namespace，则向全体发送。netlink套接字所用的skb_buff按照如下方法构造：

```
                        size_t len = strlen(action_string) + strlen(devpath) + 2;
                        char *scratch;

                        skb = alloc_skb(len + env->buflen, GFP_KERNEL);

                        scratch = skb_put(skb, len);
                        sprintf(scratch, "%s@%s", action_string, devpath);

                        skb_put_data(skb, env->buf, env->buflen);
```

因此，用户态收到的数据为如下形式：

```
ACTION@DEVPATH\0
ENV1=VALUE2\0
ENV2=VALUE2\0
......
```