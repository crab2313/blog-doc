+++
title = "Linux内核中ARP协议的实现"
date = 2021-04-08
tags = ["kernel", "net"]

+++

本文对内核中ARP协议相关的事件进行分析。本文是原先分析文档的整理，后续会进行相应复习与补足。

## 初始化

```c
void __init arp_init(void)
{
        neigh_table_init(NEIGH_ARP_TABLE, &arp_tbl);

        dev_add_pack(&arp_packet_type);
        arp_proc_init();
#ifdef CONFIG_SYSCTL
        neigh_sysctl_register(NULL, &arp_tbl.parms, NULL);
#endif
        register_netdevice_notifier(&arp_netdev_notifier);
}
```

首先是注册了一个neigh_table，这个是ARP的entry cache，ARP解析所有的缓存都会保存在这里。由于ARP协议是与IP协议同级，都是跑在Ethernet之上，所以需要使用dev_add_pack注册一个ethernet包的协议，如下：

```c
static struct packet_type arp_packet_type __read_mostly = {
        .type = cpu_to_be16(ETH_P_ARP),
        .func = arp_rcv,
};
```

因此，L2每次收到ARP包的时候，会调用`arp_rcv`函数处理。之后ARP协议注册了一个/proc/net/arp文件，如下：

```c
static int __init arp_proc_init(void)
{
        return register_pernet_subsys(&arp_net_ops);
}

static struct pernet_operations arp_net_ops = {
        .init = arp_net_init,
        .exit = arp_net_exit,
};

static int __net_init arp_net_init(struct net *net)
{
        if (!proc_create("arp", S_IRUGO, net->proc_net, &arp_seq_fops))
                return -ENOMEM;
        return 0;
}

static void __net_exit arp_net_exit(struct net *net)
{
        remove_proc_entry("arp", net->proc_net);
}
```

可以看出作为一个非常基础的协议，ARP协议是namespace-aware的，也就是说对于每一个网络namespace，arp协议会注册该文件。直接读取这个文件我们可以得到本机的ARP缓存，与直接运行arp命令效果是一样的。随后，如果系统开启的SYSCTL的支持，ARP协议初始化函数会注册对应的/proc/sys/net/neigh/文件夹，该文件夹下每个的子文件夹都代表一个网络接口，通过网络接口文件夹下的文件可以在运行时调整ARP协议的各种行为。

最后，arp_init函数中调用`register_netdevice_notifier`注册一了一个当网络设备发生变动的时候就会调用的回调函数。

## ARP协议栈实现的neigh_ops

由于我看的内核是4.12版本的，所以arp_broken_ops已经没有了，这说明新内核中已经完全将旧的驱动完全移植到了新的neighbour子系统中，所以不需要再用arp_broken_ops提供的兼容层。新的内核中只有三个ARP的ops，如下表：

|       名称        |              用途              |
| :-------------: | :--------------------------: |
| arp_generic_ops |        最通用的neigh_ops         |
|   arp_hh_ops    |    用于net_device实现了L2缓存的情况    |
| arp_direct_ops  | 用于net_device不需要使用L2Header的情况 |

先将代码贴出来再分析：

```c
static const struct neigh_ops arp_generic_ops = {
        .family =               AF_INET,
        .solicit =              arp_solicit,
        .error_report =         arp_error_report,
        .output =               neigh_resolve_output,
        .connected_output =     neigh_connected_output,
};

static const struct neigh_ops arp_hh_ops = {
        .family =               AF_INET,
        .solicit =              arp_solicit,
        .error_report =         arp_error_report,
        .output =               neigh_resolve_output,
        .connected_output =     neigh_resolve_output,
};

static const struct neigh_ops arp_direct_ops = {
        .family =               AF_INET,
        .output =               neigh_direct_output,
        .connected_output =     neigh_direct_output,
};
```

先看arp_direct_ops，这个最特别。首先可以看到它没有.solicit和.error_report，这个是显而易见的，因为使用这个neigh_ops的neighbour entry不需要进行ARP解析。所以不管处于什么状态下，对应的output函数都是neigh_direct_output，这将让对应的直接交给设备发送出去。

arp_generic_ops和arp_hh_ops的唯一区别是它的.connected_output设置为了neigh_connected_output，这是因为如果net_device的驱动程序自己实现了L2缓存的话，neigh_resolve_output函数中调用的dev_hard_header函数可以从该缓存中获取对应的L2头部。

## arp_constructor函数

neighbour子系统在使用neigh_create创建一个struct neighbour的时候会调用保存在neigh_table中的.constructor函数指针初始化这个对象。对应于ARP协议，则为arp_constructor函数。

这个函数首先设置好这个entry的地址类型，如下：

