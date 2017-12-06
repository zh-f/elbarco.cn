---
title: Python中的Coroutine与并发编程
date: 2017-10-23 10:54:23
tags: [Coroutines, Python, Concurrency]
---

接上文，在[《Python中的Coroutines》](https://elbarco.cn/2017/10/18/python-coroutines/)中，我们<!--more-->介绍了协程的执行、关闭、异常等，并结合大量数据处理的例子进行了学习，最后还留下了一个Coroutine与并发编程的坑，本文就是来填坑的……

一般来讲我们可以将数据发送给协程、将数据通过消息队列发送给线程、通过消息将数据发送给进程。协程天然适合解决线程和分布式系统中的场景，例如我们可以将协程包裹在线程或者子进程中，如下图所示：
![]()


