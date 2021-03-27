+++
title = "虚拟文件系统实现分析"
date = 2020-12-09
draft = true


tags = ["kernel", "filesystem"]

+++

本文记录开发过程中对文件系统中的一些分析。

主要分析点：

* inode实现分析，以及如果通过文件路径获取一个inode。