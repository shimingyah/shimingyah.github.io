---
layout: post
title: "BlueStore源码分析之架构设计"
date: 2019-09-25
description: "BlueStore源码分析之架构设计"
tag: Ceph
---

### 前言

Ceph早期的单机对象存储引擎是`FileStore`，为了维护数据的一致性，写入之前数据会先写`Journal`，然后再写到文件系统，会有一倍的写放大，而同时现在的文件系统一般都是日志型文件系统(ext系列、xfs)，文件系统本身为了数据的一致性，也会写`Journal`，此时便相当于维护了两份`Journal`；另外`FileStore`是针对`HDD`的，并没有对`SSD`作优化，随着`SSD`的普及，针对`SSD`优化的单机对象存储也被提上了日程，`BlueStore`便由此应运而出。

`BlueStore`最早在`Jewel`版本中引入，用于在`SSD`上替代传统的`FileStore`。作为新一代的高性能对象存储后端，`BlueStore`在设计中便充分考虑了对`SSD`以及`NVME`的适配。针对`FileStore`的缺陷，`BlueStore`选择绕过文件系统，直接接管裸设备，直接进行对象数据IO操作，同时元数据存放在`RocksDB`，大大缩短了整个对象存储的IO路径。`BlueStore`可以理解为一个支持`ACID`事物型的本地日志文件系统。

### 目录

