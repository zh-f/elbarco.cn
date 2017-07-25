---
title: OpenStack DBaaS组件Trove简介
date: 2017-07-25 16:02:28
tags: [OpenStack, Trove, Translation]
---
## Trove架构
Trove包含下面几个主要组件：<!--more-->
* API Server
* Message Bus
* Task Manager
* Guest Agent
* Conductor

### API Server
API endpoint（trove-api）本质上是一个HTTP的Web服务，具备处理鉴权、授权、与数据存储相关的基本命令和控制功能。根据数据库不同，API还有一些不同的扩展。

API Server与两个系统沟通——与Task Manager沟通，来处理复杂的异步任务；直接与Guest Agent沟通来处理简单的任务比如获取MySQL用户列表等，这部分操作均是同步的。API Server不做任何重大/复杂的事情，它的任务就是接收请求，将其转化为消息，校验它们，并将这些消息转发到任务管理器（Task Manager）和访客代理（Guest Agent）。

* 一个RESTful风格的组件
* 入口 - `Trove/bin/trove-api`
* 使用WSGI launcher，由`Trove/etc/trove/api-paste.ini`配置
	* 定义了过滤器管道、令牌认证、速率限制等
	* 定义了app_factory为`trove.common.api:app_factory`提供给trove应用
* API 类（WSGI Router）将REST路径连接到相应的Controller上
	* Controller的实现在相关的模块下（如versions/instance/flavor/limits）的`service.py`中
* Controller通常将实现重定向到`models.py`中的一个类
* 另一些组件（Task Manager，Guest Agent）的api模块通过RabbitMQ发送请求

### Message Bus

这部分组件仿照了Nova架构。Message Bus其实就是一个消息队列。

一个典型的消息传递事件从API服务器接收到用户的请求开始。API服务器认证用户确保用户具备执行响应命令的权限。对请求中涉及到的对象的可用性进行评估，如果可用，将请求路由到相关Worker的排队引擎。Workers不断根据自己的角色监听消息队列，当这种监听产生一个工作请求时，Worker将对该任务进行任务分配并开始执行。完成任务后，Worker将响应发送到消息队列，由API服务器接受并中继到始发用户的队列。在整个过程中，数据库记录根据需要会被查询、添加或者删除。

### Task Manager

Task Manager（trove-taskmanager）就是干粗活累活的家伙，比如配置一台实例，管理实例生命周期和在实例上进行操作。任务管理器接收来自API Server的消息，通过同意消息进行响应，并开始执行任务。有几个复杂的任务，比如重新分配数据库规格和创建实例等，他们均需要通过HTTP请求调用OpenStack的服务，同时也需要轮询服务，知道实例变为活动状态，并且还向客户代理发送消息。任务管理器处理在多个分布式系统中发生的进程流。

任务管理器是有状态的，它在其系统内部运行复杂的流程。如果在状态处理期间任务管理器节点脱机，则操作将失败。任务流系统将最终实现为长时间运行运行的任务。（The Task Flow system will be eventually implemented for long running tasks.）

* 这是一个监听RabbitMQ topic的服务
* 入口 - `Trove/bin/trove-taskmanager`
* 作为一个RpcService运行，通过`Trove/etc/trove/trove-taskmanager.conf.sample`配置文件进行配置，定义了`trove.taskmanager.manager.Manager`作为manager，基本上这是通过队列到达的请求的入口点
* 如上所述，使用TaskManager的api模块，使用`_cast()`或者`_call()`（同步/异步）将对该组件的请求从另一个组件推送到MQ中，并放置方法命作为一个参数
* `Trove/openstack/common/rpc/dispatcher.py` 中的`RpcDispatcher.dispatch()`通过反射的方式调用Manager中合适的方法
* 然后，Manager将该处理重定向到`models.py`模块中的一个对象，它使用context和instance_id从相关类加载一个对象
* 实际的处理一般在`models.py`中完成

### Guest Agent

