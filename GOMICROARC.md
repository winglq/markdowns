---
title: go-micro 代码阅读笔记1-构架
date: 2018-12-13
tags: [golang, go-micro, 微服务]
---

## 介绍

go-micro是一个由golang编写的微服务框架。不同的微服务一般由不同的人实现，所以各服务间通信的接口就显得尤为重要。go-micro使用protobuf定义各服务间的接口。使用定义protobuf文件还可以生成微服务的框架代码。服务实现者只需要完成生成代码的接口。go-micro提供服务发现，load balance，事件通知等功能。

## 构架

go-micro各模块划分非常清晰，每个模块在代码中都是一个独立的文件夹。他主要由以下几个模块组成：server, client, transport, codec, registry, selector, broker。这些模块都有多个实现，可以自由选择。各模块的层级关系如下图所示。

```
         ---------------------------------------------------------------
         | ----------------------------------------------------------- |
         | |                           Service                       | |
         | ----------------------------------------------------------- |
         | ------------------------               -------------------- |
         | |        Client        |               |      Server      | |
         | ------------------------               -------------------- |
         | ---------- --------- ------------ ----------- ------------- |
         | | Broker | | Codec | | Registry | | Selector| | Transport | |
         | ---------- --------- ------------ ----------- ------------- |
         ---------------------------------------------------------------
```

### Service模块

Service位于其他模块的最上层，对go-micro的操作，比如启动服务或者发送客户请求，一般都由创建Service操作开始。服务的心跳信息也由Service维护。如果某个Service停止工作，那么发送心跳的ticker就不能给registry发送心跳，客户端请求也就无法发送到这个service。当然Service down到集群意识到service down之间还是有一段间隔的，这个只能由客户端重试来保证操作的顺利完成。

### Sever模块

Server模块用来管理微服务服务器，比如启动关闭等。Server在接收到transport发送的Message时，会将请求先解码，然后route到对应的service。编码response后发送给客户端。

### Client模块

在使用RPC服务时，调用者不需要知道具体的服务所在的地址，只需要提供服务名，方法和参数。RPC调用有两种方式，一种是Call另外一种是Stream。这应该是遵循的gRPC的规范。Call只发送一个Request得到一个Response，而Stream可以工作在全双工的形态下。

```
       --------       ----------
       |      |---1-->|        | 
       |      |       |Register| 
       |Client|<--2---|        |
       | -----|       ----------    ----------
       | |    |                     |        |
       | --3->|----------4--------->|        |
       |      |                     | Server |
       |      |<---------5----------|        |
       |      |                     |        |
       --------                     ----------
```

服务发现过程如上图所示：

1. 发送服务名称请求给Register
2. Register返回所有node的地址
3. 根据策略选择node
4. 发送服务请求
5. 接受response

### Transport模块

传输模块定义了数据传输方式，go-micro默认使用HTTP传输数据，未开启HTTPS的情况下使用HTTP 1.0，否则使用HTTP 2.0。传输模块实现了一个类似Socket的接口。

### Codec模块

go-micro默认实现了两种编码方式，json和protobuf。

### Registry, Selector模块

Registry模块用于注册各个服务的名称，go-micro默认使用consul作为注册服务。也可以使用其他注册服务，比如etcd。客户端使用Selector的策略来选择对应service的node。go-micro默认有Round Robin和Random两种选择策略。

### Broker模块

Broker模块用于go-micro的消息驱动功能。Client发送消息到某个Topic，监听这个Topic的一个或多个服务可以做出对应处理。go-micro实现了很多Broker对应的plugin，比如Rabbitmq。

## 总结

从代码来看，go-micro各模块功能定义还是非常清晰的。各模块都有多个实现可以根据需要选择对应的实现，整个代码也非常灵活。
