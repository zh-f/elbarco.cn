---
title: 初识Virtio Balloon
date: 2018-07-16 11:22:07
tags: [Virtio, KVM, Virtualization]
---

> 原文地址：[Virtio balloon](https://rwmj.wordpress.com/2010/07/17/virtio-balloon/)
原创翻译，转载请注明出处。

本文主要介绍了什么是[KVM virtio balloon driver](https://github.com/torvalds/linux/blob/e241e3f2bf975788a1b70dff2eb5180ca395b28e/drivers/virtio/virtio_balloon.c)。<!-- more -->

首先，如果你从没听过这个概念，那什么是balloon driver呢？它是一种给予guest实例或者从guest实例中获取RAM的方法。（理论上来讲），如果你的guest实例需要更多的RAM，你可以通过balloon driver给他分配更多的内存，或者如果宿主机需要在guest实例中取走一下内存，balloon driver也可以做到。这些操作的执行**不需要**暂停或者重启guest实例。

virtio_balloon是一个在guest实例中的内核驱动。这个driver表现的像一种奇怪的进程，要么扩展自己的内存使用，要么压缩自己的内存使用接近什么也没有，如下图所示：
![balloon-expanded](http://7xrgsx.com1.z0.glb.clouddn.com/balloon-expanded.png)

![balloon-shrink](http://7xrgsx.com1.z0.glb.clouddn.com/balloon-shrink.png)

当balloon driver扩张时，运行在guest实例中的应用会突然少了很多可用内存，然后guest实例会像没有多少内存时做的那样，交换内存并启动OOM killer（balloon本身是不可交换并且不会被杀掉的）。

那么这个“浪费”内存的内核驱动程序有什么意义呢？有两点——
第一，驱动通过virtio通道与宿主机通讯，宿主机给其下发指令，比如扩展到指定的大小、现在开始缩小。guest实例配合操作，但是并不直接控制balloon。
第二，balloon中的内存也从guest实例中取消映射，并传回宿主机，因此宿主机可以将这部分内存传递给其他的guest实例使用。看起来就像guest虚机的内存缺失了一块一样：

![balloon-chunk](http://7xrgsx.com1.z0.glb.clouddn.com/balloon-chunk.png)

Libvirt有两个配置项，`currentMemory`和`maxMemory`（详见[Memory Allocation](https://libvirt.org/formatdomain.html#elementsMemoryAllocation)）：

![balloon-labels](http://7xrgsx.com1.z0.glb.clouddn.com/balloon-labels.png)

`maxMemory`（或`<memory>`）是在guest实例启动阶段分配的内存。KVM和Xen的guest虚机目前（译者注：原文发布时间是2010-07-17 14:33）无法找过这个内存上限。`currentMemory`控制了请求分配给guest实例上应用程序的内存。balloon填充剩余的内存，并且把这部分内存还给宿主机，用于宿主机的在其他地方使用。

该项配置可以手动调整，或者通过编辑XML文件，或者通过`virsh setmem`命令。
