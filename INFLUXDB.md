---
title: Influxdb 代码分析
date: 2018-07-26
---


## 版本         
influxdb 0.11.1 

## 目录结构
TSDBStore根目录: /var/lib/influxdb/data
每个database为TSDBStore根目录下的一个字母，这里称为DB目录
DB目录下的子目录为retention policy目录，这里简称为RP目录
RP目录下的子目录为Shard目录，这里简称SH目录
SH目录下有两种类型的文件夹，分别是shard(名为shard id)和WAL
Hinted Handoff目录/var/lib/influxdb/hh,其中的文件以每个node的id命名。 hinted handoff 的作用是监控所有data节点，如果data节点不正常工作，那数据可以先写在hinted handoff的queue中。 根据什么规则来分配？
Metadata 目录从配置文件读取，默认是不是？ /var/lib/influxdb/meta

每个shard都会启动一个engine实例，0.11.1中只有tsm1

## MetaClient初始化

连接到meta server获得当前cluster数据的snapshot。获取snapshot的url为。snapshot会放到当前client的Cachedata中。CacheData或者Snapshot的数据如下

```go
type Data struct {
	Term      uint64 // associated raft term
	Index     uint64 // associated raft index
	ClusterID uint64
	MetaNodes []NodeInfo
	DataNodes []NodeInfo
	Databases []DatabaseInfo
	Users     []UserInfo
	MaxNodeID       uint64
	MaxShardGroupID uint64
	MaxShardID      uint64
}
```

然后启动一个routine poll整个cluster的snapshot，并更新本地的CacheData。meta data server 端会根据index判断是否有比idnex更新的snapshot可用，如果没有的话meta data server不会返回，这个请求就会一直等待response。

    Scheme://server?index=0 # scheme 为http或https

MetaClient启动完成后会把当前节点做为一个DataNode加入到集群中去，创建操作会在每个meta server上尝试，直到成功或者达到最大尝试次数。只要在一个节点上创建成功，这个DataNode信息会同步到所有meta server上，这个由raft保证。而其他client也会收到这个更新，这个由上面提到的poll cluster data的go routine完成。Client执行命令通过发送POST请求到Meta Server，URL如下，POST的内容为具体的Command。命令返回意味着本地的CacheData已经更新。

    Scheme://server/exec

当这个DataNode加入成功后，Meta Server会分配一个ID给这个Node。Client获得meta server最新的snapshot后将这个DataNode的ID赋值给server的Node.ID属性。获取snapshot与从CacheData中获得自己的nodeID的过程是在两个go routine中进行的，所以如果创建完DataNode但是snapshot数据还没更新，那么赋值操作就不能成功。如果失败系统会不断重试，直到赋值成功。最后server的Node（包括path和ID）会被序列化到文件，以便重启是Server知道自己的ID。

## Meta service

Meta service启动会创建一个新的store用来存放meta data。

Meta service handle 主要实现4个GET方法，包括ping/lease/peers/snapshot(default case)，上面提到的client获取cluster的snapshot就是对应到这里的snapshot handler。POST方法主要就是实现exec这个方法，用于执行raft命令。

### snapshot handler

Meta Client发送snapshot命令的唯一参数是index。这个参数可以让Meta service知道client的当前index，然后meta service将自己的index和client的比较，如果自己的数据比client的新就将自己数据的一份clone发送给client。如果Meta service没有比index更新的snapshot，那么这个请求就会等待在一个名为`s.dataChanged`的channel上。如果数据发生更新，这个channel就会关闭。等待的请求继续执行，首先会对store做一次snapshot，将所有的meta service的所有数据clone一次。 

### ping handler

如果不是请求ping所有的metadata service的话，当前节点直接返回leader的名字。如果是ping所有节点，收到请求的节点会再ping其他metadata service，并比较返回的leader信息是否一致。

### lease handler

client将nodeid, 和lease的Name发给meta service，如果当前节点不是leader的话，这个请求会被重定向到leader节点，否则就返回lease给client。lease作用？