```c
        __be32 addr = *(__be32 *)neigh->primary_key;
        neigh->type = inet_addr_type_dev_table(dev_net(dev), dev, addr);
```

随后开始设置其对应的初始化状态。如果网络接口设备没有设置dev->header_ops指针，亦即网络接口设备的驱动不会增加L2的Header，则将neigh->nud_state设置为NUB_NOARP，并将neigh->ops设置为arp_direct_ops，neigh->output设置为neigh_direct_output。

接下来处理几个特殊的IPv4地址类型：组播，广播，本地回环，并处理net_device的flag标记了NOARP的情况：

```c
                if (neigh->type == RTN_MULTICAST) {
                        neigh->nud_state = NUD_NOARP;
                        arp_mc_map(addr, neigh->ha, dev, 1);
                } else if (dev->flags & (IFF_NOARP | IFF_LOOPBACK)) {
                        neigh->nud_state = NUD_NOARP;
                        memcpy(neigh->ha, dev->dev_addr, dev->addr_len);
                } else if (neigh->type == RTN_BROADCAST ||
                           (dev->flags & IFF_POINTOPOINT)) {
                        neigh->nud_state = NUD_NOARP;
                        memcpy(neigh->ha, dev->broadcast, dev->addr_len);
                }
```

最后根据网络设备驱动时候实现了L2地址缓存来确定这个struct neigh到底使用哪个ops:

```C
                if (dev->header_ops->cache)
                        neigh->ops = &arp_hh_ops;
                else
                        neigh->ops = &arp_generic_ops;

                if (neigh->nud_state & NUD_VALID)
                        neigh->output = neigh->ops->connected_output;
                else
                        neigh->output = neigh->ops->output;
```

顺带一提，__neigh_alloc函数中将neigh->nud_state设置成了NUD_NONE，并将neigh->output设置成了neigh_blackhole。

## arp_rcv函数

来看ARP协议是如何处理接收到的ARP包的。首先是做一些简单的检查，确定该ARP包是系统应该处理的：

```c
        /* do not tweak dropwatch on an ARP we will ignore */
        if (dev->flags & IFF_NOARP ||
            skb->pkt_type == PACKET_OTHERHOST ||
            skb->pkt_type == PACKET_LOOPBACK)
                goto consumeskb;
```

可以看到，只要网络设备设置了NOARP，或者该sk_buff的pkg_type字段是PACKET_OTHERHOST | PACKET_LOOPBACK，ARP协议栈是不会处理的，而是将这个包简单丢弃，并不做任何记录。接下来是对这个sk_buff做一些通用的处理：

```c
        skb = skb_share_check(skb, GFP_ATOMIC);
        if (!skb)
                goto out_of_mem;

        /* ARP header, plus 2 device addresses, plus 2 IP addresses.  */
        if (!pskb_may_pull(skb, arp_hdr_len(dev)))
                goto freeskb;
        arp = arp_hdr(skb);
```

这样的处理甚至可以说是标准处理，首先通过skb_share_check确定这个包没有被其他人引用，如果有，那么就clone一个出来。这样做是因为后面的pskb_may_pull有可能改变sk_buff中的指针，导致不一致的情况出现。`pskb_may_pull`函数确保data与tail指针之间数据的长度至少有arp_hdr_len(dev)这么长。这里提一句，这么令人费解的操作是因为一个sk_buff的数据可以分为三段，第一段称为线性区（即data与tail指针之间的数据），第二段称为ummaped page区域，第三段为fragment list。我们可以看到后面直接使用arp_hdr(skb)获取了ARP Header，而arp_hdr的代码如下：

```c
static inline struct arphdr *arp_hdr(const struct sk_buff *skb)
{
        return (struct arphdr *)skb_network_header(skb);
}
static inline unsigned char *skb_network_header(const struct sk_buff *skb)
{
        return skb->head + skb->network_header;
}
```

可以看到是直接通过指针进行类型转换的，因此如果线性区中的数据不够长的话，得到的arp_hdr返回的struct arphdr指向的数据就是不完整的，尾巴上少了一截。

随后开始检查ARP包中的数据是否合法：

```c
        if (arp->ar_hln != dev->addr_len || arp->ar_pln != 4)
                goto freeskb;
```

这里检查了ARP包中Hardware Length是否与接收到这个包的网络接口的链路层地址是否一致。由于ARP协议是和IP协议绑定的，所以这里直接检查协议地址长度的长度是否为4（即IPv4的地址长度）。最后我们看到了一个新的Netfilter:

```c
        memset(NEIGH_CB(skb), 0, sizeof(struct neighbour_cb));

        return NF_HOOK(NFPROTO_ARP, NF_ARP_IN,
                       dev_net(dev), NULL, skb, dev, NULL,
                       arp_process);
```

