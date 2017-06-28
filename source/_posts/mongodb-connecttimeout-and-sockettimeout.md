---
title: 由一次服务连接MongoDB超时引发的思考
date: 2017-06-28 17:12:28
tags: [MongoDB]
---

## 起因

今天某个业务操作突然执行失败，查询服务日志发现，服务在些写数据的时候，连接被重置，和MongoTimeoutException，截取部分日志如下：
<!-- more -->
```
Caused by: com.mongodb.MongoException$Network: Operation on server xx.xx.xx.xx:27017 failed
	at com.mongodb.DBTCPConnector.doOperation(DBTCPConnector.java:215) ~[mongo-java-driver-2.13.0.jar:na]
	... 63 common frames omitted
Caused by: java.net.SocketException: Connection reset
	at java.net.SocketOutputStream.socketWrite(SocketOutputStream.java:118) ~[na:1.7.0_80]
	... 72 common frames omitted
.....
.....
Caused by: com.mongodb.MongoTimeoutException: Timed out after 0 ms while waiting for a server that matches 
{serverSelectors=[ReadPreferenceServerSelector{readPreference=primary}, LatencyMinimizingServerSelector{acceptableLatencyDifference=15 ms}]}. 
Client view of cluster state is {type=ReplicaSet, servers=[{address=xx.xx.xx.xx:27017, type=ReplicaSetArbiter, averageLatency=1.3 ms, state=Connected}, 
{address=xx.xx.xx.xx:27017, type=Unknown, state=Connecting}, 
{address=xx.xx.xx.xx:27017, type=ReplicaSetSecondary, averageLatency=1.1 ms, state=Connected}]
	at com.mongodb.BaseCluster.getServer(BaseCluster.java:82) ~[mongo-java-driver-2.13.0.jar:na]
	... 49 common frames omitted
```
可以看到我们的服务在连接MongoDB时超时——没找到primary节点，在0ms后Timeout，抛出异常，即下面这段异常才是暴露问题的地方：
```
Caused by: com.mongodb.MongoTimeoutException: Timed out after 0 ms while waiting for a server that matches 
...Client view of cluster state is...{address=xx.xx.xx.xx:27017, type=Unknown, state=Connecting}...]
```

## 分析

首先，通过在日志中找到最关键的那一部分，分析得出当时业务异常的问题是业务server到MongoDB Master节点网络稍微有些抖动/延迟，而我们配置的connectTimeout=0ms，所以业务server没有取得replica set中的primary节点，不知道该如何写入数据，才会抛出这个异常。

这里需要将connectTimeout适当调整即可。

## 探究

在我们的上层，无论何时创建MongoClient，driver会建立和service的连接，应用建立连接的等待时长和客户端请求后等待服务器响应的时长，取决于我们的`connection timeout`和`socket timeout`两个参数。

### Connection timeout

这个参数决定了我们的客户端等待建立与服务器建立一个连接的最长时间。一方面，我们希望连接超时时间足够长，这样我们的应用即使在面对较大的服务器负载或者断断续续的网络延迟的情况下，也可以较为可靠的与服务器端建立起一个连接，但是，另一方面，我们又不希望这个值过大，否则应用会`hang`住，在服务器暂时不可访问的情况下，过度的浪费时间在等待服务器的连接上。 所以设置这个值的大小便是仁者见仁智者见智的事情了。

这个参数的默认值，在Java的dirver中，`com.mongodb.MongoClientOptions.Builder#connectTimeout`的默认值是`1000*10`ms，即10s。

那在上面我们的案例中，设置为0ms显然是不可理的，网络延迟是不可能不存在的。

### Socket timeout

这个参数决定了我们的客户端等待服务端响应的最长时间，这个timeout参数控制了所有类型的请求——query、write、commands、authentication等等。

如果我们将数值设置为30s，则客户端不会等服务端响应超过30s钟。所以通常来讲，我们是不会限制这个时间的，这样可以使数据库的操作响应比较自由。

在大多数的driver中，这个参数都是无限大（或者没有限制的）。这个参数在Java的driver中，`com.mongodb.MongoClientOptions.Builder#socketTimeout`值是`0`,用于表示不限制。

## 参考

[[1].Do you want a timeout?](https://blog.mlab.com/2013/10/do-you-want-a-timeout/)
[[2].mongodb connection timed out error](https://stackoverflow.com/questions/40216639/mongodb-connection-timed-out-error)
[[3].Class MongoClientOptions](https://mongodb.github.io/mongo-java-driver/3.4/javadoc/com/mongodb/MongoClientOptions.html)


