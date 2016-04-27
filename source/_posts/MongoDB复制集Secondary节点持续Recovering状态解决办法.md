---
title: MongoDB复制集Secondary节点持续Recovering状态解决办法
date: 2016-04-27 05:51:27
tags: [MongoDB]
---

前段时间发现MongoDB Replica Set中的某个Secondary节点一直持续Recovering状态，无法恢复，且上次操作时间（optimeDate）已经是N天前了，经过查看[官方文档](https://docs.mongodb.org/manual/tutorial/resync-replica-set-member/#replica-set-auto-resync-stale-member)，得知出现这种情况的原因在于复制集中主节点（Primary）一直写入oplog，而从节点（Secondary）的复制过程远远落后，赶不上主节点的oplog写入，就像赌气的孩子跑步一样，赶不上前面的小伙伴，索性一赌气就不走了……<!-- more -->当遇到这种情况的时候，是不可能指望从节点自己恢复的，需要我们手动重新同步（initial sync）。

官方给出了两种执行重新同步的方式——

* 完全清空数据目录然后重启mongod服务
* 在其他成员的数据目录下拷贝最近的数据然后重启mongod服务

这里，偷懒不想打包scp数据，索性采用了第一种方式：

1. 停止mongod服务：可在mongo shell中执行`db.shutdownServer()`来关闭mongod服务，也可以在shell中直接敲`mongod --shutdown`，或者简单粗暴直接`kill -2 <PID>`（这里不推荐`-9`，会造成下次启动不起来的情况，需要删除dbPath目录下的`mongo.lock`再尝试重新启动）。
2. 对旧的dbPath的目录重命名，以做备份
3. 启动mongod，指向新的空的dbPath目录

简单三步，MongoDB就会重新进行初始化同步，受限于数据量和网络环境等因素的影响，重新同步时间有长有短。重新同步完毕后，打开mongo shell查看复制集状态，一般情况下，这个从节点状态就会恢复正常了。然后要做的就是验证主从数据一致性，确保没问题之后，重命名过的dbPath目录可以删除了。

第二种方式，利用其它成员的最近数据进行启动的操作可见[官方文档](https://docs.mongodb.org/manual/tutorial/resync-replica-set-member/#replica-set-resync-by-copying)，这里就不赘述了。