客户代理（Guest Agent，trove-guestagent）运行在客户实例内部，负责管理和执行数据存储本身的操作。它负责使数据存储在线，这可能是一个复杂的任务。热支持（Heat support）将来将成为Trove的默认配置和仪器引擎，从而减少了将数据存储库联机的任务。Guest Agent还通过Conductor（指挥器）向API Server发送心跳信息。

每个数据存储器都实现有一个客户端代理，负责为该数据存储器执行特定人物。比如Redis的客户代理行为与MySQL的客户代理行为就会不同。不过他们必须履行诸如创建和调整规格的基础操作。

* 与Task Manager类似，服务运行起来监听RabbitMQ topic
* Guest Agent在每个数据库实例中运行，所以使用专有的RabbitMQ topic（通过实例ID来标识）
* 入口 - `Trove/bin/trove-guestagent`
* 作为一个RpcService运行，通过`Trove/etc/trove/trove-guestagent.conf.sample`配置文件进行配置，定义了`trove.guestagent.manager.Manager`作为manager，基本上这是通过队列到达的请求的入口点
* 如上所述，使用Guest Agent的api模块，使用`_cast()`或者`_call()`（同步/异步）将对该组件的请求从另一个组件推送到MQ中，并放置方法命作为一个参数
* `Trove/openstack/common/rpc/dispatcher.py` 中的`RpcDispatcher.dispatch()`通过反射的方式调用Manager中合适的方法
* 然后，Manager将对对象的处理重定向到`dbaas.py`中
* 实际处理一般在`dbaas.py`中完成
 
### Conductor

指挥器（Conductor）是运行在宿主机上的饿一个服务，负责接收客户实例中的消息，并在宿主机上更新信息，比如，实例的状态和当前备份的状态。有了指挥器，用户的实例不需要直接连接到宿主机的数据库。指挥器通过Message Bus监听RPC消息，并执行相关的操作。指挥器与客户代理有些类似，因为它是一个监听RabbitMQ主题的服务，不同的是Conductor运行在宿主机上，而非客户实例内部。客户代理通过将消息放入配置的消息队列——conductor_queue，默认为`trove-conductor`——来与指挥器进行信息交互。

* 入口 - `Trove/bin/trove-conductor`
* 作为一个RpcService运行，通过`Trove/etc/trove/trove-conductor.conf.sample`配置文件进行配置，定义了`trove.conductor.manager.Manager`作为Manager
* 如上面的客户代理类似，请求通过其他组件使用_cast()（异步的）推送到消息队列。一般来讲，消息格式为`{"method": "<method_name>", "args": {<arguments>}}`
* 实际的数据库更新操作由`trove/conductor/manager.py`完成
* "heartbeat"操作更新实例的状态，通常由Guest Agent来报告实例状态，如从NEW到BUILDING到ACTIVE等等
* "update_backup"方法修改备份的详情，包括它的当前状态、备份大小、类型和校验码（checksum）

## 代码仓库

* Trove Server (https://github.com/openstack/trove)
* Trove Integration (https://github.com/openstack/trove-integration)
* Trove Client (https://github.com/openstack/python-troveclient)

## 安装部署

* How to install trove as part of devstack: [trove/installation](https://wiki.openstack.org/wiki/Trove/installation)
* How to use trove-integration: [trove/trove-integration](https://wiki.openstack.org/wiki/Trove/trove-integration)
* How to set up unit tests to run with tox: [trove/unit-testing](https://wiki.openstack.org/wiki/Trove/unit-testing)
* How to set up a testing environment and run redstack tests after installation: [trove/integration-testing](https://wiki.openstack.org/wiki/Trove/integration-testing)
* How to set up your Mac dev environment to debug: [trove/dev-env](https://wiki.openstack.org/wiki/Trove/dev-env)
* Releasing python-troveclient [trove/release-python-troveclient](https://wiki.openstack.org/wiki/Trove/release-python-troveclient)
* Creating release notes with Reno [trove/create-release-notes-with-reno](https://wiki.openstack.org/wiki/Trove/create-release-notes-with-reno)

## 说明

翻译自[Trove wiki](https://wiki.openstack.org/wiki/Trove)