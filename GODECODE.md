---
title: Golang 通过reflect.Type和Json二进制流Unmarshal对象
date: 2018-06-14
---


使用reflect.Type和Json字符串Unmarshal出一个reflect.Value，代码如下:

```go
package main

import (
	"fmt"
	"encoding/json"
	"reflect"
)

type User struct {
        Name string `json:"name"`
}

func BuildFromJson(T reflect.Type, s string) {
        v := reflect.New(T)
        e := json.Unmarshal([]byte(s),v.Interface())
        if e != nil {
              fmt.Println(e)
        }
 
        u := v.Interface().(*User)
        fmt.Println(u.Name)
}

func main() {
        u := User{Name: "xx"}
        r, e := json.Marshal(u)
        if e != nil {
              panic(e)
        }
    	fmt.Println(string(r))
    	BuildFromJson(reflect.TypeOf(u), string(r))
}
```

[Golang playgroud](https://play.golang.org/p/9plGj_-DeEJ)
