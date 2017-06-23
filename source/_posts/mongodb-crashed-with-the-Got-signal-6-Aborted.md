---
title: Got signal:6 (Aborted) 引起的MongoDB崩溃分析解决
date: 2017-06-23 09:51:41
tags: [MongoDB]
---
## 一、背景

近日，同事在对MongoDB的读写压力进行测试，再插入大量数据时，常会遇到MongoDB服务莫名崩溃。于是，这边对日志进行了分析——
<!--more-->
发现，在日志中，有如下的一段backtrace：
```
2017-06-21T11:59:31.290+0800 F -        [conn963] Got signal: 6 (Aborted).

 0xf5e669 0xf5dce2 0xf5e096 0x3221032660 0x32210325e5 0x3221033dc5 0xda0c59 0x8dd622 0x8de181 0x8b31d7 0x8d1a17 0x8d34d6 0x9bdc64 0x9bebed 0x9bf8fb 0xb9340a 0xaa3480 0x7e99fd 0xf1badb 0x3221407aa1 0x32210e8aad
----- BEGIN BACKTRACE -----
{"backtrace":[{"b":"400000","o":"B5E669"},{"b":"400000","o":"B5DCE2"},{"b":"400000","o":"B5E096"},{"b":"3221000000","o":"32660"},{"b":"3221000000","o":"325E5"},{"b":"3221000000","o":"33DC5"},{"b":"400000","o":"9A0C59"},{"b":"400000","o":"4DD622"},{"b":"400000","o":"4DE181"},{"b":"400000","o":"4B31D7"},{"b":"400000","o":"4D1A17"},{"b":"400000","o":"4D34D6"},{"b":"400000","o":"5BDC64"},{"b":"400000","o":"5BEBED"},{"b":"400000","o":"5BF8FB"},{"b":"400000","o":"79340A"},{"b":"400000","o":"6A3480"},{"b":"400000","o":"3E99FD"},{"b":"400000","o":"B1BADB"},{"b":"3221400000","o":"7AA1"},{"b":"3221000000","o":"E8AAD"}],"processInfo":{ "mongodbVersion" : "3.0.6", "gitVersion" : "1ef45a23a4c5e3480ac919b28afcba3c615488f2", "uname" : { "sysname" : "Linux", "release" : "2.6.32-642.6.2.el6.x86_64", "version" : "#1 SMP Wed Oct 26 06:52:09 UTC 2016", "machine" : "x86_64" }, "somap" : [ { "elfType" : 2, "b" : "400000" }, { "b" : "7FFC4BCCC000", "elfType" : 3 }, { "path" : "/lib64/libpthread.so.0", "elfType" : 3 }, { "path" : "/lib64/librt.so.1", "elfType" : 3 }, { "path" : "/lib64/libdl.so.2", "elfType" : 3 }, { "path" : "/usr/lib64/libstdc++.so.6", "elfType" : 3 }, { "path" : "/lib64/libm.so.6", "elfType" : 3 }, { "path" : "/lib64/libgcc_s.so.1", "elfType" : 3 }, { "path" : "/lib64/libc.so.6", "elfType" : 3 }, { "path" : "/lib64/ld-linux-x86-64.so.2", "elfType" : 3 } ] }}
 mongod(_ZN5mongo15printStackTraceERSo+0x29) [0xf5e669]
 mongod(+0xB5DCE2) [0xf5dce2]
 mongod(+0xB5E096) [0xf5e096]
 libc.so.6(+0x32660) [0x3221032660]
 libc.so.6(gsignal+0x35) [0x32210325e5]
 libc.so.6(abort+0x175) [0x3221033dc5]
 mongod(_ZN5mongo12SecureRandom6createEv+0x1B9) [0xda0c59]
 mongod(_ZN5mongo31SaslSCRAMSHA1ServerConversation10_firstStepERSt6vectorISsSaISsEEPSs+0x16F2) [0x8dd622]
 mongod(_ZN5mongo31SaslSCRAMSHA1ServerConversation4stepERKNS_10StringDataEPSs+0x2F1) [0x8de181]
 mongod(_ZN5mongo31NativeSaslAuthenticationSession4stepERKNS_10StringDataEPSs+0x27) [0x8b31d7]
 mongod(+0x4D1A17) [0x8d1a17]
 mongod(+0x4D34D6) [0x8d34d6]
 mongod(_ZN5mongo12_execCommandEPNS_16OperationContextEPNS_7CommandERKSsRNS_7BSONObjEiRSsRNS_14BSONObjBuilderEb+0x34) [0x9bdc64]
 mongod(_ZN5mongo7Command11execCommandEPNS_16OperationContextEPS0_iPKcRNS_7BSONObjERNS_14BSONObjBuilderEb+0xC1D) [0x9bebed]
 mongod(_ZN5mongo12_runCommandsEPNS_16OperationContextEPKcRNS_7BSONObjERNS_11_BufBuilderINS_16TrivialAllocatorEEERNS_14BSONObjBuilderEbi+0x28B) [0x9bf8fb]
 mongod(_ZN5mongo8runQueryEPNS_16OperationContextERNS_7MessageERNS_12QueryMessageERKNS_15NamespaceStringERNS_5CurOpES3_+0x77A) [0xb9340a]
 mongod(_ZN5mongo16assembleResponseEPNS_16OperationContextERNS_7MessageERNS_10DbResponseERKNS_11HostAndPortE+0xB10) [0xaa3480]
 mongod(_ZN5mongo16MyMessageHandler7processERNS_7MessageEPNS_21AbstractMessagingPortEPNS_9LastErrorE+0xDD) [0x7e99fd]
 mongod(_ZN5mongo17PortMessageServer17handleIncomingMsgEPv+0x34B) [0xf1badb]
 libpthread.so.0(+0x7AA1) [0x3221407aa1]
 libc.so.6(clone+0x6D) [0x32210e8aad]
-----  END BACKTRACE  -----

```

