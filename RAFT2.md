---
title: hashicorp raft 代码分析2 - Workflow
date: 2019-01-31
tags: ["golang", "raft"]
---

## 简介

[上一篇文章](https://qingsblog.com/post/raft1/)介绍了Raft的基本概念以及选主过程，这篇文章分析raft的主要工作流程。

## Raft工作流程

首先定义下客户端（Client)，客户端指使用raft库的程序。下图为Raft Apply一个命令时的主要流程，图中省略了大部分的Response示意，比如像步骤B中Log存储到LogStore后的返回就没有画出。错误处理也不在这个部分做分析，因为可能出现的错误情况太多，后面会单独整理。

```
+----------------Host A----------------------------------------+             +--Host B---+
| +--------+          +---------------+         +-----------+  |             |Follower 1 |
| |        |          |Raft(Leader)   |         |LogStore   |  |             |           |
| |        |---(A)--->|               |---(B)-->|           |  |      +----->|           |
| |        |          |               |         |           |  |      |      |           |
| |        |          |               |         +-----------+  |      |      |           |
| |        |          |               |                        |      |      +-----------+
| |        |          |               |                        |      |                   
| |        |<--(A.1)--|               |         +-----------+  |      |      +--Host C---+
| |Client  |          |               |---(C)-->|Replicator |  |      |      |Follower 2 |
| |        |          |               |         |           |--+--(D)-+----->|           |
| |        |          |               |         |           |  |      |      |           |
| |        |          |               |         +-----------+  |      |      |           |
| |        |          |               |                        |      |      |           |
| |        |          |               |         +-----------+  |      |      +-----------+
| |        |          |               |         |FSM        |  |      |            .      
| |        |<--(F)----|               |---(E)-->|           |  |      |            .      
| |        |          |               |         |           |  |      |            .      
| +--------+          +---------------+         +-----------+  |      |      +--Host N---+
|                                                              |      |      |Follower N |
+--------------------------------------------------------------+      |      |           |
                                                                      +----->|           |
                                                                             |           |
                                                                             |           |
                                                                             +-----------+
```

上图中各过程的说明如下：

* A. Client通过Raft.Apply提交一个Cmd到raft.Raft
* A.1. Raft.Apply是个异步函数，提交的命令不会被立即执行。Cmd提交后返回raft.ApplyFuture，可以使用raft.ApplyFuture同步获得命令执行结果。
* B.1. Raft生成一个新的raft.Log对象，并将Cmd放入这个对象的Data属性中。接着将Log放入Raft.applyCh channel中。
* B.2. 在Raft.leaderLoop中所有pending在Raft.applyCh上的Log被放入Raft.leaderState.inflight链表中(不同与调用Rafft.Apply的go routine), Logs被存入raft.LogStore。
* C. 异步通知Replicator有新的Logs可以被复制到Followers。
* D. Leaderd的Replicator发送AppendEntries RPC给所有followers。RPC Request中包含Leader最新的commit id，follower收到这个request后如果发现已经Apply的log id落后于Leader中最新的commit id，那么这个Follower会从自己的LogStore中取出这些log并Apply。而RPC中包含的Log会被存到Follower的LogStore中。
* E. 如果有半数以上Follower的AppendEntries返回成功，也就是这个Log commit成功了，那么这个Log可以被Apply到FSM。Apply的过程大致是将Log发送到FSM的channel，FSM的后台goroutine取出这个Log后调用raft.FSM.Apply。raft.FSM是一个接口，用于Raft跟FSM交互。具体怎么解释和处理这个命令由FSM实现者自己定义。
* F. 如果Client调用了raft.ApplyFuture.Error后调用raft.ApplyFuture.Response，FSM后台goroutine将raft.FSM.Apply的结果反馈给Client。

如上所述客户端发来的Cmd，会以Log的形式被封装并复制到所有Raft节点上。Log.Data用以存储客户端发送的Cmd。raft.LogStore是用来存储和查询Log的接口。Log和LogStore的定义如下：

```go
// Log entries are replicated to all members of the Raft cluster
// and form the heart of the replicated state machine.
type Log struct {
	// Index holds the index of the log entry.
	Index uint64

	// Term holds the election term of the log entry.
	Term uint64

	// Type holds the type of the log entry.
	Type LogType

	// Data holds the log entry's type-specific data.
	Data []byte
}

// LogStore is used to provide an interface for storing
// and retrieving logs in a durable fashion.
type LogStore interface {
	// FirstIndex returns the first index written. 0 for no entries.
	FirstIndex() (uint64, error)

	// LastIndex returns the last index written. 0 for no entries.
	LastIndex() (uint64, error)

	// GetLog gets a log entry at a given index.
	GetLog(index uint64, log *Log) error

	// StoreLog stores a log entry.
	StoreLog(log *Log) error

	// StoreLogs stores multiple log entries.
	StoreLogs(logs []*Log) error

	// DeleteRange deletes a range of log entries. The range is inclusive.
	DeleteRange(min, max uint64) error
}
```

## 异步机制实现

Raft的工作流程中使用了大量异步调用，一部分使用raft.Futrue而另一部分则使用异步通知机制。

### Future

Future一般作为一个需要长时间处理函数的返回结果。调用者获得Future时真正的操作可能还没结束，只有在等待一段时间后才能获得调用结果，所以这个接口名字叫Future。因为调用者得到Future后已经拿回了程序的控制权，所以可以做一些不依赖于调用结果的处理。raft.Future接口只需要实现一个函数，如下所示。

```go
// Future is used to represent an action that may occur in the future.
type Future interface {
	// Error blocks until the future arrives and then
	// returns the error status of the future.
	// This may be called any number of times - all
	// calls will return the same value.
	// Note that it is not OK to call this method
	// twice concurrently on the same Future instance.
	Error() error
}
```
调用者在得到Future的一个实例后可以选择在适当的时候调用Future.Error, Future.Error会block调用者直到操作执行完毕。golang中实现Future比较简单的一个思路是，在Error函数中读取一个channel，因为channel为空所以调用者会被阻塞。操作结束后将操作的结果放入channel中，如果没有出错就放一个nil即可。raft.deferError就使用了这种实现方式。

如果除了Error外还有其他结果需要检查就可以将Future内嵌到其他接口中，然后增加对应的函数。因为获取这些结果要等待函数执行完成，所以要跟调用者约定：只有在Error返回之后才能调用对应的函数。比如raft.IndexFuture,raft.ApplyFuture都有这样的约定，接口中Error之外的函数都要在Error返回后调用。

### 异步通知机制

raft中通过异步通知机制通知Replicator复制Log到Followers。所有没有被commit的Log会被放在一个名为inflight的list中。inflight每增加一个新的Log就会给通知channel发送一个struct{}{}对象，如果当前无法处理这个信号，那么这次通知就会被丢弃（select/case/default 可以实现这种机制）。如果工作go routine正好空闲，接收到信号后，他会将inflight中的所有Log放入一个队列中，然后一起发送到Followers。

我本来在想为什么不使用bufferred channel来实现这种异步机制呢？但是仔细一想bufferred channel虽然在实现上会简单很多，但是还是会对客户端照成一定的阻塞。当buffer满的时候，如果工作go routine无法及时处理channel中的log就会造成客户端阻塞。

## 总结

Raft的工作过程主要是Log的分发,存储,应用，由于这个过程中还应用了大量的异步机制，本文中也介绍了两种异步机制的实现方式。