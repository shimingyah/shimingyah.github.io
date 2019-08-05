---
layout: post
title: "BadgerDB源码分析之kv杂谈"
date: 2019-08-02
description: "BadgerDB源码分析之kv杂谈"
tag: BadgerDB
---

### 前言

随着分布式关系型数据库[TiDB](https://github.com/pingcap/tidb)和[CockroachDB](https://github.com/cockroachdb/cockroach)以及分布式图数据库[Dgraph](https://github.com/dgraph-io/dgraph)的流行，我们认识到传统单机数据库也可以基于分布式KVS来实现分布式存储，几乎任何的分布式存储都可以基于分布式KVS来做[A KVS For Any Scale](http://db.cs.berkeley.edu/jmh/papers/anna_ieee18.pdf)。而分布式KVS便是基于单机的KV引擎来实现的。众所周知，单机KV引擎里面应用的最广泛的莫过于LevelDB的变种RocksDB了，关于LevelDB以及RocksDB的源码分析文章网上已经很多了，在这里不在赘述。然而任何事物都有其两面性，已经很优秀的LevelDB以及RocksDB也有其劣势：写放大严重、不适合大value、不适合大规模数据存取等。于是便有人提出了KV分离存储的方案[Wisckey](https://www.usenix.org/system/files/conference/fast16/fast16-papers-lu.pdf)，同时github上也有了其GO语言的开源实现[BadgerDB](https://github.com/dgraph-io/badger)。

本系列文章便会先进行源码分析，然后在对象存储场景中指出BadgerBD有哪些不足和需要改进的点，让其更适用于做对象存储的单机KV。

`GO语言到底适不适合做底层存储？`网上也有不少声音说因为GO语言的GC以及CGO问题不适合做底层存储引擎。比如TiDB的存储层TiKV就抛弃GO语言使用Rust、CockroachDB的单机引擎也是基于RocksDB、Purge-GO的[Go-LevelDB](https://github.com/syndtr/goleveldb)真正生产环境使用的也寥寥无几。然而也有吃螃蟹的，比如基于Facebook的[Haystack](https://www.usenix.org/legacy/event/osdi10/tech/full_papers/Beaver.pdf)论文实现的[SeaweedFS](https://github.com/chrislusf/seaweedfs)、网易自研的对象存储等也都在生产环境跑了几年。同时由同样有GC的JAVA语言实现的HDFS，被各大公司应用在生产环境，除了离线数据存储的应用场景，也有基于其做对象存储准在线场景的。GC语言的STW(Stop-The-World)怕是影响其做实时在线数据存储的最大因素了，但是随着GO语言GC机制的不断完善，除了数据库、块存储、NAS、内存Cache等对实时性、低延迟要求比较严格的场景外，我想大多数的存储场景使用GO就足够了吧。

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [BadgerDB源码分析之kv杂谈](https://shimingyah.github.io/2019/08/BadgerDB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8Bkv%E6%9D%82%E8%B0%88/)