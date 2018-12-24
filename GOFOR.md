---
title: 在golang for循环中使用goroutine引起的问题及分析
date: 2018-12-24
tags: [golang]
---

今天无意中搜到[Handling 1 Million Requests per Minute with Go](http://marcio.io/2015/07/handling-1-million-requests-per-minute-with-golang/)这篇博客，在评论区里有人发现文章中的代码有一个隐蔽的bug。在平时coding中这个问题比较容易被忽略，所以在此记录和分析下。
有问题的模拟代码如下。

```go
package main

import (  
    "fmt"
    "time"
)

type Payload struct {
    data int
}

func (p *Payload) UploadToS3() error {
    fmt.Println("payload data:",p.data)
    return nil
}

func main() {  
    payloads := []Payload{Payload{1},Payload{2},Payload{3},Payload{4}}

    for _,payload := range payloads {
        go payload.UploadToS3()
    }

    time.Sleep(4 * time.Second)
    }
```

运行上面的代码，输出都为`payload data: 4`，而不是4个不同的值。这是由于for只在第一个Iteration声明payload，后续的迭代只是改变playload的值。因为for循环所在的goroutine在4个UploadToS3 goroutine之前运行，所以当UploadToS3开始被调用的时候playload的值已经确定，即slice的最后一个Payload。因为UploadToS3的receiver是`*Payload`，所以golang编译器会在遇到`payload.UploadToS3()`的时候自动使用`&payload`去调用UploadToS3。因此4个goroutine取到的结果是同一个。如果UploadToS3()的receiver是Payload就不会有这个问题，因为编译器会复制一个临时的non-point receiver(Payload)传递给method(UploadToS3)。
当然由于goroutine运行顺序的不确定性，也有可能出现不是输出都为`payload data: 4`的结果。

这个问题涉及的知识点比较多，忽略任何一点都可能忽略这个bug。
