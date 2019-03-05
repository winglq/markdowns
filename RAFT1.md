---
title: hashicorp raft 代码分析1 - 选举leader
date: 2019-01-02
tags: ["golang", "raft"]
---

## Raft介绍

Raft是一种一致性算法，一致性算法的作用是将几个机器看成一个整体，即使其中的某台或者某几台机器发生故障，整个系统仍能正常工作。在Raft出现之前Paxos几乎是一致性算法的代名词。Ceph就使用Paxos算法同步各个节点的状态。虽然Raft和Paxos的作用相同，但是Raft算法更容易理解和实现，可以给学习一致性算法带来极大的方便。hashicorp的raft实现在各个开源项目中有着广泛应用，比如influxdb，consul。这里分析的代码是基于commit id: `9c733b2b`。

Raft只有三种节点：follower，candidate和leader。leader节点接收来自client的请求，接收到的请求以Log形式会被储存在本节点并分发到follower节点，如果整个集群中有半数以上节点已经存储了这个Log Entry，那么这个Log Entry就算提交（commit）成功了。Commit成功的Log Entry会被Apply到一个FSM（有限状态机）。FSM的任何一个状态都能通过Apply之前所有的Log Entry而获得，所以即便有节点因为故障而离开集群一段时间，在这个节点恢复时可以Apply它缺失的Log Entry而重新与其他节点保持一致。如果无限制保存Log Entry会需要无限大的磁盘空间，Raft提供Snapshot机制解决这个问题。Snapshot将当前FSM中需要保护的状态保存下来，然后删除这个状态之前的Log Entry，如果出现follower短暂故障而恢复，leader只需要先让follower install leader的最新snapshot，然后再将这个snapshot之后的Log Entry发送给这个follower就可以使其状态与leader保持一致。

完整的Raft协议可以查看[这里](https://raft.github.io/raft.pdf)。

## 选举Leader过程

Raft集群中leader负责接收客户端发来的请求并把这些请求保存和分发到follower，作用至关重要，是集群和外界通信的唯一节点。各个Raft节点通过投票选出leader，每个节点在一个Term中只能投一票，获得半数以上票数的node胜出。这也意味着一个集群中最好有奇数个node。下图为集群启动后的选举过程。

```
+----1----+      +-----2-----+      +----3-----+
| Follower|      | Follower  |      | Follower |
+---------+      +-----------+      +----------+
                       | (A)
+----1----+      +-----2-----+      +----3-----+
|         |<-(B)-|           |-(B)->|          |
| Follower|      | Candidate |      | Follower |
|         |-(C)->|           |<-(C)-|          |
+---------+      +-----------+      +----------+
                       |(D)
+----1----+       +----2-----+      +----3-----+
|         |<-(E)--|          |-(E)->|          |
| Follower|       |  Leader  |      | Follower |
|         |-(F)-->|          |<-(F)-|          |
+---------+       +----------+      +----------+
```

* 系统启动时node 1，2，3都是follower。
* A: node 2的计时器第一个timeout，因为系统现在没有leader所以node 2在设置时间内没有跟其他节点通讯过。node 2将自己的角色变成Candidate。
* B: node 2首先将自己的Term加1，并且给自己投上一票，然后将这个Term 放到RequestVote的RPC请求中，发送给node 1和node 3。
* C: node 1和node 3收到RPC请求后，检查以下几个条件是否都成立，如果是那么他们就可以把自己的一票投给node 2，并且更新自己的Term为最新值。
     * C.1 自己没有leader。
     * C.2 自己当前Term小于收到的RPC Term。
     * C.3 在这个Term中没有把票投给其他node。
     * C.4 自己的Index小于RPC请求中的Index。
* D: node 2根据步骤C中的response检查自己的得票总数，如果超过半数就把自己的角色变成Leader。(如果在规定时间内没有得到足够票数的话，则启动新一轮投票。)
* E: node 2成为Leader后开始发送AppendEtries RPC给node 1和node 3。
* F: node 1和node 3接收到AppendEtries后记录node 2为最新的leader，返回RPC response给node 2。

这个过程中大家可能会有疑问**启动过程中会不会出现几个node同时变成Candidate呢？**这种情况是可能的，但是很少发生。因为从follower到candidat的时间是由两部分相加组成的，1. 一个可以设置的固定时间， 2. 一个随机时间。由于第二部部分的存在同时变成candidate的可能性很小。当然这也不代表着不可能，如果candidate在等待时间内无法得到足够的票数就会进入下一轮投票。由于candidate等待投票结果的时间也是由一个固定时间和一个随机时间相加而得，所以同一时间内进入下一轮投票的可能性也很小。这两个时间可以保证Raft集群可以很快选举出一个Leader。

## 选举Leader的代码流程

raft.Raft是这个库的主要数据结构记录着raft的各个节点，自己当前状态等。raft.Raft创建过程中会启动三个主要的后台go routines，分别是

* run: 根据自己的角色(Leader， Candidate， Follower)执行对应的操作。
* runFSM: 调用FSM接口
* runSnapshots: 执行快照操作

选举Leader主要集中在(*Raft).run中。三种角色分别对应函数(*Raft).runFollower,(*Raft).runCandidate,(*Raft).runLeader。

三种角色共用一个（*Raft).processRPC函数用于处理RPC，RPC就三种类型：Raft.AppendEntriesRequest, Raft.RequestVoteRequest, Raft.InstallSnapshotRequest。选举Leader过程会使用前两个RPC请求。下面简单介绍RPC的调用流程。

## RPC调用流程

RPC调用的过程如下图所示。

```
+-------------------Node 1 (Leader)---------------+   +--------------------Node 2-------------------------+
|+--------------+            +--------------+     |   |  +--------------+            +--------------+     |
||Raft          |            |Network       |     |   |  |Network       |            |Raft          |     |
||              |            |Transform     |     |   |  |Transform     |            |              |     |
||              |            |              |     |   |  |  +-----------|            |------+       |     |
||              |---RPC----->|              |-----+---+->|  |consumeCh  |---RPC----->|rpcCh |       |     |
||              |            |              |<----+---+--|  +-----------|            |------+       |     |
||              |            |              |     |   |  |              |            |---------+    |     |
||              |<-Respond---|              |     |   |  |              |<-Respond---|         |    |     |
||              |            |              |     |   |  |              |            |---------+    |     |
|+--------------+            +--------------+     |   |  +--------------+            +--------------+     |
+-------------------------------------------------+   +---------------------------------------------------+
```
Node1（Leader）要发送一个AppendEtries RPC给node 2需要经过如下步骤。

* Node 1的Raft构造一个raft.AppendEtriesRequest，通过Transform.AppendEntries传递给Transform。
* Transform定义了传输层的接口，比如AppendEntries就是接口的一部分。raft.NetworkTransport是基于网络传输的Transform实现。
* 传输层将RPC请求编码后通过网络将RPC请求发送给Node 2的传输层。
* Node 2的传输层将RPC请求放入consumeCh（在Raft中这个channel的变量名为rpcCh）channel中。
* Node 2的Raft从rpcCh channel中取出请求，处理后调用(*RPC).Respond将请求返回给Node 2的传输层。
* RPC的response最终返回到Node 1的raft.Raft。

## 总结

本文是Hashicorp raft协议实现分析的第一部分，主要分析了Raft集群中Leader的选举，还简单介绍了RPC的调用过程。
