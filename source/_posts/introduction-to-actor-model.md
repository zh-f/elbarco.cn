---
title: 并发编程模型之Actor模型
date: 2017-01-21 17:30:30
tags: [Concurrency, Actor Model]
---


上一篇文章[《几个概念：并发、并行、进程、线程和协程》](https://elbarco.cn/2017/01/20/general-concepts-concurrency-parallelism-process-thread-coroutine/)中，对并发和并行的概念做了一个简单的解释，而本文中则从两种并发编程模型讲起，简单的介绍一下Actor模型。

<!--more-->

## 两种并发编程模型

并发编程中有两类常见的模型：共享内存和消息传递。

### 共享内存模型

![](http://elbarco.eos.eayun.com/imgs/shared-memory.png)

在并发编程的共享内存模型中，各组件通过读写内存中的共享对象进行交互。

共享内存模型的其他示例：
* A、B两个在同一个电脑中的处理器（或者同一个处理器的两个核）共享同一物理内存
* A、B两个运行在同一电脑上的程序，共享同一文件系统，其中文件可以读写
* A、B是同一Java程序中的两个线程，共享相同的Java对象。

### 消息传递模型

![](http://elbarco.eos.eayun.com/imgs/message-passing.png)

在消息传递模型中，并发模块通过通信信道将消息发送到彼此进行交互。发出的消息会在队列中等待处理。

消息传递模型的示例还有：
* A和B是通网络中的两台计算机，通过网络通讯
* A是一个浏览器，B是一个web服务器，A打开连接请求网页，B发送页面数据给A

## Actor模型

### 认识Actor模型

上面我们认识了两种并发模型，actor模型就属于消息传递模型。actor模型的基本思想是使用actor作为并发基元，可以根据接收的消息做出不同的响应（或动作、行为）：
* 将有限数量的消息传递给其他的actor
* 产生有限数量的新的actor
* 当下一个传入的消息被处理时，改变自己的内部行为

actor模型使用异步消息传递进行通信。特别要指出，actor之间不适用任何中间实体，比如通道，相反的，每个actor拥有可以被寻址的信箱。不要把地址和身份信息弄混淆，每个actor可以有零个、一个或多个地址。当一个actor发送信息时，它必须知道接收方的地址。此外，actor可以给自己发信息，这样他们就可以自己接受信息并且稍后进行处理。注意，这里提到的邮箱并不是概念的一部分，而是一个特性的实现，

actor模型可以用于并发系统的建模，正是因为每个actor与其他actor完全独立，没有共享状态，则就没有了竞争状态（race condition），actor之间的通讯和交互完全通过异步消息。

支持actor模型的编程语言有Elixir、Erlang、Scala等，Java语言层面并不支持，但是可以引入Akka，一个用Scala编写的库，用于简化编写容错的、高可伸缩性的Java和Scala的actor模型应用。

## 写在后面

本文重点关注并发编程的两种模型及对Actor模型做一个简单的介绍，Akka的学习会放到后面，由于对Scala不了解，网上看到的例子没有办法贴到这里与大家一起分析。


## 参考资料

[1].MIT.6.005.[Reading 17:Concurrency](http://web.mit.edu/6.005/www/fa14/classes/17-concurrency/)
[2].Wikipedia.[Actor Model](https://en.wikipedia.org/wiki/Actor_model)
[3].Ruben Vermeersch.[Concurrency in Erlang and Scala](https://rocketeer.be/articles/concurrency-in-erlang-scala/)
[4].Marko Dvečko.[Introduction to Concurrent Programming](https://www.toptal.com/software/introduction-to-concurrent-programming)