* [设计理念](#chapter1)
* [整体架构](#chapter2)
* [核心模块](#chapter3)
* [IO流程](#chapter4)

### <a name="chapter1"></a>设计理念

**存储系统中，数据的可靠性是至关重要的**。所有的读操作都是同步的，也就是除非命中缓存，否则必须从磁盘上读到指定的数据才能返回。但是写操作则不一样，一般为了性能考虑，所有写操作都会先写内存缓存`Page-Cache`便返回客户端成功，然后由文件系统批量刷盘。但是内存是易失性存储介质，掉电后数据便会丢失，所以为了数据可靠性，我们不能这么做。

**一种可行的替代方案便是将数据先写入性能更好的非易失性存储介质**(SSD、NVME等)充当的中间设备，然后再将数据写入内存缓存，便可以直接返回客户端成功，等到数据写入到普通磁盘的时候再释放中间设备对应的空间。写入中间设备的过程我们称为写日志`Journal`。如果写`Journal`的过程失败了，那么直接返回失败即可，如果刷磁盘失败了，那么我们可以基于`Journal`来重放相应的数据，并不会影响系统的正确性。但是由于引入了`Journal`，所以存在双写，造成了写放大。在生产环境一般用`SSD`或者`NVME`做`Journal`，`HDD`做普通大容量存储。

**除了数据可靠性，数据一致性也是重中之重**。涉及数据修改的所有操作，要么完全完成，要么没有变化，而不能是介于两者之间的中间状态，也就是所有的修改都要符合事物的`ACID`语义，同时符合上述语义的存储系统我们称之为`事务型存储系统`。

**`BlueStore`便是一个事务型的本地日志文件系统**。因为面向下一代全闪存阵列的设计，所以`BlueStore`在保证`数据可靠性`和`一致性`的前提下，需要尽可能的减小日志系统中`双写`带来的影响。全闪存阵列的存储介质的主要开销不再是磁盘寻址时间，而是数据传输时间。因此当一次写入的数据量超过一定规模后，写入`Journal`盘(SSD)的延时和直接写入数据盘(SSD)的延迟不再有明显优势，所以`Journal`的存在性便大大减弱了。但是要保证`OverWrite(覆盖写)`的`数据一致性`，又不得不借助于`Journal`，所以针对`Journal`设计的考量便变得尤为重要了。

**一个可行的方式是使用`增量日志`**。针对大范围的覆盖写，只在其前后非磁盘块大小对齐的部分使用`Journal`，即`RMW`，其他部分直接重定向写`COW`即可。

**`BlockSize`**：磁盘IO操作的最小单元(原子操作)。HDD为`512B`，SSD为`4K`。即读写的数据就算少于`BlockSize`，磁盘IO的大小也是`BlockSize`，是原子操作，要么写入成功，要么写入失败，即使掉电不会存在部分写入的情况。

**`RWM(Read-Modify-Write)`**：指当覆盖写发生时，如果本次改写的内容不足一个`BlockSize`，那么需要先将对应的块读上来，然后再内存中将原内容和待修改内容合并`Merge`，最后将新的块写到原来的位置。但是`RMW`也带来了两个问题：**一是**需要额外的读开销；**二是**`RMW`不是原子操作，如果磁盘中途掉电，会有数据损坏的风险。为此我们需要引入`Journal`，先将待更新数据写入`Journal`，然后再更新数据，最后再删除`Journal`对应的空间。

**`COW(Copy-On-Write)`**：指当覆盖写发生时，不是更新磁盘对应位置已有的内容，而是新分配一块空间，写入本次更新的内容，然后更新对应的地址指针，最后释放原有数据对应的磁盘空间。理论上`COW`可以解决`RMW`的两个问题，但是也带来了其他的问题：**一是**`COW`机制破坏了数据在磁盘分布的物理连续性。经过多次`COW`后，读数据的顺序读将会便会随机读。**二是**针对小于块大小的覆盖写采用`COW`会得不偿失。**是因为**：**一是**将新的内容写入新的块后，原有的块仍然保留部分有效内容，不能释放无效空间，而且再次读的时候需要将两个块读出来做`Merge`操作，才能返回最终需要的数据，将大大影响读性能。**二是**存储系统一般元数据越多，功能越丰富，元数据越少，功能越简单。而且任何操作必然涉及元数据，所以元数据是系统中的热点数据。`COW`涉及空间重分配和地址重定向，将会引入更多的元数据，进而导致系统元数据无法全部缓存在内存里面，性能会大打折扣。

### <a name="chapter2"></a>整体架构

基于以上设计理念，`BlueStore`的写策略综合运用了`COW`和`RMW`策略。**非覆盖写**直接分配空间写入即可；**块大小对齐的覆盖写**采用`COW`策略；**小于块大小的覆盖写**采用`RMW`策略。

整体架构设计如下图：

![](http://img-ys011.didistatic.com/static/anything/bluestore.png)

### <a name="chapter3"></a>核心模块

* BlockDevice：物理块设备，使用`Libaio`操作裸设备，AsyncIO。
* RocksDB：存储`WAL`、对象元数据、对象扩展属性Omap、磁盘分配器元数据。
* BlueRocksEnv：抛弃了传统文件系统，封装RocksDB文件操作的接口。
* BlueFS：小型的`Append`文件系统，实现了`RocksDB::Env`接口，给RocksDB用。
* Allocator：磁盘分配器，负责高效的分配磁盘空间。

#### BlueFS

RocksDB是基于本地文件系统的，但是文件系统的许多功能对于RocksDB不是必须的，所以为了提升RocksDB的性能，需要对本地文件系统进行裁剪。最直接的办法便是为RocksDB量身定制一套本地文件系统，BlueFS便应运而生。

BlueFS是个简易的用户态日志型文件系统，恰到好处的实现了`RocksDB::Env`所有接口。根据设计理念这一章节，我们知道引入`Journal`是为了进行写加速，`WAL`对于提升RocksDB的性能至关重要，所以BlueFS在设计上支持把`.log`和`.sst`分开存储，`.log`使用速度更快的存储介质(NVME等)。

在引入BlueFS后，BlueStore将所有存储空间从逻辑上分了3个层次：

* **慢速空间(Block)**：存储对象数据，可以使用`HDD`，由`BlueStore`管理。
* **高速空间(DB)**：存储`RocksDB`的`sst`文件，可以使用`SSD`，由`BlueFS`管理。
* **超高速空间(WAL)**：存储`RocksDB`的`log`文件，可以使用`NVME`，由`BlueFS`管理。

#### Alloactor

`BlueStore`的磁盘分配器，负责高效的分配磁盘空间。目前支持`Stupid`和`Bitmap`两种磁盘分配器。都是仅仅在内存中分配，并不做持久化。

#### FreeListManager

负责管理空闲空间列表。目前支持`Extent`和`Bitmap`两种，由于`Extent`开销大，新版中已经移除，只剩`Bitmap`。空闲的空间列表会持久化在`RocksDB`中，为了保证元数据和数据写入的一致性，`BitmapFreeListmanager`会由使用者调用，会将对象元数据和分配结果通过`RocksDB`的`WriteBatch`接口原子写入。

#### Cache

BlueStore抛弃了文件系统，直接管理裸设备，那么便用不了文件系统的Cache机制，需要自己实现元数据和数据的Cache，缓存系统的性能直接影响到了`BlueStore`的性能。

`BlueStore`有两种Cache算法：`LRU`和`2Q`。元数据使用`LRU`Cache策略，数据使用`2Q`Cache策略。

### <a name="chapter4"></a>IO流程

#### 读流程

处理读请求会先从RocksDB找到对应的磁盘空间，然后通过`BlockDevice`读出数据。

#### 写流程

处理写请求时会进入事物的状态机，简单流程就是先写数据，然后再原子的写入对象元数据和分配结果元数据。写入数据如果是对齐写入，则最终会调用`do_write_big`；如果是非对齐写，最终会调用`do_write_small`。

具体的IO流程会在之后的篇幅分析。

### 参考资源

* [Ceph BlueStore](http://blog.wjin.org/posts/ceph-bluestore.html)
* [ceph存储引擎bluestore解析](http://www.sysnote.org/2016/08/19/ceph-bluestore/)
* [Ceph设计原理与实现](https://book.douban.com/subject/27178824/)
* [sage:bluestore-a-new-storage-backend-for-ceph](https://www.slideshare.net/sageweil1/bluestore-a-new-storage-backend-for-ceph-one-year-in)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [BlueStore源码分析之架构设计](https://shimingyah.github.io/2019/09/BlueStore%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1/)