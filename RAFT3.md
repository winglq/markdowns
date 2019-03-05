---
title: hashicorp raft 代码分析3 - Snapshot
date: 2019-03-05
tags: ["golang", "raft"]
---

## 简介

[上一篇文章](https://qingsblog.com/post/raft2/)介绍了Raft的主要工作流程。
这篇文章主要介绍raft的snapshot机制。Raft中引入Snapshot的目的是为了避免存储过多的Log。比如集群维护一个状态Status，他的可取值是Running和Stopped。假设集群运行的过程中Status经历了Stopped->Running->Stopped->Running这几个状态。如果在最后的Running状态有个新的节点加入到集群，在没有做snapshot的情况下就要发送3次Log以使新节点的同步到Running的状态。如果在最后一个Running状态后，集群做了一次Snapshot，那新加节点只需要安装最新的snapshot，明显数据量会少很多。

## 生成Snapshot

默认情况下Raft库每120s检查一次是否需要snapshot，默认设置下如果当前log id比上次成功的snapshot的log id大于8192时raft开始执行snapshot操作。下图为Raft库生成snapshot的流程。
                                                                                                    
```
+--------------------+                                                                              
| Raft lib           |                                                                                       
|   +--------------+ |    +----------------------+
|   |              | |    |Lib user              |
|   |              | |    |    +---------------+ |
|   |runSnapshots  | |    |    |SnapshotStore  | |
|   |go routine    |-+-D--+--->|Create         | |
|   |              | |    |    +---------------+ |
|   |              | |    |    +---------------+ |
|   |              |-+-E--+--->|FSMSnapshot    | |
|   |              | |    |    |Persist        | |
|   |              | |    |    +---------------+ |
|   |              | |    |    +---------------+ |
|   |              |-+-E--+--->|SnapshotSink   | |
|   |              | |    |    |               | |
|   +--------------+ |    |    +---------------+ |
|          |  |      |    |                      |
|          A  C      |    |                      |
|          |  |      |    |                      |
|   +--------------+ |    |    +---------------+ |
|   |runFSM        |-+-B--+--->|FSM interface  | |
|   |go routine    | |    |    |Snapshot       | |
|   +--------------+ |    |    |               | |
|                    |    |    +---------------+ |
+--------------------+    +----------------------+
```

* A. Snapshot操作由runSnapshot go routine发起（最新log id和上次生成snapshot时的log id已经超过threshhold），这个go routine将Raft.reqSnapshotFuture请求发送给runFSM go routine
* B. runFSM go routine调用FSM.Snapshot获得raft.FSMSnapshot接口
* C. raft.FSMSnapshot接口返回给runSnapshot go routine
* D. runSnapshot go routine使用raft.SnapshotStore接口创建一个raft.SnapshotSink接口
* E. raft.FSMSnapshot.Persist以raft.SnapshotSink为参数，将数据持久化到存储介质上

raft.FSM, raft.SnapshotStore, raft.FSMSnapshot, raft.SnapshotSink都由使用者实现，因此由Raft库的使用者实际负责保存数据。这里的数据就是raft集群维护的各节点一致的状态，比如简介例子中的Status状态。这里再简单介绍下几个接口函数的作用。SnapshotStore.Create返回一个FSMSnapshot接口，FSMSnapshot.Persist用于将raft集群维护的状态序列化，然后将序列化后的数据写入SnapshotSink提供的IO（这里使用io.Writer）接口中。

Snapshot生成后，snapshot之前的Log会被删除，这样Log的数量就不会无限增长。

## 从Snapshot恢复数据

当出现节点关闭一段时间重新打开或者新节点加入集群时都需要从Snapshot恢复数据。当然如果在最新snapshot之后有其他Log，这些Log也会被Apply到待恢复的节点上。数据恢复主要有以下两种方式。

### 从当前节点的Snapshot中恢复数据

要使用本地Snapshot恢复数据，首先使用SnapshotStore.List获得所有snapshot，然后从最新的snapshot开始遍历，调用FSM.Restore恢复当前snapshot的数据，如果恢复失败则尝试下一个snapshot。

### Leader发送InstallSnapshot RPC恢复follower状态

如果有新的节点加入集群，可以通过InstallSnapshot安装leader上最新的snapshot进而同步数据。

Follower收到InstallSnapshot的RPC request后，首先将Snapshot存储到SnapshotStore，存储的方法是将request body的内容直接复制到raft.SnapshotSink。然后将raft.restoreFuture发送到Raft.fsmMutateCh channel。runFSM go routine负责将raft.restoreFuture从channel中取出，再根据restore的snapshot id，打开对应的snapshot(这里是刚存储的snapshot)，然后调用FSM.Restore恢复snapshot。FSM.Restore由库的使用者实现，因此如何反序列化这个snapshot完全由实现者自己定义。

## 总结

其实生成和恢复snapshot的过程就是一个序列化和反序列化数据的过程，当然这个过程中要考虑的就是这些数据存储的位置。hashicorp的Raft库定义了比较清晰的接口用于完成序列化，反序列化和存储接口，这给使用这个库带的人来了很大的方便。