---
title: GoLang reflect.Kind小记
date: 2018-07-06
tags:
draft: true
---


在做go的动态类型转换的时候发现有时候Type和Value的Kind是不同的，我原来理解他们应该是相同的。于是写了下面的验证程序。因为reflect.New会返回Value，但这个Value实际是实际Type的一个对象的指针，所以Kind是Ptr。而他的Type中没有Value中对应的信息，只知道这是个Value。reflect.Value是个struct。

```go
package main

import (
	"fmt"
	"reflect"
)

func test(vi interface{}) {
	vt := reflect.TypeOf(vi)
	v := reflect.ValueOf(vi)
	fmt.Println(fmt.Sprintf("Type kind %v, Value kind %v, Type Elem %v, Value Name %v", vt.Kind(), v.Kind(), vt.Elem().Kind(), vt.Name()))
	ppv := reflect.New(vt)
	ppvt := reflect.TypeOf(ppv)
	fmt.Println(fmt.Sprintf("Newed Type kind %v, Newed Value kind %v, Newed Value Name %v", ppvt.Kind(), ppv.Kind(), ppvt.Name()))
}

func main() {
	s := "hello world"
	test(&s)
}
```

输出如下:

```
Type kind: ptr, Value kind: ptr, Type Elem: string, Value Name:
Newed Type kind: struct, Newed Value kind: ptr, Newed Value Name: Value
```
