---
title: 一种快速找出RocksDB相同前缀的方法
date: 2018-05-29
tags:
---


## 问题产生

这个问题是在在测试CEPH RGW S3接口的时候发现的。S3接口在使用list object的时候如果提供delimiter的话，response会将delimiter前具有同样内容的多个object聚合成一条CommonPrefix记录。这样做的一个好处是对象存储可以在一定程度支持目录结构。对象存储本身不存在目录结构，但是聚合后的记录可以看作一个目录。RGW在实现这个功能时只是简单的遍历所有记录，然后提取出相同前缀。如果RGW中的对象非常大，那么速度会非常慢。

## 问题的解决

要解决这个问题就要找到一种方法从omap中快速提取CommonPrefix的方法。由于Ceph omap是通过rocksdb实现的，所以问题就变成rocksdb能否快速提取Key中的CommonPrefix。经过跟社区的讨论，视乎可以通过先找出第一条记录，然后使用比第一个CommonPrefix的前缀大，但是比下一个CommonPrefix的前缀小的Seek前缀来快速定位下一个CommonPrefix的前缀。
为了验证种方法的可行性，我写了一个demo，代码如下。

```c
char sep = '/';
char bigger_than_sep = '/'+1;
auto it = db->NewIterator(ReadOptions());
for (it->SeekToFirst(); it->Valid(); ){
        auto key = it->key().ToString();
        auto p = key.find(sep);
        if (p == string::npos) {
              cout << key << endl;
              it->Next();
              continue;
        }
        auto cmax = key.substr(0,p) + bigger_than_sep;
        it->Seek(cmax);
        cout << key.substr(0, p) << endl;
}
```

代码中sep就是指S3中的delimiter，通过在第一个CommonPrefix后将delimiter改为delimiter的unicode值加1，就能快速定位到下一个不同前缀的位置。

经过简单对比，使用上述方法比完全遍历所有Key来获取CommonPrefix的速度快非常多。
