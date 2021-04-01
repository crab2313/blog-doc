+++
title = "DRM框架分析（一）：简介"
date = 2021-03-28
tags = ["kernel", "drm"]

+++

真的没有想到真的有人能看到这篇东西，所以我准备抽出时间把这个东西整理完。本文是我阅读内核DRM子系统的一些随笔，其目的是记录我对内核代码的阅读过程和自己的一些理解，希望景嘉微的大佬们多多指正。

接触linux桌面比较多的人一定会对DRM这个名字比较熟悉。网上也有大把资料在解释DRM到底是什么。DRM主要分为KMS与Render两大部分，本文实质上是在分析DRM中KMS相关的框架实现。而Render相关的API是特定于驱动的，内核并不为用户态提供一个通用的IOCTL接口。如果后续时间充足，我将会分析一下VC4的Render API，以及相应的用户态实现。从功能上讲，KMS负责搭建显示控制器的pipeline，并控制显示硬件将图像缓冲区scanout到屏幕上，而如何加速生成framebuffer中的内容则是3D引擎（即Render API）负责的事情。

对于KMS，有如下要素：

* 显存管理器
* modesetting API
* 显示API

在研究初期，研究案例最好硬件无关，且比较容易懂原理，且能够正常运行，可以参考virtio-gpu和QXL虚拟显卡。等到对内核DRM子系统有一定的理解后，可以分析简单的显卡硬件，如VC4（树莓派3B的显卡）。

## 阅读路径

需要看的东西有点多，甚至说比较乱。先列举一下：

内核中的内容：

* DRM驱动通用代码：包括GEM，KMS
* AMD显卡相关的代码：AMDGPU，RADEON，AMDKFD（通用计算ROCM框架内核驱动）

用户态代码：

* MESA： OpenGL state tracker， gallium 3D， vulkan， egl（重点），gbm
* libdrm：基本为内核提供的IOCTL的wrapper

我想要重点理解的部分：

* context到底是什么？如OpenGL和egl创建的context
* mesa的架构
* wayland渲染的基本原理
* DRI到底由什么构成

这里我觉得还是先从MESA这里着手，毕竟内核驱动缺少文档，且我对接口层到底怎么用还是不是很熟悉。看一下简单的DUMB驱动如何实现也是一个理解KMS比较好的方法。

[1]: https://www.kernel.org/doc/html/latest/gpu/drm-kms-helpers.html#atomic-modeset-helper-functions-reference	"Atomic Modeset Helper Functions Reference"
[2]: https://lwn.net/Articles/653071/	"Atomic mode setting design overview, part 1"
[3]: https://lwn.net/Articles/653466/	"Atomic mode setting design overview, part 2"
