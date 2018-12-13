---
title: go-micro 代码分析笔记
date: 2018-11-29
draft: true
---

## 参数传递

go micro中参数传递主要有两种方式Option和Context。

### Option

Option的定义如下，要设置某个选项时，实际是返回一个修改Options的函数。

```go
type Option func(*Options)
```

比如Options中有一个属性是Name，要修改这个属性需要定义下面这样的Option

```go
func Name(n string) Option {
        return func(o *Options) {
                o.Name = n
        }
}
```

这么做的一个好处是可以大大减少参数传递时的变量数量，只需要一个Option的list就足够了。参数的使用者只需遍历所有的Option并调用即可。不需要每个参数都进行一次赋值。

这种方式很像一个build模式，Options是一个很大的结构体，属性较多，可以用多个build的函数改变不同的属性。

### Context

go-micro中另外一种外一种参数传递方法是使用Context。Context可以做为Options的一个属性。Option做为接口共有的参数，而Context里存储的是具体实现自己特有的参数。


## Service 

service是go-micro中最上层的一个概念，一个service包括两部分，Client和Server。在Client端Service的作用是创建客户端，以便给服务端发送请求(rpc/broker message)。在Server端Service用来管理Server的启动和关闭。Service还负责定时将自己注册为服务，以便客户端能够发现自己。

### Client

protoc生成的代码中会有一个service名称对应的struct，这个struct中有一个client.Client成员。client.Client是一个接口，作用是创建一个RPC请求，然后将这个request交给Transport，由Transport负责发送这个请求，最后接收返回内容。在Client和Transport都可以对自己的内容编码，Client代码如下。

```go
// Client is the interface used to make requests to services.
// It supports Request/Response via Transport and Publishing via the Broker.
// It also supports bidiectional streaming of requests.
type Client interface {
	Init(...Option) error
	Options() Options
	NewMessage(topic string, msg interface{}, opts ...MessageOption) Message
	NewRequest(service, method string, req interface{}, reqOpts ...RequestOption) Request
	Call(ctx context.Context, req Request, rsp interface{}, opts ...CallOption) error
	Stream(ctx context.Context, req Request, opts ...CallOption) (Stream, error)
	Publish(ctx context.Context, msg Message, opts ...PublishOption) error
	String() string
}
```

client.Client.Call和client.Client.Stream的区别是收到response后，谁负责关闭连接。Call由client.Client负责关闭连接，而Stream由调用者负责关闭。默认情况下无论Call和Stream都不会关闭Transport层的连接，因为client.Client的默认实现rpcClient使用conn pool来管理所有连接。close一般只是将conn放回pool里。

rpcClient是Client的一个实现。rpcClient.Call函数首先检查ctx中有没有设置Timeout时间并设置。接着检查当前时间是否已超过超时时间。然后根据重试次数调用对应的api。可以callOpts中设置CallWrapper用于在rpcClient.call之前做操作，api的node通过Select获取的Next函数获得。Select在注册的服务中根据策略(Random或Roundrobin)选择node，默认的注册服务是consul。selector接口代码如下:

```go
// Selector builds on the registry as a mechanism to pick nodes
// and mark their status. This allows host pools and other things
// to be built using various algorithms.
type Selector interface {
	Init(opts ...Option) error
	Options() Options
	// Select returns a function which should return the next node
	Select(service string, opts ...SelectOption) (Next, error)
	// Mark sets the success/error against a node
	Mark(service string, node *registry.Node, err error)
	// Reset returns state back to zero for a service
	Reset(service string)
	// Close renders the selector unusable
	Close() error
	// Name of the selector
	String() string
}
```

rpcClient.call将ctx中的metadata封装在messsage中的Head中。然后在connection pool中取出对应address的connection。根据request的content type选择对应的codec，go-micro提供json和protobuf两种codec，protobuf是默认codec。Codec接口定义如下:

```go
// Codec encodes/decodes various types of messages used within go-micro.
// ReadHeader and ReadBody are called in pairs to read requests/responses
// from the connection. Close is called when finished with the
// connection. ReadBody may be called with a nil argument to force the
// body to be read and discarded.
type Codec interface {
	ReadHeader(*Message, MessageType) error
	ReadBody(interface{}) error
	Write(*Message, interface{}) error
	Close() error
	String() string
}
```

rpcStream的两个成员codec(client.clientCodec)，request(client.Request)。

Specific Service.Request -> client.Request(client.rpcRequest) -> body

```
transport.Message
  codec.Message
    client.request
    client.Request
      service.Request
jsonrpc.clientRequest
  codec.Message
    client.request
    client.Request
      service.Request
```

```go
type rpcRequest struct {
	service     string
	method      string
	contentType string
	request     interface{}  // Specific Service.Request
	opts        RequestOptions
}
```


rpcPlusCodec


### Server

Server端生成的代码作用是提供一个所有method的handler接口，实现代码需要实现所有接口，生成代码会把Hander传给Server，以便对应的method可以route到正确的handler上。

server.Server的默认实现是server.rpcServer，使用的默认Transport是http实现的Transport。

## Service 代码组织

### Error

go micro的error使用如下数据结构，Id可以用api.service.method来表示，比如api.hotel.rates. Code一般使用HTTP Status Code， Status一般用HTTP Status Text表示。Detail为自定义具体错误内容。

```go
type Error struct {
	Id     string `json:"id"`
	Code   int32  `json:"code"`
	Detail string `json:"detail"`
	Status string `json:"status"`
}
```

### Trace

Trace的包是golang.org/x/net/trace，不是golang自带的Trace包。
Trace的family可以使用api.version来表示，比如api.v1。Title使用service.method，比如Hotel.Rates。可以将trace加到method的context中。

```
ctx = trace.NewContext(ctx, tr)
```

### Metadata

Metadata 相当于HTTP的Header，可以在不同service之前专递信息。比如Authorization/traceID就可以放在Metadata中。Metadata定义如下。

```
// Metadata is our way of representing request headers internally.
// They're used at the RPC level and translate back and forth
// from Transport headers.

type Metadata map[string]string
```
