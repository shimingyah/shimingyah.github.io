---
layout: post
title: "BadgerDB源码分析之vlog优化"
date: 2019-08-13
description: "BadgerDB源码分析之vlog优化"
tag: BadgerDB
---

### 前言

回顾一下VLog的格式，这种由一个个record(即BadgerDB的entry)组成的WAL称为[recordio](https://github.com/google/or-tools/blob/stable/ortools/base/recordio.h)，可以有效的防止磁盘静默错误。
但是recordio也有一些缺点，比如要读取部分数据开销太大等。同时VLog只有一个可写的vlog文件，对于HDD尚还可以，毕竟HDD的并发度基本为0，但是对于SSD来说，由于只有一个goroutine在append写入，并没有充分利用SSD的并行性，而且写WAL通常使用`Libaio`更高效。所以VLog中value的写入可以使用Libaio直接操作裸设备，跳过文件系统，同时实现cache支持高效读操作。

### 目录

* [recordio优化](#chapter1)
* [裸设备优化](#chapter2)
* [未完待续](#chapter3)

### <a name="chapter1"></a>recordio优化

#### recordio缺点

在使用recordio组织WAL格式时，对于读取整个record来说没有什么影响，直接读取整个记录并且计算比对checksum即可。但是对象存储场景中通常存在读取对象的一部分数据，以便并发下载。此时如果我们使用recordio组织VLog的话，就需要读取整个record，然后计算比对checksum，确保数据无误后，再返回部分数据，而此时便造成了IO和内存的浪费。而之所以需要读取整个record是为了校验数据是否损坏。如果一个4MB的文件每次只读取64KB，那么读放大达到了`4*1024/64=64`倍，内存也多占用了`4*1024/64=64`倍。

当然也可以把文件按照64KB切片，存储为一个个record，这样读放大便降低为了1。但是如果一个文件是5GB的话，按照64KB切片，需要`5*1024*1024/64=81920`个分片，可见分片数量之多；如果按照4MB分片的话，需要`5*1024/4=1280`个分片，还是可以接受的，而通常大文件我们也按照4MB来切分。

#### block + recordio
针对对象存储的场景，我们可以借鉴RocksDB WAL的格式[Write-Ahead-Log-File-Format](https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format)。
将VLog划分为一个个连续的`kBlockSize `大小的`Block`，一个Block可以包含多个、一个、不足一个record。

**VLog Format：**

```
       +-----+-------------+--+----+----------+------+-- ... ----+
 File  | r0  |        r1   |P | r2 |    r3    |  r4  |           |
       +-----+-------------+--+----+----------+------+-- ... ----+
       <--- kBlockSize ------>|<-- kBlockSize ------>|

  rn = variable size records
  P = Padding
```

**Record Format：**

```
--------------------------------------------------------------------------------------------------------------
|Format: |klen + vlen + expireAt + meta + userMeta|     key     |     value     |           checksum           |
---------------------------------------------------------------------------------------------------------------
|Comment:|     header: 4 + 4 + 8 + 1 + 1 = 18B    |   key data  |   value data  | crc32(4B): header+key+value  |
---------------------------------------------------------------------------------------------------------------
```

### <a name="chapter2"></a>裸设备优化

因为VLog是append写入的，所以在删除的时候为了回收空间需要GC机制，会带来读写放大，消耗内部磁盘IO，在线上读写IO较高时，延迟会增加。除此之外，GC一般都是延时回收的，会带来一定的空间浪费，所以合理的GC时机很重要，BadgerDB把GC的操作暴露给了用户，让用户去选择GC的阈值和时机。

而且VLog的读取是通过mmap，脏页的淘汰算法由内核控制，上层无法干预而且也不一定适用于对象存储的场景。

所以VLog中value的写入我们可以通过Libaio直接操作裸设备，省去了文件系统调用的开销，同时异步化磁盘IO，提升读写的性能。由于BadgerDB把RocksDB的WAL使用VLog代替了，而且裸设备存储的仅仅是value，所以我们还需要引入WAL，来持久化key等一些元数据。但同时也引入了新的问题：`如果管理裸设备`以及`如何缓存`。

#### 磁盘分配器
既然直接操作裸设备，此时裸设备便相当于一个大文件，写入的数据落在裸设备的哪个位置，怎么分配裸设备空间以最大化IO性能等需要依赖一个良好的磁盘分配器。

##### Bitmap
我们将磁盘划分为一系列的`Block`，通过单调自增的ID来标识每一个Block。同时磁盘分配单元为Block，由`Bitmap`来管理。值为0表示当前Block没有被使用，值为1表示当前Block已被占用。

Block大小默认为`64KB`，一块8T的磁盘Block总数为`8 * 1024 * 1024 * 1024 / 64 / 1024 / 1024 = 128MB`，那么Bitmap占用的内存大小为`128MB / 8 = 16MB`，checksum占的内存大小为`128MB * 4 = 512MB`。

##### 分配算法

好的分配算法应该尽可能的使写趋近于顺序写。在使用Bitmap分配空闲Block的时候，

1. 顺序的分配磁盘从未被使用过的Block，保证整个磁盘的写为顺序写。如果没有这样的Block，则说明磁盘已经被顺序的写过一遍了，此时跳转到步骤2。
2. 顺序的查找满足分配空间的连续的块，保证磁盘局部的写为顺序写。如果没有这样的Block，则跳转到步骤3。
3. 顺序的查找最大的连续块和周围的Block，保证磁盘局部的写大部分为顺序写，小部分为随机写。

##### 持久化

Bitmap的信息是全部缓存在内存中的，当有写请求时，

##### 写请求
1. 磁盘分配器分配空闲Block，更改内存Bitmap。
2. 通过Libaio将value写入裸设备。
3. 将key、分配信息
4. checksum存储到LSM-Tree中，key为自增ID，value为当前BlockID的checksum。

### <a name="chapter3"></a>未完待续

未完待续

### 参考资源

* [Write-Ahead-Log-File-Format](https://github.com/facebook/rocksdb/wiki/Write-Ahead-Log-File-Format)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [BadgerDB源码分析之vlog优化](https://shimingyah.github.io/2019/08/BadgerDB%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8Bvlog%E4%BC%98%E5%8C%96)