### peer handler

发送所有peers给meta client。

### exec handler

#### join

* 调用raft.AddPeer将新的Node加入到raft中。
* 调用raft.Apply，创建一个metadata node。

#### 其他命令

* 调用raft.Apply执行命令
* 返回data的最新index给client

## raft

### raft的启动

* log和stable信息存储在raft.db文件中，这个文件用boltdb管理。
* snapshot存储在filestore中。
* peerStore只存储在内存中。

metadata的store 实现了raft的FSM接口，也就是说raft要管理的状态都在store中。

### storeFSM的Apply

Apply把Log Entry中的Data cast成命令，然后执行这个命令，raft协议会保证这个命令会在每个成员上执行，因为所有成员之前的状态是一致的，那么执行同一个命令后状态应该还是一致的。所有的命令都是顺序执行的，store的mu用来保证不能同时执行两个命令。命令执行完后会用log entry的index和term更新raft_state的信息。命令作用的大部分对象是针对store.Data的，跟Client的CacheData同一个数据结构，见上。

### storeFSM的Snapshot

Snapshot返回一个`raft.FSMSnapshot`接口，这个接口实现`Persist`和`Release`功能。raft应该会在做snapshot的时候调用Snapshot，然后调用FSMSnapshot的Persist函数将数据持久化。Metadata store实现的Persist实际是将store.Data持久化。

### storeFSM的Restore

Restore的过程和Snapshot相反。首先读取之前的Snapshot内容，然后Unmashel成store.Data

## 读写流程

httpd service 在接收到POST write请求后将数据解压然后根据数据的type交给PointWriter service。

WritePoint 服务订阅订阅服务的points channel。WritePoint服务有新的写请求先写入到shard，然后就把这个point放入points这个channel中，给订阅服务处理。
写入shard的过程如下：

1. point的映射过程为先通过metadata client获得write data request的retention policy。然后根据所有point的时间和retention policy创建shard group。shard group包含所有的机器，points会根据下面的步骤hash到机器上。
2. 所有的point会放到shard id到points的mapping中去, mapping 过程如下

  写入的机器根据point的hash值然后用shard group里Shard数量求余获得。

```go
sgi.Shards[hash%uint64(len(sgi.Shards))] // sgi: Shard group info.
```

  上面的hash通过下面的代码求出。

```go
func (p *point) HashID() uint64 {
         h := fnv.New64a()
         h.Write(p.key)
         sum := h.Sum64()
         return sum
}
```

  其中`p.key`为measurement+sorted tags。所以同一series的point会被写到同一个机器上。

3. 将所有的point放入shardid->points的map中，对应每个shard启动一个go routine，把对应的point写到这个shard中去。
4. 将point写到对应shard的所有ower上。如果有replication，一个shard会有多个owner。
5. 如果shard的owner id等于PointerWriter的id说明这个Point就应该写在这个机器上。调用TSDBStore的WriteToShard。如果当前shard的owner id不等于PointerWiter的ID，PointWriter调用ShardWriter.WriteShard，将Point写入对应的owner。

  * ShardWriter.WriteShard的步骤如下：1. 在connection pool中获取一个到owner的connection。 2. 使用metaclient获取所有这个shard的shard group，如果这个shard group不存在了就丢弃这些points。3. 将Points和其他信息放入WriteShardRequest中，通过网络写入对应的owner
  * WriteToShard的步骤为: 1.从store中根据shard id获取Shard, 调用Shard的WritePoints。2.创建Points中当前Shard中没有的series和measurement的fields。3. 检查已经存在的Filed的type和Point中的数据type是否一致。如果数据类型不匹配直接返回。4.如果measurement不存在，在database index中添加新的measurement。5. 如果是新的series,在measurement建立(Tag.Key)->(Tag.Value)->(Series ID)的映射关系 6. 创建Measurement和他的Fields。7. 将Point写入Engine(TSM1)中。将Point放入mapping中，key为`key := string(p.Key()) + keyFieldSeparator + k`，其中的sep为`keyFieldSeparator = "#!~#"`。用每个的Point的field创建一个Value对象，Value本身是一个接口。然后把mapping写入TSM1的WAL中，写入文件前会先压缩数据。WAL超过10M会重新分文件。WAL写入的第一个字节为这个Entry的类型，接下来是个字节为压缩后数据的长度。

