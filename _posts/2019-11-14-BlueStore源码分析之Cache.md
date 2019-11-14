---
layout: post
title: "BlueStore源码分析之Cache"
date: 2019-11-14
description: "BlueStore源码分析之Cache"
tag: Ceph
---

### 前言

BlueStore使用`DIO + Libaio`跳过文件系统直接操作裸设备，那么便使用不了PageCache，为了提升读的性能，需要自行管理Cache。BlueStore主要涉及元数据、数据的Cache，同时也实现了`LRU`、`2Q`两种Cache算法，默认使用`2Q`算法。

### 目录

* [数据结构](#chapter1)
* [初始化](#chapter2)
* [元数据](#chapter3)
* [数据](#chapter4)
* [Trim](#chapter5)

### <a name="chapter1"></a>数据结构

```
// a cache (shard) of onodes and buffers
struct Cache {
	// 统计缓存命中率等
	PerfCounters *logger;
	
	// 缓存几乎所有的操作都要加锁
	std::recursive_mutex lock;
	
	// 当前缓存的extent数
	std::atomic<uint64_t> num_extents = {0};
	
	// 当前缓存的blob数
	std::atomic<uint64_t> num_blobs = {0};  
}
```

BlueStore默认的Cache算法是`2Q`，其缓存的元数据为`Onode`，数据为`Buffer`。

* LRUCache：Onode和Buffer各使用一个链表，采用LRU淘汰策略。
* TwoQCache：Onode使用一个链表，采用LRU策略；Buffer使用三个链表，采用2Q策略。

由于缓存的绝大部分操作都要加锁而且Onode使用一个链表，为了更高的性能，需要对Cache进行分片，hdd默认5片，ssd默认8片。

### <a name="chapter2"></a>初始化

```
BlueStore::BlueStore(CephContext *cct, const string& path)
	: ObjectStore(cct, path),
{
	......
	set_cache_shards(1); // 初始化cache
}

void BlueStore::set_cache_shards(unsigned num)
{
	size_t old = cache_shards.size();
	assert(num >= old);
	cache_shards.resize(num);

	for (unsigned i = old; i < num; ++i) {
		cache_shards[i] = Cache::create(cct, cct->_conf->bluestore_cache_type, logger); // 默认采用2q
	}
}

BlueStore::Cache *BlueStore::Cache::create(CephContext* cct, string type, PerfCounters *logger)
{
	Cache *c = nullptr;
	if (type == "lru")
		c = new LRUCache(cct); // LRU
	else if (type == "2q")
		c = new TwoQCache(cct); // 2q
	......
}

// 在osd初始化的时候，会根据参数调整shard的值。
int OSD::init()
{
	......
	store->set_cache_shards(get_num_op_shards());
	......
}

int OSD::get_num_op_shards()
{
	if (cct->_conf->osd_op_num_shards)
		return cct->_conf->osd_op_num_shards; // 默认0
	if (store_is_rotational)
		return cct->_conf->osd_op_num_shards_hdd; // 默认5
	else
		return cct->_conf->osd_op_num_shards_ssd; // 默认8
}
```

### <a name="chapter3"></a>元数据

BlueStore主要的元数据有两种类型：Collection、Onode；其中Collection常驻内存，缓存的便是Onode了。

#### Collection

**Collocation对应PG在BlueStore中的内存数据结构，Cnode对应PG在BlueStore中的磁盘数据结构。**

```
// PG的内存数据结构
struct Collection : public CollectionImpl {
	BlueStore *store;
	
	// 每个PG有一个osr，在BlueStore层面保证读写事物的顺序性和并发性。
	OpSequencerRef osr;
	
	// PG对应的 cache shard
	Cache *cache;
	
	// pg的磁盘结构
	bluestore_cnode_t cnode
	
	// cache缓存onode
	// cache onodes on a per-collection basis to avoid lock contention.
	OnodeSpace onode_map;
}

// PG的磁盘数据结构
struct bluestore_cnode_t {
	uint32_t bits;
}
```
在BlueStore启动的时候，会调用`_open_collections `创建Collection(会将PG的信息持久化到RocksDB)并且从RocksDB里面获取所有的Collection信息，同时也会指定一个Cache给PG。

```
int BlueStore::_open_collections(int *errors);

int BlueStore::_create_collection(TransContext *txc, const coll_t &cid, unsigned bits, CollectionRef *c)
{
	// 	给PG指定一个cache
	c->reset(new Collectioni(this,cache_shards[cid.hash_to_shard(cache_shards.size())], cid)); 
	(*c)->cnode.bits = bits;
	
	// map存放collection
	coll_map[cid] = *c;

	// 持久化PG信息到RocksDB
	::encode((*c)->cnode, bl);
	txc->t->set(PREFIX_COLL, stringify(cid), bl);
}

```

单个BlueStore管理的Collection是有限的(Ceph推荐每个OSD承载100个PG)，同时Collection结构体比较小巧，所以BlueStore将Collection设计为常驻内存。

#### Onode

每个对象对应一个Onode数据结构，而Object是属于PG的，所以Onode也属于PG，为了方便针对PG的操作(删除Collection时，不需要完整遍历Cache中的Onode队列来逐个检查与被删除Collection的关系)，引入了中间结构OnodeSpace使用一个map来记录Collection和Onode的映射关系。

```
struct Collection : public CollectionImpl {
	OnodeSpace onode_map; // objcet name -> onode
	......
};

struct OnodeSpace {
	private:
		mempool::bluestore_cache_other::unordered_map<ghobject_t,OnodeRef> onode_map;
};
```

单个BlueStore上面有海量的Onode，不一定能够全部缓存在内存里面，只能最大程度的缓存Onode信息在内存。

### <a name="chapter4"></a>数据

对象的数据使用`BufferSpace`来管理。

```
struct BufferSpace {
	// buffer map
	mempool::bluestore_cache_other::map<uint32_t, std::unique_ptr<Buffer>> buffer_map;
	// 包含脏数据的缓存队列
	state_list_t writing;
	......
};
```

当数据写完时，会根据标记决定是否缓存。`BlueStore::BufferSpace::finish_write`

当读取完成时，也会根据标记决定是否缓存。`bluestore_default_buffered_read`

### <a name="chapter5"></a>Trim

BlueStore的元数据和数据都是使用内存池管理的，后台有内存池线程监控内存的使用情况，超过内存使用的限制便会做trim，丢弃一部分缓存的元数据和数据。

```
void *BlueStore::MempoolThread::entry() {
    Mutex::Locker l(lock);
    ......
    while (!stop) {
    	 ......
    	 // 执行trim。
        _trim_shards(log_stats);
        
        // 定时唤醒执行trim。
        wait += store->cct->_conf->bluestore_cache_trim_interval;
        cond.WaitInterval(lock, wait);
    }
    ......
}
```

### 参考资源

* [Ceph BlueStore Cache](http://blog.wjin.org/posts/ceph-bluestore-cache.html)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [BlueStore源码分析之Cache](https://shimingyah.github.io/2019/11/BlueStore%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8BCache/)