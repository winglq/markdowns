title: Detectron使用小结
author: qing
date: 2018-05-02
description: Facebook Detectron
tags:
category:
acl: 00

# Detectron使用

## 安装

我使用的是docker的部署，这种部署方式可以避免Caffe2相关的安装，尤其是在非官方支持的版本上安装时会带来很多问题。比如我就在下载Nvidia的CuDnn时死活无法注册账户，导致无法下载对应的package。其实Detectron的安装文档已经讲了绝大部分过程，只是遗漏了nvidia-docker安装的步骤，补充如下。
Docker部署方式只需要在宿主机上安装GPU的驱动，可以使用`lspci |grep Nvidia -i`查看GPU的型号。然后在Nvidia官网上下载和安装。接着安装Nvidia-docker，具体步骤参见[官文](https://github.com/NVIDIA/nvidia-docker)。安装完成后build detectron的dock image之后就可以使用了。具体步骤参见[detectron的install.md](https://github.com/facebookresearch/Detectron/blob/master/INSTALL.md)

## 使用

由于GPU数量有限，而GPU处理图像的速度比较慢，无法应用在分析大量视频的需求上。基本没怎么深入使用Detectron，只是根据guide跑了下sample。
