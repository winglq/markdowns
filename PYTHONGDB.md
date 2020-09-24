---
title: 如何用gdb调试python程序
date: 2020-09-24
tags: [python, gdb]
---

之前写的一个python服务突然挂住了，于是想看下挂住线程的堆栈，用pstack只能看到python的栈，不能看到python code的栈，记录下使用gdb调试python程序的方法。下面的方法都是基于CentOS 7。

## 安装python debug包

首先打开CentOS的Debuginfo repo，将下面内容加到/etc/yum.repos.d/CentOS-Debuginfo.repo。这个配置文件是默认安装的，只是enabled=0，把他改成1就可以了。

```
[base-debuginfo]
name=CentOS-7 - Debuginfo
baseurl=http://debuginfo.centos.org/7/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-Debug-7
enabled=1
```
然后安装python的debug包。注意，先用rpm命令看下python的rpm包版本，然后再python-debuginfo后加对应的版本号。不加版本号可能会安装错误的debug包。

```
yum install -y python-debuginfo<-version>
```

## 安装libpython解析python frame

进如[cpython的项目主页](https://github.com/python/cpython)，根据自己的python版本号，选择对应的branch，比如我还使用的是python 2.7就选择branch 2.7。**版本号一定要选择正确**，我使用master版本的libpythoh.py去调试python 2.7就会遇到`Unable to locate gdb frame for python bytecode interpreter`或者`Unable to locate python frame`的错误，这是由于新版本的python和老版本的python evaluate python代码的frame名字发生了变化，导致新版libpython找不到老版本python的code frame。进入目录`Tools/gdb/`将`libpython.py`下载到本地`/usr/lib/python2.7/site-packages`中。

## 开始调试

使用下面命令进入gdb，第一个命令直接attach到需要调试的进程，第二个命令使用core file来调试。

```
gdb python <pid>
gdb python <core_file>
```

接着就可以使用`py-list`, `py-bt`等命令调试程序了。可以用`py-`加`Tab`键来显示所有py命令。
