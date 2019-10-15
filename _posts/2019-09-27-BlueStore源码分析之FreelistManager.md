---
layout: post
title: "BlueStore源码分析之FreelistManager"
date: 2019-09-27
description: "BlueStore源码分析之FreelistManager"
tag: Ceph
---

### 前言

BlueStore直接管理裸设备，需要自行管理空间的分配和释放。`Stupid`和`Bitmap`分配器的结果是保存在内存中的，分配结果的持久化是通过`FreelistManager`来做的。

一个block的状态可以为**占用**和**空闲**两种状态，持久化时只需要记录一种状态即可，便可以推导出另一种状态，BlueStore记录的是空闲block。主要有两个原因：一是回收空间的时候，方便空闲空间的合并；二是已分配的空间在Object中已有记录。

FreelistManager最开始有`extent`和`bitmap`两种实现，现在默认为bitmap实现，extent的实现已经废弃。空闲空间持久化到磁盘也是通过RocksDB的Batch写入的。FreelistManager将block按一定数量组成段，每个段对应一个k/v键值对，key为第一个block在磁盘物理地址空间的offset，value为段内每个block的状态，即由0/1组成的位图，1为空闲，0为使用，这样可以通过与1进行异或运算，将分配和回收空间两种操作统一起来。

### 目录

