---
title: 为CentOS6.5安装Kernel3.10
date: 2016-03-12 09:08:34
tags: [Linux, CentOS6.5]
---

最近有想学习下Docker，在Linux下安装Docker对内核的要求至少是3.10以上，然而CentOS 6.5内核版本是2.6，所以首先要做的就是为CentOS 6.5安装3.10的Kernel。
<!--more-->

我们并不需要自己编译安装，而是有小伙伴在在[ELRepo](http://elrepo.org/tiki/tiki-index.php)上为我们准备好了一个package，我们只关心如何安装就好了。

## 启用ELRepo

```bash
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org  
rpm -Uvh http://www.elrepo.org/elrepo-release-6-6.el6.elrepo.noarch.rpm 
```

## 安装Kernel

```bash
yum --enablerepo=elrepo-kernel install kernel-lt
```

## 配置grub

需要编辑`/etc/grub.conf`来更改kernel顺序，将默认的1改为0.所以看起来应该是酱婶儿的：

```
default=0
timeout=5
splashimage=(hd0,0)/boot/grub/splash.xpm.gz
hiddenmenu
title CentOS (3.10.99-1.el6.elrepo.x86_64)
        root (hd0,0)
        kernel /boot/vmlinuz-3.10.99-1.el6.elrepo.x86_64 ro root=UUID=94e4e384-0ace-437f-bc96-057dd64f42ee rd_NO_LUKS rd_NO_LVM LANG=en_US.UTF-8 rd_NO_MD SYSFONT=latarcyrheb-sun16 crashkernel=auto  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet
        initrd /boot/initramfs-3.10.99-1.el6.elrepo.x86_64.img
title CentOS (2.6.32-573.12.1.el6.x86_64)
        root (hd0,0)
        kernel /boot/vmlinuz-2.6.32-573.12.1.el6.x86_64 ro root=UUID=94e4e384-0ace-437f-bc96-057dd64f42ee rd_NO_LUKS rd_NO_LVM LANG=en_US.UTF-8 rd_NO_MD SYSFONT=latarcyrheb-sun16 crashkernel=auto  KEYBOARDTYPE=pc KEYTABLE=us rd_NO_DM rhgb quiet
        initrd /boot/initramfs-2.6.32-573.12.1.el6.x86_64.img
...
```

## 重启并查看

```
reboot
```
重启后通过`uname -a`来查看内核版本
```
[root@iZ2853cmjatZ ~]# uname -a
Linux iZ2853cmjatZ 3.10.99-1.el6.elrepo.x86_64 #1 SMP Fri Mar 4 11:53:07 EST 2016 x86_64 x86_64 x86_64 GNU/Linux
```

大功告成！


