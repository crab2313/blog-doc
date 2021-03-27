+++
title = "egdb-server(名称暂定)项目"
date = 2020-08-31
draft = true


tags = ["debug", "project"]

+++

本文记录egdb-server项目的一些设计与思考。

## 项目目标

使用rust实现一个通用的gdb server实现，向外提供通用的接口，主要满足cloud-hypervisor的需求，使得我可以使用cloud-hypervisor对内核进行调试。除此之外，尝试提供更加高级的功能，如进程、内存的监控。当前的主要平台为x86以及ARM64，日后在RISC-V的调试标准定稿之后，可以对其提供支持。

## 主要计划

* 首先实现简单的GDB Server，对其wire protocol有一定理解
* 在此之上设计简单的嵌入模块，考虑线程以及async io模型
* 研究x86以及ARM64下的调试指令集，尝试借用硬件辅助功能提升效率
* 研究内核的vmcore结构？