* [通用接口](#chapter1)
* [数据结构](#chapter2)
* [初始化](#chapter3)
* [Merge](#chapter4)
* [Allocate](#chapter5)
* [Release](#chapter6)

### <a name="chapter1"></a>通用接口

FreelistManager最主要的接口就是allocator和release。

```
virtual void allocate(
	uint64_t offset, uint64_t length,
	KeyValueDB::Transaction txn) = 0;

virtual void release(
    uint64_t offset, uint64_t length,
    KeyValueDB::Transaction txn) = 0;
```

### <a name="chapter2"></a>数据结构

```
class BitmapFreelistManager : public FreelistManager {
	// rocksdb key前缀：meta_prefix为 B，bitmap_prefix为 b
	std::string meta_prefix, bitmap_prefix;
	
	// key-value DB的指针，封装了rocksdb的操作
	KeyValueDB *kvdb;
	
	// rocksdb的merge操作：按位异或(xor)
	ceph::shared_ptr<KeyValueDB::MergeOperator> merge_op;
	
	// enumerate操作时加锁
	std::mutex lock;

	// 设备总大小
	uint64_t size;
	
	// 设备总block数
	uint64_t blocks;

	// block大小：bdev_block_size，默认min_alloc_size
	uint64_t bytes_per_block;
	
	// 每个key包含多少个block， 默认128
	uint64_t blocks_per_key;
	
	// 每个key对应空间大小
	uint64_t bytes_per_key;

	// block掩码
	uint64_t block_mask;
	
	// key掩码
	uint64_t key_mask;

	bufferlist all_set_bl;

	// 遍历rocksdb key相关的成员
	KeyValueDB::Iterator enumerate_p;
	uint64_t enumerate_offset;
	bufferlist enumerate_bl;
	int enumerate_bl_pos; 
};
```

### <a name="chapter3"></a>初始化

BlueStore在初始化osd的时候，会执行mkfs，初始化FreelistManager(create/init），后续如果重启进程，会执行mount操作，只会对FreelistManager执行init操作。

```
int BlueStore::mkfs()
{
	......
	r = _open_fm(true);
	......
}

int BlueStore::_open_fm(bool create)
{
	......
	fm = FreelistManager::create(cct, freelist_type, db, PREFIX_ALLOC);

	// 第一次初始化，需要固化meta参数
	if (create) {
		fm->create(bdev->get_size(), min_alloc_size, t);
	}

	......
	int r = fm->init(bdev->get_size());
}

// create固化一些meta参数到kvdb中，init的时候，从kvdb读取这些参数
int BitmapFreelistManager::create(uint64_t new_size, uint64_t min_alloc_size,
		KeyValueDB::Transaction txn)
{
	txn->set(meta_prefix, "bytes_per_block", bl); // min_alloc_size
	txn->set(meta_prefix, "blocks_per_key", bl); // 128
	txn->set(meta_prefix, "blocks", bl);
	txn->set(meta_prefix, "size", bl);
}

// create/init 均会调用下面这个函数，初始化block/key的掩码
void BitmapFreelistManager::_init_misc()
{
	// 128 >> 3 = 16，每个block用1个bit表示。
	// 即一个key的value对应128个block，需要16字节。
	bufferptr z(blocks_per_key >> 3); 
	memset(z.c_str(), 0xff, z.length());
	all_set_bl.clear();
	all_set_bl.append(z);
	
	// 0x FFFF FFFF FFFF F000
	block_mask = ~(bytes_per_block - 1); 
	bytes_per_key = bytes_per_block * blocks_per_key;
	
	// 0xFFFF FFFF FFF8 0000
	key_mask = ~(bytes_per_key - 1);
}
```

### <a name="chapter4"></a>Merge

异或Merge接口实现：

`https://github.com/ceph/ceph/blob/master/src/os/bluestore/BitmapFreelistManager.cc#L21`

```
// 继承rocksdb merge接口：异或操作(xor)
struct XorMergeOperator : public KeyValueDB::MergeOperator {
	 // old_value不存在，那么new_value直接赋值为rdata。
    void merge_nonexistent(const char *rdata, size_t rlen,
                           std::string *new_value) override {
        *new_value = std::string(rdata, rlen);
    }
    
    // old_value存在，则与rdata逐位异或xor。
    void merge(const char *ldata, size_t llen, const char *rdata, size_t rlen,
               std::string *new_value) override {
        assert(llen == rlen);
        *new_value = std::string(ldata, llen);
        for (size_t i = 0; i < rlen; ++i) {
            (*new_value)[i] ^= rdata[i];
        }
    }
    // We use each operator name and each prefix to construct the
    // overall RocksDB operator name for consistency check at open time.
    string name() const override { return "bitwise_xor"; }
};
```

异或Merge接口应用：

`https://github.com/ceph/ceph/blob/master/src/kv/RocksDBStore.cc#L91`

```
bool Merge(const rocksdb::Slice& key,
	     const rocksdb::Slice* existing_value,
	     const rocksdb::Slice& value,
	     std::string* new_value,
	     rocksdb::Logger* logger) const override 
{
	// for default column family
	// extract prefix from key and compare against each registered merge op;
	// even though merge operator for explicit CF is included in merge_ops,
	// it won't be picked up, since it won't match.
	for (auto& p : store.merge_ops) {
		if (p.first.compare(0, p.first.length(),
					  key.data(), p.first.length()) == 0 &&
			  key.data()[p.first.length()] == 0) {
			// 如果old_value存在，那么直接merge，否则直接替换。
			if (existing_value) {
				p.second->merge(existing_value->data(), existing_value->size(),
							  value.data(), value.size(),
							  new_value);
			} else {
				p.second->merge_nonexistent(value.data(), value.size(), new_value);
			}
			break;
		}
	}
	return true;
}
```

最终调用Rocksdb的Batch的Merge方法。Batch可以实现简单写入和条件写入的原子操作。

### <a name="chapter5"></a>Allocate

分配和释放空间两种操作是完全一样的，都是调用异或(Xor)操作，我们着重看`_xor`函数。

```
void BitmapFreelistManager::allocate(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn)
{
	_xor(offset, length, txn);
}

void BitmapFreelistManager::release(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn)
{
	_xor(offset, length, txn);
}
```

```

void BitmapFreelistManager::_xor(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn)
{
	// 注意offset和length都是以block边界对齐
	uint64_t first_key = offset & key_mask;
	uint64_t last_key = (offset + length - 1) & key_mask;

	if (first_key == last_key) { // 最简单的case，此次操作对应一个段
		bufferptr p(blocks_per_key >> 3); // 16字节大小的buffer
		p.zero(); // 置为全0
		unsigned s = (offset & ~key_mask) / bytes_per_block; // 段内开始block的编号
		unsigned e = ((offset + length - 1) & ~key_mask) / bytes_per_block; // 段内结束block的编号

		for (unsigned i = s; i <= e; ++i) { // 生成此次操作的掩码
			p[i >> 3] ^= 1ull << (i & 7); // i>>3定位block对应位的字节， 1ull<<(i&7)定位bit，然后异或将位设置位1
		}

		string k;
		make_offset_key(first_key, &k); // 将内存内容转换为16进制的字符
		bufferlist bl;
		bl.append(p);
		bl.hexdump(*_dout, false);
		txn->merge(bitmap_prefix, k, bl); // 和目前的value进行异或操作

	} else { // 对应多个段，分别处理第一个段，中间段，和最后一个段，首尾两个段和前面情况一样

		// 第一个段
		{
			// 类似上面情况
			......

			// 增加key，定位下一个段
			first_key += bytes_per_key;
		}

		// 中间段，此时掩码就是全1，所以用all_set_bl
		while (first_key < last_key) {
			string k;
			make_offset_key(first_key, &k);
			all_set_bl.hexdump(*_dout, false);

			txn->merge(bitmap_prefix, k, all_set_bl); // 和目前的value进行异或操作

			// 增加key，定位下一个段
			first_key += bytes_per_key;
		}

		// 最后一个段
		{
			// 和前面操作类似
		}
	}
}
```

xor函数看似复杂，全是位操作，仔细分析一下，分配和释放操作一样，都是将段的bit位和当前的值进行异或。一个段对应一组blocks，默认128个，在k/v中对应一组值。例如，当磁盘空间全部空闲的时候，k/v状态如下: (b00000000，0x00), (b00001000, 0x00), (b00002000, 0x00)……b为key的前缀，代表bitmap。

### <a name="chapter6"></a>Release

释放空间和分配空间都是一样的操作。

```
void BitmapFreelistManager::release(uint64_t offset, uint64_t length, KeyValueDB::Transaction txn)
{
	_xor(offset, length, txn);
}
```

### 参考资源

* [Ceph BlueStore FreelistManager](http://blog.wjin.org/posts/ceph-bluestore-freelistmanager.html)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [BlueStore源码分析之FreelistManager](https://shimingyah.github.io/2019/09/BlueStore%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8BFreelistManager/)