除了`Got signal: 6 (Aborted)`还有点意义，下面的这些trace，完全不知所云。

## 二、查询分析

找到关键词之后，查询这件事情就很简单的了，Google一下，发现在MongoDB的JIRA上，有人提问相同的问题，[>>传送门](https://jira.mongodb.org/browse/SERVER-28001)，在下面的回复中，提到了，原因是因为我们在插入数据时，打开的文件数量超过了操作系统的`ulimit`中的配置，并给出了配置的文档说明，[>>>传送门](https://docs.mongodb.com/manual/reference/ulimit/)，下面简单的总结一下——

大多数类Unix的操作系统，如Linux和Mac OS X，提供了一些限制和控制系统资源使用的机制，这里的系统资源比如说：线程、文件、网络连接数等等。这个控制即`ulimit`，用于避免单用户使用过多的系统资源，当然，有些时候`ulimit`的一些默认值相对较低，所以会影响一些正常的MongoDB操作。

简单的看一下如何设置资源限制——我们可以使用`ulimit`命令来检查目前的配置，例如：
```
[root@mongodb-master ~]# ulimit -a
core file size          (blocks, -c) 0
data seg size           (kbytes, -d) unlimited
scheduling priority             (-e) 0
file size               (blocks, -f) unlimited
pending signals                 (-i) 63706
max locked memory       (kbytes, -l) 64
max memory size         (kbytes, -m) unlimited
open files                      (-n) 1024
pipe size            (512 bytes, -p) 8
POSIX message queues     (bytes, -q) 819200
real-time priority              (-r) 0
stack size              (kbytes, -s) 10240
cpu time               (seconds, -t) unlimited
max user processes              (-u) 63706
virtual memory          (kbytes, -v) unlimited
file locks                      (-x) unlimited
```
在使用`ulimit`修改具体某个配置项的值时，例如修改open file时，语法为`ulimit -n <value>`。修改时还要注意，有`hard`和`soft`两个选项：

|选项|含义|例子|
|:---|:---|:---|
|-H|设置硬资源限制，一旦设置不能增加|ulimit -Hs 64，限制硬资源，线程栈大小为64K|
|-S|设置软资源限制，设置后可以增加，但是不能超过硬资源设置|ulimit -Sn 32，限制软资源，32个文件描述符|

MongoDB官方手册中给出的`mongod`和`mongos`的设置推荐值为：
* -f (file size): unlimited
* -t (cpu time): unlimited
* -v (virtual memory): unlimited 
* -n (open files): 64000
* -m (memory size): unlimited 
* -u (processes/threads): 64000

所以对照推荐值，修改我们mongodb-master的ulimit配置即可。具体配置的语法，根据不同的Linux发行版本可能不同，可以阅读[手册](https://docs.mongodb.com/manual/reference/ulimit/#review-and-set-resource-limits)获得帮助。

>注：修改后需要对应重启`mongod`服务。

## 三、延伸

ulimit作为对资源使用限制的一种方式，是有其作用范围的，它的作用对象是当前shell进程以及其派生的子进程，也就是说，上面我们配置完open file的值后，如果再打开一个shell终端，再次查看`ulimit -a`会发现open file的值看起来像“恢复原状”（revert）一样。

那么问题来了，刚才我们的设置是否生效还如何检查呢？
首先，我们要知道修改后重启的`mongod`服务的PID，然后使用命令：`cat /proc/<PID>/limits`来查看当前进程的`ulimit`配置：
```
[root@mongodb-master ~]# ps -ef | grep mongo
root      4802     1 42 Jun21 ?        20:52:00 /root/mongodb-linux-x86_64-3.0.6/bin/mongod -f /root/mongodb-linux-x86_64-3.0.6/master.conf
root     29337 28455  0 15:17 pts/0    00:00:00 grep mongo

[root@mongodb-master ~]# cat /proc/4802/limits
Limit                     Soft Limit           Hard Limit           Units     
Max cpu time              unlimited            unlimited            seconds   
Max file size             unlimited            unlimited            bytes     
Max data size             unlimited            unlimited            bytes     
Max stack size            10485760             unlimited            bytes     
Max core file size        0                    unlimited            bytes     
Max resident set          unlimited            unlimited            bytes     
Max processes             63706                63706                processes 
Max open files            64000                64000                files     
Max locked memory         65536                65536                bytes     
Max address space         unlimited            unlimited            bytes     
Max file locks            unlimited            unlimited            locks     
Max pending signals       63706                63706                signals   
Max msgqueue size         819200               819200               bytes     
Max nice priority         0                    0                    
Max realtime priority     0                    0                    
Max realtime timeout      unlimited            unlimited            us        

```
可以看到，这里我们的配置是生效的，如果服务重启后，对应是否生效，还需要检查和验证。

那么，是否有针对某个具体用户的资源加以限制的方法呢？对于CentOS6来说，可以修改系统的`/etc/security/limits.conf`配置文件，格式如下：
```
<domain>      <type>  <item>         <value>
```
其中，`<domain>`表示用户或者组的名字，还可以使用`*`作为通配符，不过**通配符对`root`用户可是不生效的**，切记。

不过我尝试各种软硬修改配置文件后，并没有发现`ulimit -a`有丝毫的变化，真的是扎铁了，老心，也许因为我用的是`root`用户？欢迎邮件交流：[zh.f@outlook.com](mailto:zh.f@outlook.com)

## 四、参考

[[1].Mongodb Crashed with the Got signal: 6 (Aborted)](https://jira.mongodb.org/browse/SERVER-28001)
[[2].Unix ulimit Settings](https://docs.mongodb.com/manual/reference/ulimit/)
[[3].How do I change the number of open files limit in Linux?](https://stackoverflow.com/questions/34588/how-do-i-change-the-number-of-open-files-limit-in-linux)
[[4].Linux ulimit命令](http://www.cnblogs.com/wangkangluo1/archive/2012/06/06/2537677.html)
[[5].ulimit -n not changing - values limits.conf has no effect](https://serverfault.com/questions/569288/ulimit-n-not-changing-values-limits-conf-has-no-effect)



