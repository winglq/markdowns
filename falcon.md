title: Falcon使用小结
author: qing
date: 2017-11-21
description: 最近几个月使用Falcon做了几个小项目，Falcon的优势很明显，上手容易，框架结构清晰。这里主要把中间遇到的问题在记录下。
tags:
category:

# Falcon使用小结

最近几个月使用Falcon做了几个小项目，Falcon的优势很明显，上手容易，框架结构清晰。这里主要把中间遇到的问题在记录下。

Falcon本身是为REST设计的，如果每个资源的操作相对简单独立，那么Falcon将比较适合这种项目。但是如果资源在`GET/PUT/POST/DELETE`无法满足要求时，代码就会显得比较凌乱。有时候都不知道要把一个LIST操作放到哪里合适。

如果一个资源除了增删改查还有其他操作，我现在想到的代码结构有两种。

1. 为除了增删改查以外的操作创建单独的类。
比如有一个`class Resource`，除了增删改查外还有list，eat的操作。那么要分别建两个类，`class ResourceList`和`class ResourceEat`。这种方法的一个弱点是一个资源的类会比较多，一种方案是将所有相同资源的类放在同一个文件中，如果一个文件中内容太多，就需要为这个资源创建文件夹，创建对应的module。

2. 给POST添加一个ACTION query参数。
如`/resource/action?create`为创建操作，在`on_post` method里根据不同方法调用对应的功能。调用的参数放在`POST`的body。 这种方法的问题在于可能使的这个资源的调用方式和其他资源的不一致。因为`POST`方法一般直接被用来创建资源（比如表单提交），而不会另作他用。

Falcon比较适合用作纯粹的API服务器, 如果对服务器还有其他用途，比如web页面展示还是选择Django和Flask比较合适。这两种框架的开发时间也会少很多。因为使用Falcon很多功能都需要自己完成，所以能对web服务器实现的学习有一个较大的提高。
