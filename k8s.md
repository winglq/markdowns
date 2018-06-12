title: K8S代码分析
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
从上面这段注释可以了解到`package conversion`可以对同一个对象定义多个版本，并在这些版本之间做转换。同一个数据结构在两个不同版本之间转换时可能出现缺少或增加新的字段，或者字段的名称类型发生了变化，所以`conversion`包主要就是用来处理这些不一致情况下的结构转换。`FieldMatchingFlags`定义了是从目标端还是源端开始扫描所有字段，如果对端字段不存在时做何种处理。

    :::go
    type Converter struct {
    	conversionFuncs          ConversionFuncs // 转换类型到转换函数的map, key为源和目标type，value为转换函数
	structFieldDests map[typeNamePair][]typeNamePair // 在转换过程中目标结构中的字段可以由源结构中的哪些(由一个list组成)字段(字段名称，和类型）转换
	structFieldSources map[typeNamePair][]typeNamePair // 同上一个字段做反方向处理
	inputFieldMappingFuncs map[reflect.Type]FieldMappingFunc //保存将一个map映射为struct的函数
    }

Convert函数用于在两个对象之间做转换，`src`, `dest`分别是源端和目标端对象的指针，`flags`表明发生转换的方向，从源端还是目标端开始扫描。`meta`可以提供额外参数给Customer的convert函数。转换首先使用`c.genericConversions`中注册的转换函数，如果任何一个成功转换就直接返回，否则就调用`c.doConversion`。该函数会调用各个Customer的convert函数，知道成功。

    :::go
    func (c *Converter) Convert(src, dest interface{}, flags FieldMatchingFlags, meta *Meta) error {
    	if len(c.genericConversions) > 0 {
    		// TODO: avoid scope allocation
    		s := &scope{converter: c, flags: flags, meta: meta}
    		for _, fn := range c.genericConversions {
    			if ok, err := fn(src, dest, s); ok {
    				return err
    			}
    		}
    	}
    	return c.doConversion(src, dest, flags, meta, c.convert)
    }

由于`struct`中字段可能是新的`struct`或者指针或者`map`等复杂数据结构，所以在转换的过程中会有stack的概念，每遇到一个新的复杂数据结构式转换就需要将这些结构压入stack。`Scope`中有两个stack分别用于记录源端和目标端需要转换的字段。

## Cobra
K8S使用Cobra做为命令行框架。

## kubadm init命令
用于初始化一个K8S控制节点(master)。

init命令会初始化一个`MasterConfiguration`对象，并调在Scheme中查找是否有这个type的默认函数，如果有的话会调用这些函数用于为这个对象设置默认值。
init命令会根据传入参数设置`SelfHosting`, `StoreCertsInSecrets`, `HighAvailability`, `CoreDNS`, `DynamicKubeletConfig`, `Auditing`这些feature是否被启用, 比如`SelfHosting=1,StoreCertsInSecrets=0`。
根据初始化文件配置API `Init`对象，如果没有配置就使用默认值。IP和Port等信息则通过动态获取。需要部署的组件版本在`KubernetesVersion`中配置，可以是具体的版本号，也可以是版本label，kubadm会根据label获得正确的版本号。要部署的版本号需要大于程序内定义的最小版本号。如果没有指定[Token](https://kubernetes.io/docs/reference/setup-tools/kubeadm/kubeadm-token/)的话，会生成一个新的Token。
`Init`对象生成之后，程序开始绑定各个命令行参数，默认值为cfg中的值。

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

### kubctl plugins

[Extend kubectl with plugins](https://kubernetes.io/docs/tasks/extend-kubectl/kubectl-plugins/)

## k8s一些默认配置

* 默认Port 6443

## 其他

*  /proc/net/route 记录着所有的Gateway和active ip等信息，`route -n`也是从这个文件中读取。
* CIDR(Classless Inter-Domain Routing) notation example: 192.168.1.15/24