### shard group的创建

1. 根据Retention Policy名字查到对应的Retention Policy的数据结构，如下所示：

    ```go
    type RetentionPolicyInfo struct {
    	Name               string
    	ReplicaN           int
    	Duration           time.Duration
    	ShardGroupDuration time.Duration
    	ShardGroups        []ShardGroupInfo
    	Subscriptions      []SubscriptionInfo
    }
    ```

2. 在得到的RetentionPolicyInfo中遍历所有ShardGroups，查找Point的插入时间在shard group的开始时间和结束时间之间的那个shard group。
3. 如果存在这样的shard group，不需要创建新的shard group，否则进入第四部
4. 调用raft发送创建新的shard group的命令

   * 获得当前副本数量 replicaN
   * 根据副本数量计算出单个shard group的shard数`shardN := len(data.DataNodes) / replicaN`
   * 新建新的shard group，开始和结束时间根据Retention Policy计算获得
   * 创建shardN个新的Shard
   * 随机挑选一个DataNode开始给每个Shard分配replicaN个owner。
   * Shard group的duration根据对应的policy计算，代码如下。rp duration在半年以上的sg的duration为一周，大于两天的为一天，两天以下的为一小时。

    ```go
    func shardGroupDuration(d time.Duration) time.Duration {
            if d >= 180*24*time.Hour || d == 0 { // 6 months or 0
                    return 7 * 24 * time.Hour
            } else if d >= 2*24*time.Hour { // 2 days
                    return 1 * 24 * time.Hour
            }
            return 1 * time.Hour
    }
    ```


5. 因为client调用执行raft的命令会等待到命令执行成功并且CacheData被更新，所以命令执行完后重新查找shard group info并返回。

### WAL -> db file(tsm)

1. 将WAL segment file全部读入到cache中。

    * 读取类型和长度，根据长度读取entry，解压entry。
    * Unmashal entry,根据数据类型将数据一个个取出。因为每个数据结构(int64, float64...)都有自己的Marshel和UnMarshel方法，读取二进制数据后对其进行UNMarshall。
    * 将 entry 添加到cache中，如果新加points不有序，或者新point的时间早于cache中时间,设置需要排序标志位。
2. 将Cache中的数据做一次Snapshot，然后把Snapshot放入Flush队列，Flush队列中的数据会被写入到tsm文件中。做Snapshot的时候会对Cache和entry(包含一个Value队列)加锁，但是不会对Value加锁，应该是没有更改Value的可能性。tsm的命名方式是generation+sequence。 TSM的layout，为什么利于读，如何找到指定数据？

### TSM layout

TSM文件的组成是：1. magic number(4bytes), 2. version(1bytes) 3.blocks 3.1 CRC(4bytes) 3.2 Values, 4.Index, 5.Index start position(END-8bytes)

### 读流程(Select)

httpd handle初始化QueryExecutor为cluster的QueryExecutor。query的url和sql语句解析完毕后就会调用QueryExecutor，根据select的中请求的时间范围，确定哪些shard会被检索，接着把这些shard按nodeid分类，如果一个shard有多个owner，则随机取一个owner。根据node,shard和expr（where的条件）创建iteration。如果接收select的服务器和shard所在的服务器不在同一个节点，则建出来的Iteration会dial到对应的机器。根据Series和Field name会在shard上创建一个cursor，然后根据条件轮询这个Field的值。

#### Merge Iteration

在读取数据是需要平凡的对Iteration进行合并，比如一个measurement有多个Field，那么每个Field都会有一个Iteration，而所有的这些Field最终会合并成一个Iteration。合并后的Iteration只有一个数据类型，这个类型是适用于所有被合并的Iteration中占空间最小的那个。比如有两个Field的类型分别是float64， int32，那么合并后的Iteration的类型为float64。

