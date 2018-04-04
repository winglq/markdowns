title: K8S
author: qing
date: 2018-03-29
description: K8S
tags:
category:
acl: 00

# K8S代码分析

## K8S的<a name="version">版本</a>

根据version.go中parse函数可以得出K8S的版本由以下几个部分注册

* 开头的字母`v`
* 版本号 如果使用[SemVer](https://semver.org/lang/zh-CN/)规范，版本号应该有三部分，并使用`.`分割，否者这部分应该由三个或以上部分组成。
* 其他 用于标识版本发布前的信息比如alpha,beta版，和build的元数据。版本发布前信息以`.`分隔各个部分。

## Golang

`reflect.Type` 是对一个`type`的抽象接口，比如这个接口中包含`Name() string`方法，使用这个方法就可以知道对象的`type`名称。
`reflect.Value` 是对一个值的抽象，比如有一个`interface{}`对象时无法知道真正的值，可以使用`reflect.Value`提供的接口去获取真正的值。

## API

### Scheme

Scheme defines methods for serializing and deserializing API objects, a type registry for converting group, version, and kind information to and from Go schemas, and mappings between Go schemas of different versions. A scheme is the foundation for a versioned API and versioned configuration over time.
从上面这段代码中的注释可知Scheme是K8S对一个对象序列化和反序列化时用到的数据结构。在Scheme中Kind用来表示对象的type。

#### Scheme的初始化

使用Scheme的nameFunc创建一个`Converter`, `Converter`在不知道实际type的情况下，将一个type转换成另一个type。添加转换函数。添加输入类型映射函数(`JSONKeyMapper`)。

#### Conversion

Specifically, conversion provides a way for you to define multiple versions of the same object. You may write functions which implement conversion logic, but for the fields which did not change, copying is automated. This makes it easy to modify the structures you use in memory without affecting the format you store on disk or respond to in your external API calls.
从上面这段注释可以了解到`package conversion`可以对同一个对象定义多个版本，并在这些版本之间做转换。



## Cobra
K8S使用Cobra做为命令行框架。

## kubadm init命令
用于初始化一个K8S控制节点(master)。

init命令会初始化一个`MasterConfiguration`对象，并调在Scheme中查找是否有这个type的默认函数，如果有的话会调用这些函数用于为这个对象设置默认值。
init命令会根据传入参数设置`SelfHosting`, `StoreCertsInSecrets`, `HighAvailability`, `CoreDNS`, `DynamicKubeletConfig`, `Auditing`这些feature是否被启用, 比如`SelfHosting=1,StoreCertsInSecrets=0`. 

### Features Gates
Feature gates are a set of key=value pairs that describe alpha or experimental features.


Feature的数据结构

    :::go
    type Feature struct {
    	utilfeature.FeatureSpec            //控制默认是否打开,和发布前版本号
    	MinimumVersion   *version.Version  //最小需要版本号
    	HiddenInHelpText bool
    }

关于K8S的版本规则可以参见[版本](#version)。
