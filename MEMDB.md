---
title: hashicorp memdb 代码分析
date: 2019-07-30
tags: ["golang", "MVCC", "MemDB"]
---
![power switch](/powerswitch.jpg)
<center> Poor man's power switch </center>

## MemDB介绍

MemDB是一个纯内存数据库。Consul使用MemDB存储元数据，因为Raft的数据可以通过Log和Snapshot恢复，所以数据库只需要在内存中就可以。
对MemDB感兴趣是因为他实现了MVCC，当然他同时具有ACID，watch和rich index功能。
MVCC(Multi-Version Concurrency Control)可以使数据库在进行写操作的同时进行多个只读操作。这就需要在含有写操作的事务在提交的时候，只读操作的事务看到的数据还是一致的。MVCC通过写事务和读事务分别在不同的版本上操作来实现这个功能。
数据库在处理并发事务时除了可以使用MVCC外，比较常用的一种技术是两阶段锁（Two-Phase Locking）。使用2PL时每个读操作需要一个共享锁，而每个写操作需要一个排他锁。共享锁可以允许多个读事务同时发生，但是不允许写操作发生。排他锁及不允许读也不允许其他的写操作。也就是说除了多个读事务可以并发外，写写和读写都不能并发。
相比2PL，MVCC的优势在于读和写相互不干扰，因为他们各自在不同的版本上执行。MVCC不允许不同事务写同一列。为了多个Index的一致性，MemDB的MVCC不允许两个写事务同时发生，无论这两个事务是不是写同一列。这点使得整个数据库发生冲突的可能大非常多。

## immutable radix tree

MemDB使用immutable radix tree来存储数据。immutable radix tree是一个不可变的[基数树](https://en.wikipedia.org/wiki/Radix_tree)，一旦有树的某个节点发生了写操作，这个节点及这个节点以上的所有节点都会被新节点代替。如下例所示，如果要将root节点的左子节点的内容从3变成6，不能只是改变内容，而是要生成一个新的节点（新节点后面加了*号）。跟节点也是一个新节点，只是内容和原来节点一样。此时这个基数树就有了两个版本，第一个版本的root节点不加*号，而第二个版本的root有*号。如果有读操作在生成新树的操作之前发生，而且这个操作时间很长，在生成新树之后才结束。即便如此读操读到的始终是根节点为没有*号的那棵树。
```
               b                  b          b
              +--+               +--+       +--+
              |2 |               |2 |       |2 |*
              +--+               +--+       +--+
              /  \               /          /  \
             a    cd            a          a    cd
            /      \           /          /      \
          +--+    +--+       +--+       +--+    +--+
          |3 |    |5 |  -->  |3 |       |6 |*   |5 |
          +--+    +--+       +--+       +--+    +--+
         /                       \       /
        x                         x     x
       /                           \   /
     +--+                           +--+
     |3 |                           |5 |
     +--+                           +--+
```
radix tree已KV的形式读取和存入数据，使用Key路由到某个具体的节点。比如上例中要读到内容5，可以使用key="bax"。
## MemDB结构

从immutable radix tree的特性可以看到，他是实现MVCC的基础。那么MemDB的表到底是怎么映射到immutable radix tree的呢？
MemDB由TableSchema组成，TableSchema定义了Table的名称和这个table所有的IndexSchema。IndexSchema定义了Index的名称和Indexer接口。我们知道Index的作用是加快查询速度。如果没有Index，那么要找出含有表中含有某个条件的对象就只能遍历所有对象。这里的条件一般指对象的一个或者多个属性符合某个条件。那么查找一个对象的操作就可以分解为先确定哪些属性是查询需要的，然后再通过对应属性的Index查找对象，因为Index一般以一些能够加速查找的数据结构存储（比如基数树），所以无须遍历所有对象。Indexer接口就定义了FromArgs方法，该方法的作用是获得Index的key。Indexer对象还要实现FromObject方法，该方法的作用是根据对象获得其Index属性值。比如StringFieldIndex就是Indexer的一个实现。一个StringFieldIndex中包含Field的属性。Field指被存储对象的属性名称，通过golang的reflection可以得到对应的值。
比如有如下的对象需要存在名为Students的Table中，Table中有一个ID的索引，indexer。
```go
type Student struct {
        ID int
        Name string
}

indexer := StringFieldIndex{
        Field: "ID",
        Lowercase: false,
}
```
使用indexer.FromObject(obj)就可以获得obj的ID属性对应的值。
因为每个表中可能会有多个IndexSchema，所以在插入一个对象的时候，需要遍历所有IndexSchema，把该对象插入到每一个IndexSchema中。可以想象查找的时候首先要指定使用哪个Table和Index，然后程序会在对应的IndexSchema中根据值找到对应的对象。
MemDB使用radix tree存储Index，比如Student示例中Student.ID的值可以做key，而一个Student对象就可以作为value存储在radix tree中。那么Index本身要如何存储呢？MemDB中一个IndexSchema所组成的radix tree子树会插入到MemDB的radix treeroot节点下，而key就是TableName.IndexName。这样查找的过程就变成，根据TableName.IndexName找到对应的radix tree子树，再用用户要查找的值作为key在SchemaIndex对应的radix tree子树中找到对应的对象。MemDB的存储结构如下所示。

```
                          +------------------------------+
                          |            root              |
                          +------------------------------+
                         /                                \
                    TableName1.                        TableName2.
                       /                                   \
                    +-----------+                      +-----------+
                    |           |                      |           |
                    +-----------+                      +-----------+
                    /          \                       /          \
              IndexName1   IndexName2            IndexName3   IndexName4
                  /              \                   /              \
               +---+            +---+             +---+            +---+
               |   |            |   |             |   |            |   |
               +---+            +---+             +---+            +---+
              /     \          /     \           /     \          /     \
             x   ..  y        x   ..  y         x   ..  y        x   ..  y
            / \     / \      / \     / \       / \     / \      / \     / \
          +--++--++--++--+ +--++--++--++--+  +--++--++--++--+ +--++--++--++--+
          |  ||  ||  ||  | |  ||  ||  ||  |  |  ||  ||  ||  | |  ||  ||  ||  |
          +--++--++--++--+ +--++--++--++--+  +--++--++--++--+ +--++--++--++--+

```

## 总结

immutable radix tree使用已空间换性能的方式，减少数据库不同事务间的冲突。在处理并发写的时候MemDB没有检查各个写之间的冲突域，而是对整个DB加了个全局锁，会降低写性能。
