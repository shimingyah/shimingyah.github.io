---
layout: post
title: "BadgerDB源码分析之VLog详解"
date: 2019-08-09
description: "BadgerDB源码分析之VLog详解"
tag: BadgerDB
---

### 前言

VLog里面记录了key-value的完整数据，发生故障时可以通过读取VLog里面的数据来恢复，此时便不需要LSM-Tree里面的WAL了，也就减少了一次写放大。

### 目录

* [目录结构](#chapter1)
* [IO模式](#chapter2)
* [Entry格式](#chapter3)
* [Reply日志](#chapter4)
* [GC机制](#chapter5)

### <a name="chapter1"></a>目录结构

![](http://img-ys011.didistatic.com/static/anything/vlog.png)
BadgerDB可以为LSM-Tree以及VLog配置不同的目录，而LSM-Tree的数据量相比VLog要小很多，如果有SSD磁盘，那么便可以把LSM-Tree的目录配置在SSD上，VLog配置在HDD上，加快读取LSM-Tree的速度。

VLog下面包含多个append写入的日志文件，以递增的ID来做文件名，由`maxFid`来控制，文件名格式为`000001.vlog, 000002.vlog, 00000N.vlog`。日志文件中只有maxFid是可写的，其他的都是只读的。

同时可以通过更改`ValueLogFileSize`参数来更改日志文件的大小，BadgerDB里面目前限制日志文件大小范围为`[1MB, 2GB]`

### <a name="chapter2"></a>IO模式

目前BadgerDB支持两种IO模式：第一种是StandardIO，第二种是MMap。默认的配置写入数据使用StandardIO，读取数据使用MMap。MMap的优势在于减少文件操作的系统调用，减少一次内核态到用户态的数据Copy，但是如果写入数据量特别大，便会产生大量的脏页，会flush到磁盘上，此时写入的速度便会降低。

### <a name="chapter3"></a>Entry格式

写入VLog的单元是Entry，即VLog中包含一些列的Entry，我们先看一下Entry的结构体定义：

```
type Entry struct {
	Key       []byte
	Value     []byte
	UserMeta  byte
	ExpiresAt uint64 // time.Unix
	meta      byte

	// Fields maintained internally.
	offset uint32
}
```
* Key：大小范围[1, 65000], 不能带有"!badger!"前缀，因为内部会自动加上。
* Value：大小范围[1MB, 2GB]。
* UserMeta：用户对key-value设置的Meta位。
* ExpiresAt：过期时间，截止到某一时刻。
* meta：BadgerDB内部的位，有bitValuePointer、bitDelete等，用于内部对该key-value的操作。
* offset：key-value在日志文件中的偏移起始位置。

Entry会按照一定的格式写入到VLog，接下来我们看一下Entry写入的格式：

```
Entry format:
 
---------------------------------------------------------------------------------------------------------------
|Format: |klen + vlen + expireAt + meta + userMeta|     key     |     value     |           checksum           |
---------------------------------------------------------------------------------------------------------------
|Comment:|     header: 4 + 4 + 8 + 1 + 1 = 18B    |   key data  |   value data  | crc32(4B): header+key+value  |
---------------------------------------------------------------------------------------------------------------

```
Entry Format包含四部分：header、key、value、checksum。

* key、value：便是具体的数据了，不再多说。
* checksum：防止磁盘的静默错误。但是这种格式的checksum不太适用于对象存储场景。后续会提出缺点以及改进的方法。
* header：expireAt、meta、userMeta也是具体的数据，不再多说。klen、vlen的作用类似于TCP网络编程中经常会遇到的粘包问题，通过加入消息体的长度来区分，这样便可以完整的知道单个Entry的边界，可以完整的读取一个Entry。

### <a name="chapter4"></a>Reply日志

BadgerDB舍弃了WAL，回顾一下之前的写入流程，VLog写入成功，LSM-Tree写入成功后立刻崩溃，此时的数据是没有写入到磁盘的sstable的，当DB再次重启的时候，就需要通过读取VLog重放日志来恢复LSM-Tree里面的数据。

#### head
并不是VLog里面所有的数据都需要重放，重放的仅仅是没有写入到sstable的数据，所以此时需要一个key来记录重放的起始位置。head的key为`!badgerdb!head`，value为`valuePointer`

```
type valuePointer struct {
	Fid    uint32
	Len    uint32
	Offset uint32
}
```

head也是作为key-value存储在LSM-Tree的。每次写入VLog并且写入memtable之后，都会更新vhead的值(vhead变量类型是valuePointer，一直保存在内存中)，此时vhead和memtable的值都是在内存中的。等到memtable flush成sstable的时候，会先设置head值为vhead，然后连同memtable的值一起flush到sstable中。

只有memtable flush到level0的时候才会设置head的值为vhead，此时head的值便持久化到磁盘了。其他写入memtable的时候只会更新内存vhead的值，避免每次都要设置head值，否则存在好多个head值版本。

#### reply
每次重启DB的时候都会先读取head的值，然后解析出重放的起始位置，然后依次遍历每个Entry，调用`replayFunction`重放记录到LSM-Tree中。重放的时候写入memtable就返回了，重放完成后，更新vhead的值为VLog的末尾。

可能会出现一种情况就是，在重放的过程中或者重放完成(memtable还没有flush到sstable)又挂了，然后又重启DB，那么之前重放的数据便会丢失，此时head值没有改，还是重放之前的位置，不会影响，只需要再次重放即可。

### <a name="chapter5"></a>GC机制

由于key-value分离存储以及支持MVCC，所以需要对VLog做GC。BadgerDB内部不会自动做GC，而是对外提供了GC的接口`RunValueLogGC(discardRatio float64)`，需要使用者显式的去调用，具体使用方式可以参考官方文档：[garbage-collection](https://github.com/dgraph-io/badger#garbage-collection)，
同时BadgerDB在关闭DB的时候也会做一次GC。GC主要分为两步：`pick log`和`do run gc`。

#### pick log
我们知道head的value保存了最新写入到sstable的key-value在VLog里面的偏移位置以及对应Fd，head值的offset总是小于等于VLog的写入偏移位置，head值的Fid总是小于等于VLog的`maxFid`。

具体步骤如下：

1. 对所有只读和可写的vlog按照Fid排序。
2. 选取Fid小于head.Fid(排除maxFid，可写的vlog不做GC)的删除量最多的vlog为候选vlog。
3. 连续随机两次(目的是优先选择Fid小的vlog)从只读的vlog选取一个vlog作为另一个候选vlog。
4. 最后选择出两个待做GC的vlog。

#### do gc
pick log做完之后会选出来两个待做GC的vlog，此时需要先判断vlog是否达到做GC的阈值，会随机的从vlog的某个位置开始遍历读取entry，达到以下条件时遍历停止：

* 遍历时间超过10S。
* 遍历entry数超过`ValueLogMaxEntries * 1%`。
* 遍历entry总大小超过vlog文件大小的`10%`。

GC阈值：

* 遍历entry数大于`ValueLogMaxEntries * 1%`。
* 遍历entry总大小超过vlog文件大小的`0.075`
* 删除比例超过设置的`discardRatio`。

```
type reason struct {
	total float64
	discard float64
	count int
}

total:		vlog所有entry总大小
discard: 	vlog删除entry总大小
count:		vlog entry总数量
```

满足以上GC阈值的话便会触发rewrite，即将有效的key-value写入到`!badger!move`前缀的key space。依次从头开始遍历vlog，如果是有效的，便批量(64MB)重写到`!badger!move`前缀的key space，最后删除vlog文件。

丢弃key-value条件：

* vlog记录的version和LSM-Tree的不一致。
* 过期或者被删除。
* value存储在LSM-Tree中。由于key-value都存储在LSM-Tree，所以便不需要存储在vlog了，可以丢弃。
* 事物结束标志的key-value。

```
if vp.Fid == f.fid && vp.Offset == e.offset {
    moved++
    // This new entry only contains the key, and a pointer to the value.
    ne := new(Entry)
    ne.meta = 0 // Remove all bits. Different keyspace doesn't need these bits.
    ne.UserMeta = e.UserMe  
    // Create a new key in a separate keyspace, prefixed by moveKey. We are not
    // allowed to rewrite an older version of key in the LSM tree, because then this older
    // version would be at the top of the LSM tree. To work correctly, reads expect the
    // latest versions to be at the top, and the older versions at the bottom.
    if bytes.HasPrefix(e.Key, badgerMove) {
    	ne.Key = append([]byte{}, e.Key...)
    } else {
    	ne.Key = make([]byte, len(badgerMove)+len(e.Key))
    	n := copy(ne.Key, badgerMove)
    	copy(ne.Key[n:], e.Key)
    }
    ne.Value = append([]byte{}, e.Value...)
    wb = append(wb, ne)
    size += int64(e.estimateSize(vlog.opt.ValueThreshold))
    if size >= 64*mi {
    	tr.LazyPrintf("request has %d entries, size %d", len(wb), size)
    	if err := vlog.db.batchSet(wb); err != nil {
    		return err
    	}
    	size = 0
    	wb = wb[:0]
    }
}
```

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [BadgerDB源码分析之vlog详解](https://shimingyah.github.io/2019/08/BadgerDB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8Bvlog%E8%AF%A6%E8%A7%A3/)