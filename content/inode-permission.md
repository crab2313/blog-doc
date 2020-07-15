+++
title = "inode权限检查"
date = 2018-01-07

[taxonomies]
tags = ["kernel", "filesystem"]

+++

# inode_permission函数

`inode_permission`函数用于对inode进行权限检查，我们传给其一个mask参数，这个参数是一个bitmap，主要有三个标志：

- MAY_READ：inode是否可读
- MAY_WRITE：inode是否可写
- MAY_EXEC：inode是否可执行（对于文件夹来说意味着是否可以访问文件夹下的文件或文件夹）

如果该函数的返回值为0，说明检查通过。`inode_permission`所做的检查主要分为两个部分：

1. 文件系统级别检查
2. inode级别检查

文件系统级别检查主要是检查文件系统一级的权限，这里只考虑一种情况：文件系统只读，代码如下：

```c
        if (unlikely(mask & MAY_WRITE)) {
                umode_t mode = inode->i_mode;

                /* Nobody gets write access to a read-only fs. */
                if ((sb->s_flags & MS_RDONLY) &&
                    (S_ISREG(mode) || S_ISDIR(mode) || S_ISLNK(mode)))
                        return -EROFS;
        }
```

也就是说，如果文件系统只读，并且该inode的类型是普通文件、文件夹、链接，那么这个inode是不可写的，注意这里设备文件不受影响。

# __inode_permission函数

inode级别的检查由`__inode_permission`函数进行，这个函数首先确保两个情况下inode不能写入：

1. inode的标志inode->i_flags中存在`S_IMMUTABLE`标志
2. inode的`i_uid`或者`i_gid`非法，这会使得更新mtime的时候将错误的信息写回去。

接下来通过三个不同的函数检查权限：

1. do_inode_permission
2. devcgroup_inode_permission
3. security_inode_permission

devcgroup_inode_permission涉及到了cgroup子系统，security_inode_permission涉及到LSM子系统，这里不多讲。来看`do_inode_permission`函数，这个函数首先检查inode->op是否有.permission函数指针，如果有，直接调用这个函数指针，如果没有则调用通用的检查函数`generic_permission`。事实上这段代码被运行的频率非常高，需要重点优化，所以使用inode->i_opflags中的一个`IOP_FASTPERM`标志表明i_op->permission是否为NULL，这个缓存使得下次操作不用再次检查。

```c
static inline int do_inode_permission(struct inode *inode, int mask)
{
        if (unlikely(!(inode->i_opflags & IOP_FASTPERM))) {
                if (likely(inode->i_op->permission))
                        return inode->i_op->permission(inode, mask);

                /* This gets set once for the inode lifetime */
                spin_lock(&inode->i_lock);
                inode->i_opflags |= IOP_FASTPERM;
                spin_unlock(&inode->i_lock);
        }
        return generic_permission(inode, mask);
}
```

# generic_permission函数

只要inode没有单独实现自己的inode->op->permission操作，那么就会使用默认`generic_permission`进行权限检查。这个函数首先调用`acl_permission_check`进行基本的权限检查，然后处理capabilities。`acl_permission_check`主要做了如下检查：

1. 如果fsuid与inode->i_uid相同，那么直接比较inode->i_mode中与user相关三位和mask是否匹配
2. 如果fsuid与inode->i_uid不相同，并且inode所在的文件系统支持POSIX_ACL，那么检查POSIX_ACL，如果检索有结果（无论是允许还是阻止），都会直接返回该结果
3. 如果fsuid与inode->i_uid不相同，并且POSIX_ACL没有检索结果（包括inode所在文件系统不支持POSIX ACL的情况），但是当前进程属于inode->i_gid代表的group，那么比较inode->i_mode中与group相关的三位和mask是否匹配
4. 上述条件都不满足的情况下，比较inode->i_mode中与other相关的三位和mask是否匹配

当`acl_permission_check`认定当前进程没有权限对inode进行由mask代表的操作时，`generic_permission`才会继续进行capabilities的检查。涉及到inode权限的capabilities有两个：CAP_DAC_OVERRIDE和CAP_DAC_READ_SEARCH。CAP_DAC_OVERRIDE可以让进程（几乎）无视文件设置的三组权限位，而CAP_DAC_READ_SEARCH赋予进程搜索文件夹和读取文件的权限。由于文件夹的三组权限位与普通文件表示的意义是不一样的，所以这两个capability的行为也是不同的。

```c
        if (S_ISDIR(inode->i_mode)) {
                /* DACs are overridable for directories */
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
                        return 0;
                if (!(mask & MAY_WRITE))
                        if (capable_wrt_inode_uidgid(inode,
                                                     CAP_DAC_READ_SEARCH)
)
                                return 0;
                return -EACCES;
        }
        /*
         * Read/write DACs are always overridable.
         * Executable DACs are overridable when there is
         * at least one exec bit set.
         */
        if (!(mask & MAY_EXEC) || (inode->i_mode & S_IXUGO))
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_OVERRIDE))
                        return 0;

        /*
         * Searching includes executable on directories, else just read.
         */
        mask &= MAY_READ | MAY_WRITE | MAY_EXEC;
        if (mask == MAY_READ)
                if (capable_wrt_inode_uidgid(inode, CAP_DAC_READ_SEARCH))
                        return 0;
```

对于文件夹：

- CAP_DAC_OVERRIDE直接pass
- CAP_DAC_READ_SEARCH允许MAY_READ和MAY_EXEC

对于普通文件：

- CAP_DAC_OVERRIDE对待MAY_EXEC是不同的：如果mask中出现了MAY_EXEC，那么只有三组权限位中至少有一个X位被设置的时候才pass，其余情况全pass。
- CAP_DAC_READ_SEARCH只允许MAY_READ