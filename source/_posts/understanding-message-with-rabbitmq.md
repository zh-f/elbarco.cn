---
title: Understanding message with RabbitMQ
date: 2017-12-01 18:17:31
tags: [AMQP, RabbitMQ]
---

本文为先导文章，对消息的一些概念，AMQP的架构、基本知识点进行一个梳理和学习，为OpenStack中基于AMQP实现RPC调用的后续文章做个铺垫。
<!-- more -->

## AMQP概述

AMQP(Advanced Message Queuing Protocol)，即高级消息队列协议，是一种应用层网络协议，它为特定客户端应用(application)与消息中间件代理(messaging middleware broker)之间的通信提供支持。本文针对AMQP 0-9-1 模型作一个简单的介绍，该模型即rabbitmq所使用的模型。

## RabbitMQ中的消息流

用过RabbitMQ的同学肯定对下面这个图会非常理解：
![message-flow](http://7xrgsx.com1.z0.glb.clouddn.com/msg-flow.png)
总体来讲，消息的生产者，产生消息，将消息发到消息队列RabbitMQ；消息的消费者在队列中取得消息执行后续操作，这就是RabbitMQ中的消息流。

然而在在将消息推送到MQ或者在MQ中消费时，我们要连接到MQ上。在连接的时候，客户端会创建一个TCP连接到RabbitMQ broker上。一旦连接成功，则客户端会创建一个AMQP channel。AMQP的channel是在TCP连接上的虚拟频道，当我们发布消息，订阅一个队列或者是接收消息，均在频道中完成——为什么需要AMQP channel呢？因为TCP会话的建立和销毁对于操作系统来讲，是十分昂贵的。我们假设说，我们的客户端连接到MQ上进行消息消费，短时间内产生大量的TCP连接，消费完成后，又要将这些TCP连接销毁，这不仅会造成了TCP连接的巨大浪费，而且操作系统每秒钟创建的连接数量有限。很快我们就会遇到性能瓶颈。于是，AMQP channel就诞生了，在一个TCP连接上使用多个频道，每个频道都会被分配一个唯一ID作为标识，在保证每个线程的私有连接的前提下，显著的提高性能，下面是一个生动的示意图：
![amqp-channel](http://7xrgsx.com1.z0.glb.clouddn.com/amqp-channel.png)

## 从队列说起

现在，已经对消息整个的生产、消费过程有了大概的了解，我们再进到内部去看下，消息究竟在RabbitMQ内部是如何流转的。

从概念上来讲，消息的成功流转离不开三部分：exchange，queue和binding：
![amqp-stack](http://7xrgsx.com1.z0.glb.clouddn.com/amqp-stack.png)

* Exchange是生产者发布消息的地方
* Queue是消息结束并被消费者接收的地方
* Binding就是消息如何从特定的Exchange被路由到指定队列的一系列规则

### 获取队列中的消息

我们先来说说队列。在队列中获取消息有两种方式：
1. 使用AMQP命令`basic.consume`来启动一个队列的消费者（订阅者），如果你的消费者需要处理一个队列的大量消息或者要求一旦有消息达到队列能够立刻自动的接收到消息，则需要使用这种方式；
2. 使用AMQP命令`basic.get`直接访问队列获取一条消息。使用该命令后会使得消费者接收队列的下一条消息，并且在下次调用`basic.get`之前不会再接收队列的消息，即订阅队列，接收单条消息，取消订阅。千万不要在循环中使用`basic.get`以求替代`basic.consume`，要合理的进行订阅来提高吞吐。

### 消息队列无订阅者或有多个订阅者

如果消息队列没有订阅的消费者，消息会在队列中等待。

如果一个RabbitMQ消息队列有多个消费者，那么队列中的消息将以轮询的方式服务于消费者，即，每条消息只会发送给订阅该队列的**某一个**消费者。

### 消息确认

消费者接收到的每条消息都需要得到确认——每个消费者可以选择要么显示的通过使用AMQP命令`basic.ack`发送确认通知给RabbitMQ，或者可以选择在订阅到队列的时候设置参数`auto_ack`为`true`，指定了该参数后，RabbitMQ会在消费者接收到消息后自动认为消息已经确认收到了。注意，这里的消息确认，不是告知消息的发送者，而是告诉RabbitMQ消费者已经收到了消息，可以安全的将该消息在队列中移除了。

如果处理消息比较集中和耗时，可以考虑延迟确认消息，直到处理结束。

### 消息拒绝

如果消费者在处理某条消息的时候没有发送确认信息（如断开连接等），则RabbitMQ会认为该消费者不具备接收消息的条件，会将该消息重新发送给下一个订阅者。但是这种消息拒绝的方式会增加服务器负担。

我们还可以使用`basic.reject`来拒绝RabbitMQ发送给消费者的消息。

>注：此外，对RabbitMQ来说，还可以使用`basic.nack`，这是RabbitMQ中对reject命令特殊的扩展实现。

如果设置`reject`命令的参数`requeue`为`true`，则RabbitMQ会将消息发送给下一个订阅的消费者，否则RabbitMQ会立刻在队列中删除这条消息而不发送给新的消费者。

当然，不想处理消息的时候还可以通过确认消息已收到来处理，在收到某些格式不正确的消息并确认没有消费者能处理时，这么操作十分有效。

> 注，在RabbitMQ的某些新版本中，会支持一个特殊的[`dead letter`队列](https://www.rabbitmq.com/dlx.html)，即无法投递的消息队列。如果使用`reject`命令并设置参数`requeue`为`false`，则消息会被丢到该队列。

### 创建队列

消息的消费者或者生产者都可以使用AMQP命令`queue.declare`来创建队列。但是消费者不能在已经在相同频道上订阅到其他队列的前提下声明或创建队列，必须先取消订阅将频道至于一种“可传输”的模式。

创建队列时，一般由消费者指定队列的名字，如果没有指定，则RabbitMQ会随机生成一个名字，在`queue.declare`的返回值中体现出来。随机队列名在一些临时的匿名队列场景下非常有用，比如基于AMQP应用的RPC调用。

在创建队列时，有两个参数很有用：
* `exclusive` - 设置为true，则队列会设置为私有状态，常用于控制队列只允许有一个消费者的情况；
* `auto-delete` - 队列在最后一个消费者取消订阅后自动删除，如果只需要一个临时队列提供给一个消费者，结合`auto-delete`和`exclusive`两个参数，当消费者断开连接时，队列自动被删除。

创建一个队列，恰好这个队列已经存在，RabbitMQ会直接返回成功。这个特性可以用于判断队列是否存在，在创建队列时，指定`queue.declare`的参数`passive`为`true`即可；如果队列不存在，则直接返回一个错误信息并不创建队列。

### 小节结语

队列是AMQP消息的基石——
* 为等待被消费的消息提供了栖息地；
* 完美的适用于负载均衡，只需要使很多消费者订阅同一个队列即可——因为RabbitMQ会使用轮询的方式处理消息；
* RabbitMQ中所有消息的终点

## 开，往消息队列开……

前面我们对消息队列Queue进行了比较详尽的介绍，那现在的问题是，消息是怎么抵达消息队列的？这时候，就需要exchange和binding了。

所有的消息均要先发送到exchang（路由），然后基于特定的规则，RabbitMQ会决定将消息发往哪个队列。这些规则被称为*routing keys*，一个队列可以说“通过一个*routing key*，*绑定*到一个exchange上”。

如果遇到了多个队列该怎么办？这里就要提到四种exchange类型，分别是`direct`、`fanout`、`topic`和`headers`，每一种都实现了不同的路由算法。`headers`允许通过匹配AMQP消息的header而不是routing key，所以我们这里不去深究和探讨了。

### Direct exchange

字面意思，直接路由。如果routing key匹配，则消息会被发送到响应的队列中，如下图所示：
![direct-exchange](http://7xrgsx.com1.z0.glb.clouddn.com/direct-exchange.png)

所有的消息队列必须实现这种方式，包括创建一个名称为空字符串的exchange，如：
```
$channel->basic_publish($msg, '', 'queue-name');
```
第一个参数标识了要发送的消息，第二个参数，一个空字符串，标识了指向默认的exchange，第三个参数就是routing key，也就是声明队列所使用的名称。

如果默认的direct exchange不能满足要求，可以使用`exchange.declare`命令创建自己所需要的exchange。

### Fanout exchange

扇区路由，示意图如下：
![](http://7xrgsx.com1.z0.glb.clouddn.com/fanout-exchange.png)

 Exchange会将收到的消息组播（multicast）到绑定的消息队列中，即这种模式下，支持应用根据一个（only one）消息做出不同的反应。比如我们考虑这么一个用户场景，在用户上传完图片后，既要更新图片缓存，又要奖励用户操作，那么此时如果使用fanout exchange，只需要将两个consumer都绑定到这个exchange上即可。那么如果还需要在上传图片后增加新的处理，只需要写好消费者的功能代码绑定到exchange上即可，对于消息生产者来讲，代码是完全解耦的。

### Topic exchange

这种路由方式，可以实现来自不同消息源的消息到达同一队列，示意图如下：
![](http://7xrgsx.com1.z0.glb.clouddn.com/topic-exchange.png)

这里留一个小问题：`为什么OpenStack中使用Topic Exchange比较多？`

## 多租户：虚拟主机（vhost）和隔离

每个RabbitMQ server都有能力创建多个虚拟的消息代理，即virtual host，简称vhost。每个vhost都是一个迷你的RabbitMQ server，具有自己独有的Queue、Exchange和Binding，更重要是的是，具有自己的权限。这使得多个应用同时可以安全无忧的使用同一个RabbitMQ服务器。

在RabbitMQ中，默认的vhost=/，在不需要多租户的场景下，默认值就足够了。在创建RabbitMQ用户的时候需要指定至少一个vhost。

>注，通过vhost隔离的租户是绝对的，即你不能将vhost A的队列绑定到vhost B的exchange上。

可以使用命令查看vhosts：
```
[root@rabbit1 ~]# rabbitmqctl list_vhosts
Listing vhosts ...
/
```
使用命令创建一个vhost：
```
[root@rabbit1 ~]# rabbitmqctl add_vhost f
Creating vhost "f" ...
[root@rabbit1 ~]# rabbitmqctl list_vhosts
Listing vhosts ...
f
/
```

## 消息的持久化

每个Queue和Exchange，都有一个`durable`属性，默认值为`false`，即默认情况下RabbitMQ不会在服务器宕机或者重启后重建Queue或Exchange，所以建议这个值一定要设置成`true`。

此外，只有Queue和Exchange的`durable`还不完全够，消息的持久化还需要三个要点：
1. 将其选项`delivery mode`要设置成`2`，即`persistent`，持久的；
2. 消息被发布到`durable`的Exchange；
3. 消息抵达一个`durable`的Queue

满足上述三个条件，消息的持久化就稳了。

RabbitMQ通过将持久化的消息写入磁盘日志文件来确保消息在重启时不是丢失，即当发布一个持久化的消息到持久化的exchange，在写入到日志文件之前是不会发送消息的响应。如果持久化的消息被路由到非持久化的队列，则会自动在持久化日志中移除，即无法保证消息在重启时不会丢失。

然而持久化虽好，却不要“贪杯”。因为将消息持久化到磁盘上比直接存储在内存中要慢很多，这就会面临几个问题：
1. 会减少RabbitMQ每秒处理的消息数量，这个降低的比例甚至能达到10倍或者更多；
2. 持久化消息在RabbitMQ的内置集群中表现不佳；

那到底应不应该使用persistent/durable消息呢？首先还是要评估一下性能需求。如果单节点的RabbitMQ需要每秒处理100,000+的数据，那么可能持久化信息就不是一个好的选择。

## 解决事务的方案：发送方确认模式

由于AMQP内部事务对性能有很大瓶颈，现采取发送方确认模式保证事务，将信道设置为confirm模式，所有在此信道上发布的消息都会有一个唯一的ID号，当被投递到匹配的队列时，信道就会发送一个发送方确认模式给生产者应用程序，这个模式是异步的，应用程序可以等待确认的同时继续发送下一条，但如果是持久化的消息，会在写入磁盘之后消息发出。

如果发送内部错误而导致消息丢失，RabbitMQ会发送一条nack(not acknowledged,未确认)消息，这种模式下每分钟可追踪数以百万计的消息投递。



























