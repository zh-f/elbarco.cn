---
title: Nova架构体系概览
date: 2018-01-16 11:11:07
tags: [OpenStack, Nova]
---

## What's Nova

我们首先来看一下OpenStack的逻辑架构图：<!--more-->
![](http://bop-to.top/osog_0001.png)

接触过云计算，接触过OpenStack的童鞋都会有所了解，IaaS中最重要的就是计算、存储和网络。Nova，作为OpenStack核心项目，承担起了提供计算资源的重任，即，为用户提供了计算实例，这些实例又称为虚拟机。从上面的逻辑架构图中也可以看到，有了虚拟机之后，才可以外接存储、网络，亦或是有类似Trove（OpenStack中Database-as-a-Service的项目，详见[>>>传送门](https://www.openstack.org/software/releases/ocata/components/trove)）这种在虚机中运行数据库服务的PaaS项目。当然，虚机也需要像认证服务（Keystone）、镜像（Glance）、网络（Neutron）、存储（Cinder、Swift）这些项目的支持，可谓是你中有我，我中有你。

目前在OpenStack Nova项目的页面（详见[>>>传送门](https://www.openstack.org/software/releases/ocata/components/nova)）显示的生产环境的应用率高达95%，可以说是很“强势”。

本文就作为Nova学习系列的开篇文章，先熟悉一下Nova架构体系及代码结构。

## Components 

Nova项目也是有好几个组件构成，组件的关系架构图如下所示，其中网络模块一个是Nova-networking，一个是Neutron，当然现在大部分使用的都是Neutron，所以我们只关注第二张图就OK了：
![](http://bop-to.top/architecture.svg)

可以看到，在Nova中，有这么几个主要的服务：

* DB：用于数据存储的基础设施数据库
* API: 即nova-api服务，通过oslo.messaging队列或者HTTP，接收响应终端用户的计算服务API请求或者与其他组件进行通讯
* Scheduler: 即nova-scheduler服务，用于调度每台实例具体落到哪个计算服务节点上
* Compute: 即nova-compute服务，管理hypervisor与虚机的通讯，通过虚拟机管理程序API对虚拟机实例进行创建、终止等操作的一个工作守护进程
* Conductor: 即nova-conductor服务，处理需要协同合作的请求，比如创建实例和调整实例等操作；同时还扮演了数据库代理的角色或者是处理对象转换

在这个图中，值得注意的一点是Nova的几个主要服务组件之间，是通过oslo.messaging进行RPC调用，与外部服务之间通过HTTP的方式、RESTFul接口进行通讯和交互。

在Compute中，有一个“Hypervisor”，这又是什么呢？

我们讲，OpenStack其实是一个云管平台，即其本身不提供虚拟化功能，还是要依赖于操作系统底层的虚拟化技术，其中Hypervisor是虚拟化技术的核心。它是一种运行在物理服务器和操作系统之间的中间软件层，可允许多个操作系统和应用共享一套基础物理硬件，因此也可以看作是虚拟环境中的“元”操作系统，它可以协调访问服务器上的所有物理设备和虚拟机，也叫虚拟机监视器（Virtual Machine Monitor）——
>In computing, a hypervisor, also called virtual machine monitor (VMM), is a piece of software/hardware platform-virtualization software that allows multiple operating systems to run on a host computer concurrently.

目前常见的Hypervisor有QEMU、KVM、XEN、VMware等，其中，KVM是集成到Linux内核的Hypervisor，是X86架构且硬件支持虚拟化技术（Intel VT或AMD-V）的Linux的全虚拟化解决方案。它是Linux的一个很小的模块，利用Linux做大量的事，如任务调度、内存管理与硬件设备交互等。最为热门，也最为常用。

此外，需要提一下qemu-kvm——
> QEMU将KVM整合进来，通过ioctl调用/dev/kvm接口，将有关CPU指令的部分交由内核模块来做。KVM负责CPU虚拟化+内存虚拟化，实现了CPU和内存的虚拟化，但KVM不能模拟其他设备。QEMU模拟IO设备（网卡，磁盘等），KVM加上QEMU之后就能实现真正意义上服务器虚拟化。因为用到了上面两个东西，所以称之为qemu-kvm，来张图：
> ![](http://bop-to.top/kvm_archi_base_oh9pnk.png)

在KVM这一层之上，是libvirt，它提供统一、稳定、开放源码的对各种虚拟机进行管理的工具（守护进程libvirtd、默认命令行管理工具virsh）和应用程序接口（API）。一些常用的虚拟机管理工具（如virsh、virt-install、virt-manager等）和云计算框架平台（如OpenStack等）都在底层使用libvirt的应用程序接口。

在Nova的Compute服务中，通过不同的驱动来支持多种Hypervisor，与各种Hypervisor驱动的关系可以用下面的一张图来表示：
![](http://bop-to.top/nova-compute-drivers.jpg)

## Instance Life Cycle Management Process

在实例的生命周期管理流程中，常见的一些操作如下：

* 创建：nova boot
	* boot from image
	* boot from volume
* 重启：nova reboot
	* 软重启，默认情况
	* 硬重启：nova reboot --hard，具体不同还需要用代码说话
* 启动：nova start
* 停止：nova stop
* 挂起：nova suspend
* 暂停：nova pause，pause与suspend的区别在于pause将instance的运行状态保存在计算节点的内存中，而suspend保存在磁盘上。pause的优点在于恢复的速度比suspend快，缺点是如果计算节点重启，内存数据丢失，则无法resume。suspend就不存在该问题
* 恢复：nova resume
* 调整实例：nova resize
* 迁移实例：
	* nova live-migration
	* nova migrate，代码与nova resize相同，如果在resize时未提供flavor id，则仅migrate实例
* 重建：nova rebuild
* 快照：nova image-create，对运行的虚机创建一个快照镜像，直接上传到Glance中，可用于恢复主机或以此镜像为模板创建新的主机
* 备份：nova backup，通过创建一个`backup`类型的快照来备份主机
* 删除：nova delete，立即关闭主机并删除实例

后续需要对上面这些主要流程进行梳理，包括上面描述的几个命令的不同或者相似之处，我们让代码来说话。

## Reference

[1]. [虚拟化类型](https://huangwei.me/wiki/tech_cloud_kvm_qemu_libvirt_openstack.html)
[2]. [Under the Hood with Nova, Libvirt and KVM](https://www.openstack.org/assets/presentation-media/OSSummitAtlanta2014-NovaLibvirtKVM2.pdf)
[3]. [OpenStack系列--Nova](https://zhangchenchen.github.io/2016/08/22/openstack-nova/)
[4]. [OpenStack Compute(Nova)](https://docs.openstack.org/nova/latest/)