## arp_process函数

arp_process函数首先检查了ARP Header中的HW字段是否与设备相匹配。随后可以看到Linux系统对于接收到的ARP包***只处理ARPOP_REPLY和ARPOP_REQUEST***。随后将ARP协议的数据从ARP包中取出来，得到如下几个变量：

|  名称  |                 注释                 |
| :--: | :--------------------------------: |
| sha  | source hardware address，即发送方的MAC地址 |
| tha  | target hardware address，即发送方的MAC地址 |
| sip  |        source ip， 即发送方的IP地址        |
| tip  |        target ip，即接收方的IP地址         |

接下来可以看到ARP协议栈检查了tip的值，扔掉了目标地址为组播，广播和本地的ARP包，因为这些IP地址压根是不需要进行ARP解析的，所以处于安全性和性能的考虑直接将这些包丢掉。

### 对ARPOP_REQUEST的处理

这里Linux内核首先处理了一个特殊情况，`sip == 0`,其实就是检查sip变量的值是否为0.0.0.0。对于某些DHCP服务器或者客户端来说，可以使用以源IP地址为0.0.0.0的方式发送ARP Request，以此检测特定的IP地址已经被占用。之所以称之为特殊情况，是因为***Linux内核默认不响应一个不在它路由表中的IP地址发来的ARPOP_REQUEST***。

```c
        /* Special case: IPv4 duplicate address detection packet (RFC2131) */
        if (sip == 0) {
                if (arp->ar_op == htons(ARPOP_REQUEST) &&
                    inet_addr_type_dev_table(net, dev, tip) == RTN_LOCAL &&
                    !arp_ignore(in_dev, sip, tip))
                        arp_send_dst(ARPOP_REPLY, ETH_P_ARP, sip, dev, tip,
                                     sha, dev->dev_addr, sha, reply_dst);
                goto out_consume_skb;
        }
```

可以从代码中发现只有如下条件都满足的情况下，Linux内核才会响应一个ARPOP_REQUEST：

1. 内核知道如何与sip指向的IP地址的主机进行通信
2. tip是本机的一个IP地址，或者本机对该IP地址的ARP解析进行了代理
3. 本机的管理员没有显式的阻止对该ARPOP_REQUEST的响应

### 对ARPOP_REPLY的处理

TODO THIS

## arp_solicit函数

所有arp_ops中的.solicit指针都是指向arp_solicit函数的。为了读懂这个函数，首先要明白arp_announce的概念和用途。Linux内核中网络协议栈有一个比较明显的特点：虽然系统管理员在设置IP地址的时候看起来像是为特定的网络接口设置的，但是***在Linux内核眼中，IP地址是属于整个Host的***。因此，在早期的Linux内核中，会出现将本应该属于特定subnet中的ARP请求发送到属于其他subnet系统的网络接口中。因此，arp_announce选项就是为了解决这个问题而出现的。

arp_annouce分为三个级别，如下表：

|  级别  |           说明           |
| :--: | :--------------------: |
|  0   |   默认可以使用所有接口上的所有IP地址   |
|  1   | SADDR必须与需要解析的地址在同一个子网中 |
|  2   |   强制使用在目标主机子网上的IP地址    |

arp_solicit函数的声明如下：

```c
static void arp_solicit(struct neighbour *neigh, struct sk_buff *skb)
```

其中， neigh参数是需要进行解析的tbl_entry，skb参数是触发这次解析的包。arp_solicit函数最重要的任务就是确定ARP报文中对应字段的值。可以看到该函数开头通过arp_announce确定ARP报文的saddr：

```c
        switch (IN_DEV_ARP_ANNOUNCE(in_dev)) {
        default:
        case 0:         /* By default announce any local IP */
                if (skb && inet_addr_type_dev_table(dev_net(dev), dev,
                                          ip_hdr(skb)->saddr) == RTN_LOCAL)
                        saddr = ip_hdr(skb)->saddr;
                break;
        case 1:         /* Restrict announcements of saddr in same subnet */
                if (!skb)
                        break;
                saddr = ip_hdr(skb)->saddr;
                if (inet_addr_type_dev_table(dev_net(dev), dev,
                                             saddr) == RTN_LOCAL) {
                        /* saddr should be known to target */
                        if (inet_addr_onlink(in_dev, target, saddr))
                                break;
                }
                saddr = 0;
                break;
        case 2:         /* Avoid secondary IPs, get a primary/preferred one */
                break;
        }
        if (!saddr)
                saddr = inet_select_addr(dev, target, RT_SCOPE_LINK);
```

case 0很好懂，case 1比case 0多了一个检查，即确定target与saddr是否直接相连，当前两个case不满足的时候，默认直接就开始使用arp_annouce=2的配置。