#### 远程 Iteration

远程Iteration用于读取cluster中其他节点的数据。远程Iteration实际是维持一个TCP连接到对应的节点，远程节点构建一个本地的Iteration，然后将数据写入conn中。

#### query_executor跟store的交互
query_executor.go文件中：

e.TSDBStore.ShardIteratorCreator(shardID)
ics = append(ics, newRemoteIteratorCreator(dialer, nodeID, shardIDs))

FloatCursor接口

    :::go
    type floatCursor interface {
            cursor
            nextFloat() (t int64, v float64)
    }

Cursor和Iterator的差别是，Cursor只取一列数据，Iteration会取query相关的所有列。

Iterator接口

```go
type Iterator interface {
    Close() error
}
```


FloatIterator接口

```go
type FloatIterator interface {
        Iterator
        Next() *FloatPoint
}
```

engineFloatCursor是包括cache中Series+Field的值，和Series+Field在store中的值(只提供对应的Cursor)

store需要实现如下接口，以便influxql创建迭代器。

```go
type IteratorCreator interface {
	// Creates a simple iterator for use in an InfluxQL query.
	CreateIterator(opt IteratorOptions) (Iterator, error)

	// Returns the unique fields and dimensions across a list of sources.
	FieldDimensions(sources Sources) (fields, dimensions map[string]struct{}, err error)

	// Returns the series keys that will be returned by this iterator.
	SeriesKeys(opt IteratorOptions) (SeriesList, error)
}
```

下面是influxql和store交互的接口。

```go
type TSDBStore interface {
	CreateShard(database, policy string, shardID uint64) error
	WriteToShard(shardID uint64, points []models.Point) error

	DeleteDatabase(name string) error
	DeleteMeasurement(database, name string) error
	DeleteRetentionPolicy(database, name string) error
	DeleteSeries(database string, sources []influxql.Source, condition influxql.Expr) error
	ExecuteShowFieldKeysStatement(stmt *influxql.ShowFieldKeysStatement, database string) (models.Rows, error)
	ExecuteShowTagValuesStatement(stmt *influxql.ShowTagValuesStatement, database string) (models.Rows, error)
	ExpandSources(sources influxql.Sources) (influxql.Sources, error)
	ShardIteratorCreator(id uint64) influxql.IteratorCreator
}
```

## TCP server实现

在`tcp/mux.go`中实现了一个tcp server，accept connection后启动一个新的goroutine(mux.handleConn)处理connection。handleConn首先读出conn的第一个字节，判断这个请求的类型，每个请求类型都有一个listener与之对应。根据请求类型把conn放入listener的channel中。handleConn这个goroutine结束。listener实现了type Listener interface接口（Accept/Close/Addr)。其中Accept的作用就是讲刚才放入channel中的conn返回给调用者。
每个需要处理TCP连接的Service通过`Listen(header byte)`得到一个listener，然后在自己的serve函数中调用listener的Accept函数获得conn对象。Service在获得conn后，新启动一个goroutine，处理这个conn。每个service会有自己的Request Type，所以在这个处理conn的goroutine中会先解析Request Type，然后再parse request进行处理。解析Request Type和parse request是每个Service自己定义的。比如，在Cluster service中每个Request Type对应一个Byte，而Request的内容是protobuf编码的字节流。而在Snapshot service中，Request使用JSON编码，而非protobuf。

## v1.6.0 Influxql

Influxql的语法使用AST(abstract syntax tree)组织。

## 数据Index

主要是在DatabaseIndex中。

    ShardGroup
      Shards
        Shard ->
          Measurement Name ->
            Field objects
          DatabaseIndex ->
            Measurement Name ->
              Measurement object
                Tag.Key + Tag.Value -> Series ID
                Series ID ->
                  Series Object
            Series Key ->
              Series object
