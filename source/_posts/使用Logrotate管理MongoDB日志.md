---
title: 使用Logrotate管理MongoDB日志
date: 2016-06-30 15:26:03
tags: [Logrotate, MongoDB]
---

## 痛点

前段时间需要查询MongoDB日志，惊觉MongoDB的日志并没有配置自动切换轮转，这会导致在繁忙的业务下，日志增长量惊人。面对海量的MongoDB日志，开发和运维人员查看日志变的十分不方便，所以需要寻求使日志自动切换轮转的方式。<!-- more -->

## 选型

通过查看MongoDB官方文档，知悉MongoDB提供了几种轮转日志文件的策略，详见[这里](https://docs.mongodb.com/manual/tutorial/rotate-log-files/)（据说新版本的MongoDB已经完成了自动的日志轮转功能？）。其中，可以使用MongoDB提供的[`logRotate`](https://docs.mongodb.com/manual/reference/command/logRotate/#dbcmd.logRotate)命令或者通过向`mongod`进程发送`SIGUSR1`信号来实现。

然而看很多文章中均表示，MongoDB本身提供的logRotate机制存在很多问题，比如由于其不稳定性，会造成日志轮换中mongodb进程终止，不提供旧日志的压缩，即使轮转切换日志，还是占用了很多磁盘空间；日志文件重命名格式`mongodb.log.2016-10-22T17-44-44`不友好等等。所以我们在选择时就会变得很小心，尽量避免使用其内置logRotate。

被广泛认可的方案是通过[Logrotate](http://linux.die.net/man/8/logrotate)进行日志管理，其中可以执行脚本实现向`mongod`进程发送`SIGUSR1`信号。

## Logrotate

### 简介

Logrotate可以帮助我们管理日志文件。比如周期性的读取日志、压缩日志、备份日志、创建新的日志文件等，基本上你希望做的，都能实现。通常来讲，常被用于来避免单个日志文件增长为难以处理的大小。也常被用于删除旧的日志文件来释放磁盘空间。

通常来讲，默认的Logrotate会作为`/etc/cron.daily/`中的一个计划任务每天执行一次。
```shell
[root@localhost etc]# ls /etc/cron.daily/
cups  logrotate  makewhatis.cron  mlocate.cron  
```

### 配置说明

配置Logrotate通过编辑两处配置文件来完成：
* /etc/logrotate.conf
* /etc/logrotate.d/下面的不同服务特定的配置

`logrotate.conf`包含了通用的配置，下面是一个默认配置：

```
# see "man logrotate" for details
# rotate log files weekly
weekly

# keep 4 weeks worth of backlogs
rotate 4

# create new (empty) log files after rotating old ones
create

# use date as a suffix of the rotated file
dateext

# uncomment this if you want your log files compressed
#compress

# RPM packages drop log rotation information into this directory
include /etc/logrotate.d

# no packages own wtmp and btmp -- we'll rotate them here
/var/log/wtmp {
    monthly
    create 0664 root utmp
        minsize 1M
    rotate 1
}

/var/log/btmp {
    missingok
    monthly
    create 0600 root utmp
    rotate 1
}

# system-specific logs may be also be configured here.
```

上面的通用配置我们不用过多关心，因为我们具体服务的具体配置在目录`/etc/logrotate.d/`下。在这个目录里，许多应用在安装后已经设置了Logrotate，比如httpd，nginx等。下面，我们拿nginx的配置做一个简要的说明：

```shell
[root@localhost ~]# cd /etc/logrotate.d/
[root@localhost logrotate.d]# ll
total 44
-rw-r--r--. 1 root root 185 Aug  2  2013 httpd
-rw-r--r--. 1 root root 871 Jun 22  2015 mysqld
-rw-r--r--. 1 root root 302 Apr 26 23:10 nginx
-rw-r--r--. 1 root root 219 Nov 23  2013 sssd
-rw-r--r--. 1 root root 210 Aug 15  2013 syslog
-rw-r--r--. 1 root root 100 Feb 22  2013 yum
[root@localhost logrotate.d]# cat nginx 
/var/log/nginx/*.log {
        daily
        missingok
        rotate 52
        compress
        delaycompress
        notifempty
        create 640 nginx adm
        sharedscripts
        postrotate
                [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
        endscript
}
```
首先第一行，配置了要自动轮换的日志文件的路径`/var/log/nginx/*.log`，即针对在`/var/log/nginx`下的`*.log`文件进行轮换。

* daily：每天轮换日志。可选选项有daily，weekly，monthly和yearly
* missingok：找不到*.log文件也是ok的，不要方……
* rotate 52：保留52个日志文件，其他更老旧的日志文件删掉（在这里要配合daily使用，即保留52天的日志文件）
* compress：压缩日志文件（默认gzip格式）
	* delaycompress：延迟压缩任务直到第二次轮换日志才进行。结果会导致你会有当前的日志文件，一个较旧的没有被压缩过的日志文件和一些压缩过的日志文件
	* compresscmd：设置使用什么命令来进行压缩，默认是gzip
	* uncompresscmd：设置解压的命令，默认是gunzip。
* notifempty：不轮转空文件
* create 640 nginx adm：创建一个新的日志文件，并设置权限permissions/owner/group。本例中，使用用户ngxin和用户组adm创建了一个日志文件，文件权限为640.在很多系统中，owner和group一般都会是root。
* sharedscripts：在所有的日志轮换完毕后执行postrotate脚本。如果该项没有设置，则会在每个匹配的文件轮换后执行postrotate脚本。
* postrotate：轮换日志完成后运行的脚本。

更多的选项，参见[这里](http://linux.die.net/man/8/logrotate)。

## 使用Logrotate管理MongoDB日志

经过上面对Logrotate的简单说明，这是我们就可以开始使用它来管理MongoDB日志了。

### 找到日志文件及PID记录文件
首先，我们的MongoDB启动配置中，指定了`logpath`和`pidfilepath`：

```
logpath=/mongoData/mongodb_log/mongodb.log 
pidfilepath=/mongoData/mongodb.pid 
```
`mongod.pid`和文件`/mongoData/mongodb_data/mongod.lock`中都存有mongod的PID，用这两个文件都可以获取PID，任选其一即可。


### 编写配置文件

通过`man logrotate`查看详细参数，结合业务需求，编写的配置文件如下：

```
/mongoData/mongodb_log/mongodb.log  {
	daily
    missingok
    rotate 30
    copytruncate 
    dateext  
    compress
    notifempty
    create 644 root root 
    sharedscripts
    postrotate
    	/bin/kill -SIGUSR1 'cat /mongoData/mongodb.pid 2> /dev/null' 2> /dev/null || true
    endscript
}

```

这里做一下简单说明：
* `copytruncate` 这个命令很重要，意思是在创建副本后，将原文件清空，而不是将原文件重命名并创建新的日志文件。这样可以避免有些应用继续向原日志文件中输出，而不是新的日志文件。在没有配置这个命令之前，mongodb一直向轮换后的带时间戳的旧文件中输出日志。
* `dateext` 用于切换日志文件时命名成为`mongodb.log-YYYYMMDD`格式。 
* `create 644 root root` 644权限，即`-rw-r--r--`与之前的日志文件保持一直的权限即可。

### 验证

编写完配置文件之后，我们将文件拷贝到`/etc/logrotate.d/`下，执行命令`logrotate -f -v /etc/logrotate.d/<YOUR_CONFIG_FILE_NAME>`来验证日志是否被轮换了，示例执行结果如下：

```shell
root@localhost mongodb_log]# logrotate -f -v /etc/logrotate.d/mongologrotate
reading config file /etc/logrotate.d/mongologrotate
reading config info for /mongoData/mongodb_log/mongodb.log

Handling 1 logs

rotating pattern: /mongoData/mongodb_log/mongodb.log  forced from command line (30 rotations)
empty log files are not rotated, old logs are removed
considering log /mongoData/mongodb_log/mongodb.log
  log needs rotating
rotating log /mongoData/mongodb_log/mongodb.log, log->rotateCount is 30
dateext suffix '-20160630'
glob pattern '-[0-9][0-9][0-9][0-9][0-9][0-9][0-9][0-9]'
glob finding old rotated logs failed
copying /mongoData/mongodb_log/mongodb.log to /mongoData/mongodb_log/mongodb.log-20160630
set default create context
truncating /mongoData/mongodb_log/mongodb.log
running postrotate script
compressing log with: /bin/gzip
[root@localhost mongodb_log]# ll
total 69604
-rw-r--r--. 1 root root    37092 Jun 30 13:24 mongodb.log
-rw-r--r--. 1 root root  1047190 Jun 30 13:24 mongodb.log-20160630.gz
```

## 结语

至此，我们边完成了使用Logrotate来管理MongoDB日志了。可以看到，Logrotate十分强大，在使用时，可以通过`man logrotate`查看一下具体参数，知其然并知其所以然，让其更好地为我们所用。
