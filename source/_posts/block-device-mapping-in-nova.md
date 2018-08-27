---
title: Nova中的块设备映射
date: 2018-08-25 13:54:51
tags: [OpenStack, Nova]
---

> 原文地址：[Block Device Mapping](https://docs.openstack.org/nova/queens/user/block-device-mapping.html)
> 原创翻译，转载请注明出处。
<!-- more -->

## 一、概述

Nova中存在一个块设备的概念，这些块设备用于暴露给实例使用。实例的块设备可以有多种，租户或者用户使用哪种块设备取决于特定的部署已经设置的使用限制。块设备映射（Block device mapping）用于管理实例所有的块设备和保存数据。

当我们讨论块设备时，我们通常会涉及下列事情：
* API或者CLI命令行层面，在实例boot请求中指定块设备的请求结构或者syntax；
* Nova内部用于记录、持久化块设备数据的数据结构，最终记录会写入到`block_device_mapping`表中。然而在Nova内部，在表示同一数据时，却有几种略微不同的数据格式。除了代码中的`BlockDeviceMapping`对象，我们还有：
  * API格式，见`BlockDeviceDict`类，来处理API层面BDM的校验和格式转换，姑且称之为API BDMs
  * virt driver格式，由nova.virt.block_device中的类来定义，比如`DriverBlockDevice`、`DriverVolumeBlockDevice`等。这些类，除了暴露不同的格式之外，同时提供了处理不同类型的块设备的一些功能函数，比如挂在卷需要同是和cinder和virt driver的代码交互，下文中称之为Driver BDMs

## 二、数据格式及其历史

Nova早期的代码中的block device mapping，基本上是仿照了EC2 API中的数据结构。在Havana版本中，block device mapping的代码有了改进和提高，比如在API中暴露额外的详情和特性。为此，V2 API中增加了一个新的扩展，称之为`BlockDeviceMappingV2Boot`，实际上是在实例的boot API请求中，增加了字段`block_device_mapping_v2`。

### Block device mapping V1（又称之为 legacy）

目前Nova代码中使用和存储数据使用的是新的数据结构，但是为了处理会用遗留格式的请求，仍然需要处理v1格式，代码的转换在`nova.block_device`模块。在V1中，使用device name作为key，并且仅接受：

* Cinder卷或者快照的UUID
* Type，类型，用于区分Cinder卷和快照。
* Size，大小，可选项
* delete_on_termination标志位，可选项

### 间奏曲-块设备名称的问题

使用设备名称作为每个实例的主要的识别号，并把它们暴露给API，对于Nova来说，是有问题的，主要是因为Nova支持的几种hypervisor及其自己的drivers不能保证guest OS分配的设备就是用户在Nova中请求的设备。另外，在公共的NovaAPI中暴露这样的详情，显然也是不理想的。

解决这个问题，方案是允许用户不指定块设备名称，从而交给Nova（在virt driver的帮助下）来决定设备名称。

此外，指定设备名称常用于卷启动功能，为实例指定匹配root device设备名称，通常是`/dev/vda`。

目前，用户不鼓励在请求时指定设备名称。

### Block device mapping V2

引入新的格式，是为了解决上面提到的问题，并且支持使用简单的格式所做不到的灵活性以及额外的功能。新的格式是一个含有字典的列表，包含下列字段（除了前面提到的已有的选项）：
* source_type，可以有以下值：
  * image
  * volume
  * snapshot
  * blank
* dest_type，可以有以下值:
  * local
  * volume
* guest_format，用于告诉Nova在挂载时如何格式化设备，通常用于blank local images，如果值为swap，则表示一个swap设备
* device_name，设备名称最好还是不要设置，除非用户希望重载image metadata中指定的设备。在Libvirt中，即使指定了设备名称，在实例内部的设备名称最终由driver来设置（按照字母序，比如指定某个设备名称是/dev/sde，但是在实例内部会按照字母序设置为/dev/sdb或者/dev/sdc等等）
* disk_bus和device_type，可能某些hypervisor会支持的low level details。disk_bus可能的值有`ide`、`usb`、`virtio`、`scsi`，device_type可以是`disk`、`cdrom`、`floppy`、`lun`（**//TODO，挖个坑，lun了解一下？**）。以上值不是全部的列表，一般设置为空即可。
* boot_index，定义了hypervisor boot实例尝试存储设备的顺序。每个可用于boot device的设备，应当设置一个唯一的index值，从0开始，依次递增。有些hypervisor不支持booting from multiple devices，所以只会考虑boot_index为0的设备。设置该值为None或者负数（比如-1），则表示这个设备不可以用于启动。通常的做法是，对于boot device，boot_index设置为0，其他设备设置为None（或不设置）。

### 有效的source/destination组合

source_type和dest_type的组合用于定义块设备条目的类型，目前支持下列组合：
* image->local，预留的条目，用于表示实例在Glance的镜像启动。
* volume->volume，表示挂载到实例的Cinder卷，可以被标注为启动设备
* snapshot->volume，表示在Cinder的卷快照创建一个卷，并挂载到实例上，可以被标注为启动设备
* image->volume，表示在Glance中下载镜像到Cinder的卷，并且把卷挂载给实例，可以被标注为bootable
* blank->volume，创建一个空的Cinder卷，并把它挂载到实例上，需要设置size
* blank->local，取决于guest_format，通常表示hypervisor本地存储的ephemeral blank disk或者swap disk（实例只能有一个swap磁盘）

Nova不会在一个请求中同时允许BDMv1和BDMv2的混合使用，会在接收一个boot请求时做基础的校验。
