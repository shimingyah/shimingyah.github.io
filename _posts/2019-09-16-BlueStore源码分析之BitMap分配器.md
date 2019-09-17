---
layout: post
title: "BlueStore源码分析之BitMap分配器"
date: 2019-09-16
description: "BlueStore源码分析之BitMap分配器"
tag: Ceph
---

### 前言

`块`是磁盘进行数据操作的最小单位。任意时刻磁盘中的每个块只能是`占用`或`空闲`两种状态之一，如果将磁盘空间按照块进行切分、编号，然后每个块使用唯一的一个Bit表示其状态，那么所有的Bit最终会形成一个有序的Bit流，通过这个Bit流我们可以索引磁盘中任意空间的状态，这种磁盘空间管理方式我们称为`位图`。

BlueStore直接管理裸设备，那么必然面临着如何高效分配磁盘中的块。BlueStore支持基于`Extent`和基于`BitMap`的两种磁盘分配策略，刚开始使用的是`BitMap`分配器，由于性能问题又切换到了`Stupid`分配器(基于`Extent`)。之后`Igor Fedotov`大神重新设计和实现了[新版本BitMap分配器](https://docs.google.com/presentation/d/1_1Otkgv76fbCU2Zogjz748sEAG-1Nfiidbb6mgTON-A/edit#slide=id.p)，性能也比`Stupid`要好，默认的磁盘分配器又改回了`BitMap`，在此便简单分析一下新版本的`BitMap`分配器。

### 目录

* [名词解释](#chapter1)
* [通用接口](#chapter2)
* [架构设计](#chapter3)
* [类定义](#chapter4)
* [空间分配](#chapter5)
* [空间回收](#chapter6)

### <a name="chapter1"></a>名词解释

* `extent`：`offset + length`，表示一段连续的物理磁盘地址空间。
* `PExtentVector`：存放空间分配结果，可能是一个或者多个extent。
* `bdev_block_size`：磁盘块大小，IO的最小单元，默认4K。
* `min_alloc_size`：最小分配单元，SSD默认16K，HDD默认64K。
* `max_alloc_size`：最大分配单元，默认`0`不限制，即一次连续的分配结果只包含一个`extent`。
* `alloc_unit`：分配单元(AU)，一般设置为`min_alloc_size `。
* `slot`：uint64类型，64 bit，位操作的基本单元。
* `children`：`slot`的分配单元，可能占1 bit，也可能占2 bit。
* `slot_set`：8个slot构成一个slot集合(512bit)，构成上层Level的 `children`。
* `slot_vector`：slot数组，也即uint64的数组。每一层都用一个`slot_vector`组织。

### <a name="chapter2"></a>通用接口

BlueStore抽象了`Allocator`基类，提供了以下8个方法，相应的`Stupid`和`BitMap`分配器都实现了以下方法。

```
int64_t allocate(uint64_t want_size, uint64_t alloc_unit, uint64_t max_alloc_size, int64_t hint, PExtentVector *extents)

void release(const interval_set<uint64_t>& release_set)

void init_add_free(uint64_t offset, uint64_t length)

void init_rm_free(uint64_t offset, uint64_t length)

void dump()

void shutdown()

uint64_t get_free()

double get_fragmentation(uint64_t alloc_unit)
```

### <a name="chapter3"></a>架构设计

`BitMap`分配器在N版之后出现了新版本，新版本比老版本和`Stupid`性能都要好，因此默认的分配器是`BitMap`。

![](http://img-ys011.didistatic.com/static/anything/old-bmap.png)

老版本`BitMap`分配器树中每个节点都会统计自己子树中包含的空闲磁盘空间和已分配磁盘空间，这在分配连续大块的磁盘空间时可以跳过空间不足的子树，快速定位到剩余空间能够满足要求的子树，从而提高分配效率。但是各个节点都单独开辟内存空间，使用指针连接，使用的是离散的内存空间。

![](http://img-ys011.didistatic.com/static/anything/bmap.png)

新版本`BitMap`分配器以`Tree-Like`的方式组织数据结构，整体分为L0、L1、L2三层。每一层都包含了完整的磁盘空间映射，只不过是slot以及children的粒度不同，这样可以加快查找。

新版本分配器分配空间的大体策略如下：

1. 循环从L2中找到可以分配空间的slot以及children位置。
2. 在L2的slot以及children位置的基础上循环找到L1中可以分配空间的slot以及children位置。
3. 在L1的slot以及children位置的基础上循环找到L0中可以分配空间的slot以及children位置。
4. 在1-3步骤中保存分配空间的结果以及设置每层对应位置分配的标志位。

新版本分配器整体架构设计有以下几点优势：

1. Allocator避免在内存中使用指针和树形结构，使用vector连续的内存空间。
2. Allocator充分利用64位机器CPU缓存的特性，最大程序的提高性能。
3. Allocator操作的单元是64 bit，而不是在单个bit上操作。
4. Allocator使用3级树状结构，可以更快的查找空闲空间。
5. Allocator在初始化时L0、L1、L2三级`BitMap`就占用了固定的内存大小。
6. Allocator可以支持并发的分配空闲，锁定L2的children(bit)即可，暂未实现。

总体来说新版本BitMap分配器的优势在于使用连续的内存空间从而尽可能更多的命中CPU Cache以提高分配器性能。

### <a name="chapter4"></a>类定义

AllocatorLevel是基类，AllocatorLevel01、AllocatorLevel02都继承了此类。

```
class AllocatorLevel {
   protected:
   	// 一个 slot 有多少个 children。slot 的大小是 64bit。
   	// L0、L2的 children 大小是1bit，所以含有64个。
   	// L1的 children 大小是2bit，所以含有32个。
    virtual uint64_t _children_per_slot() const = 0;
    
    // 每个 children 之间的磁盘空间长度，也即每个 children 的大小。
    // L0 的每个 children 间隔长度为：l0_granularity = alloc_unit。
    // L1 的每个 children 间隔长度为：l1_granularity = 512 * l0_granularity。
    // L2 的每个 children 间隔长度为：l2_granularity = (512/2) * l1_granularity。
    virtual uint64_t _level_granularity() const = 0;

   public:
    // L0 分配的次数，调用_allocate_l0的次数。
    static uint64_t l0_dives;
    
    // 遍历 L0 slot 的次数。
    static uint64_t l0_iterations;
    
    // 遍历 L0 slot(部分占用，部分空闲)的次数。
    static uint64_t l0_inner_iterations;
    
    // L0 分配 slot 的次数。
    static uint64_t alloc_fragments;
    
    // L1 分配 slot(部分占用，部分空闲)的次数。
    static uint64_t alloc_fragments_fast;
    
    // L2 分配的次数 ，调用 _allocate_l2的次数。
    static uint64_t l2_allocs;

    virtual ~AllocatorLevel() {}
};
```

### <a name="chapter5"></a>空间分配

分配的原则是优先分配连续的从未被分配过的Bit。流程图图下图所示：

![](http://img-ys011.didistatic.com/static/anything/allocate.png)

最终分配的结果存储在`interval_vector_t`结构体里面，实际上就是`extent_vector`，因为分配的磁盘空间不一定是完全连续的，所以会有多个extent，而在往`extent_vector `插入`extent`的时候会合并相邻的extent为一个extent。如果`max_alloc_size`设置了，且单个连续的分配大小超过了`max_alloc_size`，那么`extent`的`length`最大为`max_alloc_size `，同时这次分配结果也会拆分会多个`extent`。

### <a name="chapter6"></a>空间回收

空间回收的流程十分简单，依次设置L0、L1、L2相应的`bit`为`1`。函数定义如下所示：

```
void _free_l2(const interval_set<uint64_t>& rr)
```

### 参考资源

* [BlueStore New Bitmap Allocator](https://docs.google.com/presentation/d/1_1Otkgv76fbCU2Zogjz748sEAG-1Nfiidbb6mgTON-A/edit#slide=id.p)
* [Ceph Bulestore磁盘空间分配初探](https://cloud.tencent.com/developer/article/1492262)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [BlueStore源码分析之BitMap分配器](https://shimingyah.github.io/2019/09/BlueStore%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8BBitMap%E5%88%86%E9%85%8D%E5%99%A8/)