+++
title = "EVM内核框架分析"
date = 2020-11-30
draft = true


tags = ["kernel", "evm"]

+++

本文意在分析EVM实现机制及用法。由于EVM内容比较少，本文应该会额外分析一下内核key service的实现。

从架构上看，EVM依附于IMA，实际上并不能独立进行工作。一旦IMA被关闭，那么EVM本身就没有被任何人调用。对EVM的分析从EVM的唯一内核命令行参数`evm=fix`开始进行。一旦内核检测到该命令行参数，就会将`evm_fixmode`设置为1。事实上，只有`evm_verify_current_integrity`函数中才会检测这个flag，如下：

```c
static enum integrity_status evm_verify_current_integrity(struct dentry *dentry)
{
    struct inode *inode = d_backing_inode(dentry);

    if (!evm_key_loaded() || !S_ISREG(inode->i_mode) || evm_fixmode)
        return 0;
    return evm_verify_hmac(dentry, NULL, NULL, 0, NULL);
}
```

这说明EVM的fix模式并不是我想象中那个原理，`evm=fix`能够直接放行对inode元数据完整性的检测。根据目前所见，合理的推测是：对安全扩展属性的更新会触发EVM的更新，更新之前会对EVM进行校验。由于`evm=fix`会强制通过EVM的校验，所以EVM一定会被更新，这就实现了fix机制。

## 计算security.evm

计算`security.evm`的值的函数有好几个变体，但最终都调用了`evm_calc_hmac_or_hash`函数。函数原型如下：

```c
static int evm_calc_hmac_or_hash(struct dentry *dentry,
                 const char *req_xattr_name,
                 const char *req_xattr_value,
                 size_t req_xattr_value_len,
                 uint8_t type, struct evm_digest *data);
```

一定要注意到它传入了一个安全扩展属性，事实上计算HMAC的按照我的理解应该是使用EVM保护的全部安全扩展属性的。函数也确实是这么做的，也就是：**除了传入的req_xattr_value**，其它的扩展安全属性都从磁盘中读取。注意函数最后调用`hmac_add_misc`将一段额外的数据用于extend哈希值，如下：

```c
    struct h_misc {
        unsigned long ino;
        __u32 generation;
        uid_t uid;
        gid_t gid;
        umode_t mode;
    } hmac_misc;
```

```c
    crypto_shash_update(desc, (const u8 *)&hmac_misc, sizeof(hmac_misc));
    if ((evm_hmac_attrs & EVM_ATTR_FSUUID) &&
        type != EVM_XATTR_PORTABLE_DIGSIG)
        crypto_shash_update(desc, (u8 *)&inode->i_sb->s_uuid, UUID_SIZE);
    crypto_shash_final(desc, digest);
```

## 更新security.evm

EVM通过各类钩子函数与内核其它部分进行集成。例如，更新文件inode的xattr的时候，EVM通过插入在`security_inode_setxattr`函数内的`evm_inode_setxattr`钩子函数检验原有inode的安全扩展属性是否完整。何时需要更新`security.evm`?只有受到EVM保护的inode元数据被修改时`security.evm`才需要一同更新，前面分析到，这包括：

* `evm_config_xattrnames`中保存的扩展安全属性名称
* `mode`，`gid`，`uid`等标准属性

这就要求EVM hook特定的操作，包括`setattr`，`setxattr`和`removexattr`。而具体的更新操作由`evm_update_evmxattr`实现。函数内部仅仅是计算`security.evm`的值，然后更新`security.evm`而已。

## 验证security.evm

### evm_verifyxattr

原先在分析IMA的时候，看到`ima_appraise_mesasurement`函数中调用了`evm_verifyxattr`校验`security.ima`扩展属性的完整性，我们从这个函数开始入手。从`ima_appraise_measurement`可以看到其调用`evm_verifyxattr`的传入参数是`security.ima`与其对应的值。注意这里的一个疑点是为什么通过HMAC可以校验单个的安全扩展属性，而不是需要一起校验。

函数开头首先检查EVM的密钥是否已经加载（即EVM是否已经启动）以及所校验的安全扩展属性是否在EVM保护的范围之内。如果传入的iint cache为空，则进行查找。最后调用`evm_verify_hmac`函数真正执行操作。后续分析重点为`evm_verify_hmac`的具体实现。

`evm_verify_hmac`的第一个操作就是检查`iint cache`，如果对应的文件已经完成了完整性检查，则直接返回：

```c
    if (iint && (iint->evm_status == INTEGRITY_PASS ||
             iint->evm_status == INTEGRITY_PASS_IMMUTABLE))
        return iint->evm_status;
```

随后函数从inode中读出`security.evm`属性的值，并检查其开头的第一个字节。我们知道`security.ima`和`security.evm`中的属性值有如下形式：

```c
struct evm_ima_xattr_data {
    u8 type;
    u8 data[];
} __packed;
```

即第一个字节表明了后续的数据类型，对于EVM，有：

* EVM_XATTR_HMAC
* EVM_IMA_XATTR_DIGSIG
* EVM_XATTR_PORTABLE_DIGSIG

对于存储了`EVM_XATTR_HMAC`的inode，函数简单调用`evm_calc_hmac`计算出对应的哈希值，然后进行比较。

## EVM初始化

EVM的初始化非常简单，这里提一嘴。EVM的初始化依赖`init_evm`函数，并由`late_initcall`注册到内核中。函数的工作非常简单：初始化`evm_config_xattrnames`列表与`securityfs`中的文件。

EVM子系统中通过静态数组的方式注册了一个列表，包含了EVM应该进行完整性保护的安全扩展属性：

```c
static struct xattr_list evm_config_default_xattrnames[] = {
#ifdef CONFIG_SECURITY_SELINUX
    {.name = XATTR_NAME_SELINUX},
#endif
#ifdef CONFIG_SECURITY_SMACK
    {.name = XATTR_NAME_SMACK},
#ifdef CONFIG_EVM_EXTRA_SMACK_XATTRS
    {.name = XATTR_NAME_SMACKEXEC},
    {.name = XATTR_NAME_SMACKTRANSMUTE},
    {.name = XATTR_NAME_SMACKMMAP},
#endif
#endif
#ifdef CONFIG_SECURITY_APPARMOR
    {.name = XATTR_NAME_APPARMOR},
#endif
#ifdef CONFIG_IMA_APPRAISE
    {.name = XATTR_NAME_IMA},
#endif
    {.name = XATTR_NAME_CAPS},
};
```

EVM初始化时，会将其添加到`evm_config_xattrnames`链表中。除此之外就是` /sys/kernel/security/evm`的注册。早期的内核中，只存在`/sys/kernel/security/evm`文件，现在变成了一个文件夹，并添加了新的用户态接口。具体的接口描述可以从`Documentation/ABI/testing/evm`文件中获取。

## 基于签名的EVM

我决定将这里单独写，因为它是横跨整个IMA & EVM架构的独立实现，不做整体分析是没有任何结果的。