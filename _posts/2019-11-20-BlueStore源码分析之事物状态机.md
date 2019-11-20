---
layout: post
title: "BlueStore源码分析之事物状态机"
date: 2019-11-20
description: "BlueStore源码分析之事物状态机"
tag: Ceph
---

### 前言

BlueStore可以理解为一个支持ACID的本地日志型文件系统。所有的读写都是以`Transaction`进行，又因为支持覆盖写，所以写流程设计的相对复杂一些，涉及到一系列的状态转换。我们着重分析一下状态机、延迟指标以及如何保证IO的顺序性和并发性。

### 目录

* [状态机](#chapter1)
* [延迟分析](#chapter2)
* [IO保序](#chapter3)
* [线程队列](#chapter4)
* [IO状态](#chapter5)
* [最后YY](#chapter6)

### <a name="chapter1"></a>状态机

#### queue\_transactions

`queue_transactions`是ObjectStore层的统一入口，KVStore、MemStore、FileStore、BlueStore都相应的实现了这个接口。` state_t state`变量记录了当前时刻事物处于哪个状态。在创建TransactionContext的时候会将`state`初始化为`STATE_PREPARE`，然后在`_txc_add_transaction`中会根据操作码类型(opcode)进行不同的处理。同时会获取PG对应的OpSequencer(每个PG有一个OpSequencer)用来保证PG上的IO串行执行，对于deferred-write会将其数据写入RocksDB(WAL)。

以下阶段就进入BlueStore状态机了，我们以写流程为导向分析状态机的每个状态。

#### STATE\_PREPARE

从state_prepare开始已经进入事物的状态机了。这个阶段会调用`_txc_add_transaction`将OSD层面的事物转换为BlueStore层面的事物；然后检查是否还有未提交的IO，如果还有就将state设置为`STATE_AIO_WAIT`并调用`_txc_aio_submit`提交IO，然后退出状态机，之后aio完成的时候会调用回调函数`txc_aio_finish`再次进入状态机；否则就进入`STATE_AIO_WAIT`状态。
_txc\_aio\_submit函数调用栈：

`bdev->aio_submit –> KernelDevice::aio_submit –> io_submit`将aio提交到内核Libaio队列。

**主要工作**：准备工作，生成大小写、初始化TransContext、deferred_txn、分配磁盘空间等。

**延迟指标**：`l_bluestore_state_prepare_lat`，从进入状态机到prepare阶段完成，平均延迟大概0.2ms左右。

#### STATE\_AIO\_WAIT

该阶段会调用`_txc_finish_io`进行SimpleWrite的IO保序等处理，然后将状态设置为`STATE_IO_DONE`再调用_txc\_state\_proc进入下一个状态的处理。

**主要工作**：对IO保序，等待AIO的完成。

**延迟指标**：`l_bluestore_state_aio_wait_lat`，从prepare阶段完成开始到AIO完成，平均延迟受限于设备，SSD 0.03ms左右。

#### STATE\_IO\_DONE

完成AIO，并进入`STATE_KV_QUEUED`阶段。会根据`bluestore_sync_submit_transaction`做不同处理。该值为布尔值，默认为false。

如果为true，设置状态为`STATE_KV_SUBMITTED`并且同步提交kv到RocksDB但是没有sync落盘(submit_transaction)，然后`applied_kv`。

如果为false，则不用做上面的操作，但是以下操作都会做。

最后将事物放在`kv_queue`里，通过`kv_cond`通知`kv_sync_thread`去同步IO和元数据。

**主要工作**：将事物放入`kv_queue`，然后通知`kv_sync_thread`，osr的IO保序可能会block。

**延迟指标**：`l_bluestore_state_io_done_lat`，平均延迟在0.004ms，通常很小主要耗在对SimpleWrite的IO保序处理上。

#### STATE\_KV\_QUEUED

该阶段主要在`kv_sync_thread`线程中同步IO和元数据，并且将状态设置为`STATE_KV_SUBMITTED`。具体会在异步线程章节分析`kv_sync_thread`线程。

**主要工作**：从`kv_sync_thread`队列中取出事物。

**延迟指标**：`l_bluestore_state_kv_queued_lat`，从事物进入队列到取出事物，平均延迟在0.08ms，因为是单线程顺序处理的，所以依赖于`kv_sync_thread`处理事物的速度。

#### STATE\_KV\_SUBMITTED

等待`kv_sync_thread`中kv元数据和IO数据的Sync完成，然后将状态设置为`STATE_KV_DONE`并且回调finisher线程。

**主要工作**：等待kv元数据和IO数据的Sync完成，回调finisher线程。

**延迟指标**：`l_bluestore_state_kv_committing_lat`，从队列取出事物到完成kv同步，平均延迟1.0ms，有极大的优化空间。

#### STATE\_KV\_DONE

如果是SimpleWrite，则直接将状态设置为`STATE_FINISHING`；如果是DeferredWrite，则将状态设置为`STATE_DEFERRED_QUEUED`并放入`deferred_queue`。

**主要工作**：如上。

**延迟指标**：`l_bluestore_state_kv_done_lat`，平均延迟0.0002ms，可以忽略不计。

#### STATE\_DEFERRED\_QUEUED

**主要工作**：将延迟IO放入`deferred_queue`等待提交。

**延迟指标**：`l_bluestore_state_deferred_queued_lat`，通常不小，没有数据暂不贴出。

#### STATE\_DEFERRED\_CLEANUP

**主要工作**：清理延迟IO在RocksDB上的WAL。

**延迟指标**：`l_bluestore_state_deferred_cleanup_lat`，通常不小，没有数据暂不贴出。

#### STATE\_FINISHING

**主要工作**：设置状态为`STATE_DONE`，如果还有DeferredIO也会提交。

**延迟指标**：`l_bluestore_state_finishing_lat`，平均延迟0.001ms。

#### STATE\_DONE

**主要工作**：标识整个IO完成。

**延迟指标**：`l_bluestore_state_done_lat`。

### <a name="chapter2"></a>延迟分析

BlueStore定义了状态机的多个延迟指标，由PerfCounters采集，函数为`BlueStore::_init_logger()`。

可以使用`ceph daemon osd.0 perf dump`或者`ceph daemonperf osd.0`来查看对应的延迟情况。

除了每个状态的延迟，我们通常也会关注以下两个延迟指标：

```
b.add_time_avg(l_bluestore_kv_lat, "kv_lat",
                   "Average kv_thread sync latency", "k_l",
                   
b.add_time_avg(l_bluestore_commit_lat, "commit_lat",
                   "Average commit latency", "c_l",                   
```

BlueStore延迟主要花费在`l_bluestore_state_kv_committing_lat `也即`c_l`，大概1ms左右。

BlueStore统计状态机每个阶段延迟的方法如下:

```
// 该阶段延迟 = 上阶段完成到该阶段结束
void log_state_latency(PerfCounters *logger, int state) {
	utime_t lat, now = ceph_clock_now();
	lat = now - last_stamp;
	logger->tinc(state, lat);
	last_stamp = now;
}
```

在块存储的使用场景中，除了用户并发IO外，通常用户也会使用`dd`等串行IO的命令，此时便受限于读写的绝对延迟，扩容加机器、增加线程数等横向扩展的优化便是无效的，所以我们需要关注两方面的延迟：**并发IO延迟**、**串行IO延迟**。

**并发IO延迟优化**：kv\_sync\_thread、kv\_finalize\_thread多线程化；自定义WAL；async read。

**串行IO延迟优化**：并行提交元数据、数据；将sync操作与其他状态并行处理。

### <a name="chapter3"></a>IO保序

保证IO的顺序性以及并发性是分布式存储必然面临的一个问题。因为BlueStore使用异步IO，后提交的IO可能比早提交的IO完成的早，所以更要保证IO的顺序，防止数据发生错乱。**客户端可能会对PG中的一个Object连续提交多次读写请求，每次请求对应一个Transaction，在OSD层面通过PGLock将并发的读写请求在PG层面串行化，然后按序依次提交到ObjectStore层，ObjectStore层通过PG的OpSequencer保证顺序处理读写请求**。

BlueStore写类型有SimpleWrite、DeferredWrite两种，所以我们分析一下SimpleWrite、DeferredWrite下的IO保序问题。

#### SimpleWrite

因为`STATE_AIO_WAIT`阶段使用Libaio，所以需要保证PG对应的OpSequencer中的txc按排队的先后顺序依次进入kv\_queue被`kv_sync_thread`处理，也即txc在OpSequencer中的顺序和在kv_queue中的顺序是一致的。

```
void BlueStore::_txc_finish_io(TransContext *txc)
{
	// 获取txc所属的OpSequencer，并且加锁，保证互斥访问osr
	OpSequencer *osr = txc->osr.get();
	std::lock_guard<std::mutex> l(osr->qlock);
	
	// 设置状态机的state为STATE_IO_DONE
	txc->state = TransContext::STATE_IO_DONE;

	// 清除txc正在运行的aio
	txc->ioc.running_aios.clear();

	// 定位当前txc在osr的位置
	OpSequencer::q_list_t::iterator p = osr->q.iterator_to(*txc);
	
	while (p != osr->q.begin()) {
		--p;
		// 如果前面还有未完成IO的txc，那么需要停止当前txc操作，等待前面txc完成IO。
		// 目的是：确保之前txc的IO都完成。
		if (p->state < TransContext::STATE_IO_DONE) {
			return;
		}
		
		// 前面的txc已经进入大于等于STATE_KV_QUEUED的状态了，那么递增p并退出循环。
		// 目的是：找到状态为STATE_IO_DONE的且在osr中排序最靠前的txc。
		if (p->state > TransContext::STATE_IO_DONE) {
			++p;
			break;
		}
	}

	// 依次处理状态为STATE_IO_DONE的tx
	// 将txc放入kv_sync_thread的kv_queue、kv_queue_unsubmitted队列
	do {
		_txc_state_proc(&*p++);
	} while (p != osr->q.end() && p->state == TransContext::STATE_IO_DONE);
	......
}
```

#### DeferredWrite

DeferredWrite在IO的时候也是通过Libaio提交到内核Libaio队列进行写数据，也需要保证IO的顺序性。

相应的数据结构如下：

```
class BlueStore {
	typedef boost::intrusive::list<
        OpSequencer, boost::intrusive::member_hook<
                         OpSequencer, boost::intrusive::list_member_hook<>,
                         &OpSequencer::deferred_osr_queue_item>>
	deferred_osr_queue_t;
   
	// osr's with deferred io pending
	deferred_osr_queue_t deferred_queue;
}

class OpSequencer {
	DeferredBatch *deferred_running = nullptr;
	DeferredBatch *deferred_pending = nullptr;
}

struct DeferredBatch {
	OpSequencer *osr;
	
	// txcs in this batch
	deferred_queue_t txcs;
}
```

BlueStore内部包含一个成员变量deferred\_queue；deferred\_queue队列包含需要执行DeferredIO的OpSequencer；每个OpSequencer包含deferred\_running和deferred\_pending两个DeferredBatch类型的变量；DeferredBatch包含一个txc数组。

如果PG有写请求，会在PG对应的OpSequencer中的deferred_pending中排队加入txc，待时机成熟的时候，一次性提交所有txc给Libaio，执行完成后才会进行下一次提交，这样不会导致DeferredIO乱序。

```
void BlueStore::_deferred_queue(TransContext *txc)
{
	deferred_lock.lock();
	// 排队osr
	if (!txc->osr->deferred_pending && !txc->osr->deferred_running) {
		deferred_queue.push_back(*txc->osr);
	}

	// 追加txc到deferred_pending中
	txc->osr->deferred_pending->txcs.push_back(*txc);
	
	_deferred_submit_unlock(txc->osr.get());
	......
}

void BlueStore::_deferred_submit_unlock(OpSequencer *osr)
{
	......
	// 切换指针，保证每次操作完成后才会进行下一次提交
	osr->deferred_running = osr->deferred_pending;
	osr->deferred_pending = nullptr;
	......

	while (true) {
		......
		// 准备所有txc的写buffer
		int r = bdev->aio_write(start, bl, &b->ioc, false);
	}

	......
	// 一次性提交所有txc
	bdev->aio_submit(&b->ioc);
}
```

### <a name="chapter4"></a>线程队列

线程+队列是实现异步操作的基础。BlueStore的一次IO经过状态机要进入多个队列并被不同的线程处理然后回调，线程+队列是BlueStore事物状态机的重要组成部分。BlueStore中的线程大致有7种。

* `mempool_thread`：无队列，后台监控内存的使用情况，超过内存使用的限制便会做trim。
* `aio_thread`：队列为Libaio内核queue，收割完成的aio事件。
* `discard_thread`：队列为discard_queued，对SSD磁盘上的extent做Trim。
* `kv_sync_thread`：队列为kv\_queue、deferred\_done\_queue、deferred\_stable\_queue，sync元数据和数据。
* `kv_finalize_thread`：队列为kv\_committing\_to\_finalize、deferred\_stable\_to\_finalize，执行清理功能。
* `deferred_finisher`：调用回调函数提交DeferredIO的请求。
* `finishers`：多个回调线程Finisher，通知用户请求完成。

我们主要分析`aio_thread`、`kv_sync_thread`、`kv_finalize_thread`。

#### aio_thread

aio\_thread比较简单，属于KernelDevice模块的，主要作用是收割完成的aio事件，并触发回调函数。

```
void KernelDevice::_aio_thread() {
	while (!aio_stop) {
		......
		// 获取完成的aio
		int r = aio_queue.get_next_completed(cct->_conf->bdev_aio_poll_ms, aio, max);
		// 设置flush标志为true。
		io_since_flush.store(true);
		// 获取aio的返回值
		long r = aio[i]->get_return_value();
		......
		// 调用aio完成的回调函数
		if (ioc->priv) {
			if (--ioc->num_running == 0) {
				aio_callback(aio_callback_priv, ioc->priv);
			}
		}       
	}
}
```

**涉及延迟指标**：`state_aio_wait_lat`、`state_io_done_lat`。

#### kv\_sync\_thread

当IO完成后，要么将txc放入队列，要么将dbh放入队列，虽然对应不同队列，但都是由`kv_sync_thread`执行后续操作。

对于SimpleWrite，都是写新的磁盘block(如果是cow，也是写新的block，只是事务中k/v操作增加对旧的block的回收操作)，所以先由aio\_thread写block，再由kv\_sync\_thread同步元信息，无论什么时候挂掉，数据都不会损坏。

对于DeferredWrite，在事物的prepare阶段将需要DeferredWrite的数据作为k/v对(也称为WAL)写入基于RocksDB封装的db\_transaction中，此时还在内存，kv\_sync\_thread第一次的commit操作中，将wal持久化在了k/v系统中，然后进行后续的操作，异常的情况，可以通过回放wal，数据也不会损坏。

kv\_sync\_thread主要执行的操作为：在Libaio写完数据后，需要通过kv\_sync\_thread更新元数据k/v，主要包含object的Onode、扩展属性、FreelistManager的磁盘空间信息等等，这些必须按顺序操作。

涉及的队列如下：

* `kv_queue`：需要执行commit的txc队列。将kv\_queue中的txc存入kv\_committing中，并提交给RocksDB，即执行操作db\-\>submit\_transaction，设置状态为STATE\_KV\_SUBMITTED，并将kv\_committing中的txc放入kv\_committing\_to\_finalize，等待线程kv\_finalize\_thread执行。
* `deferred_done_queue`：已经完成DeferredIO操作的dbh队列，还没有sync磁盘。这个队列的dbh会有两种结果: 1) 如果没有做flush操作，会将其放入deferred\_stable\_queue待下次循环继续处理 2) 如果做了flush操作，说明数据已经落盘，即已经是stable的了，直接将其插入deferred\_stable\_queue队列。这里stable的意思就是数据已经sync到磁盘了，前面RocksDB中记录的wal没用可以删除了。
* `deferred_stable_queue`：DeferredIO已经落盘，等待清理RocksDB中的WAL。依次操作dbh中的txc，将RocksDB中的wal删除，然后dbh入队列deferred\_stable\_to\_finalize，等待线程kv\_finalize\_thread执行。

```
void BlueStore::_kv_sync_thread() {
	while (true) {
		// 交换指针
		kv_committing.swap(kv_queue);
		kv_submitting.swap(kv_queue_unsubmitted);
		deferred_done.swap(deferred_done_queue);
		deferred_stable.swap(deferred_stable_queue)
		
		// 处理 deferred_done_queue
		if (force_flush) {
			// flush/barrier on block device
			bdev->flush();
			// if we flush then deferred done are now deferred stable
			deferred_stable.insert(deferred_stable.end(),
                                       deferred_done.begin(),
                                       deferred_done.end());
			deferred_done.clear();
		}
		
		// 处理 kv_queue
		for (auto txc : kv_committing) {
			int r = cct->_conf->bluestore_debug_omit_kv_commit
                                ? 0
                                : db->submit_transaction(txc->t);
         	_txc_applied_kv(txc);
		}
		
		// 处理 deferred_stable_queue
		for (auto b : deferred_stable) {
			for (auto &txc : b->txcs) {
				get_deferred_key(wt.seq, &key);
				synct->rm_single_key(PREFIX_DEFERRED, key);
 			}
		}
		
		// submit synct synchronously (block and wait for it to commit)
		// 同步kv，有设置bluefs_extents、删除wal两种操作
		int r = cct->_conf->bluestore_debug_omit_kv_commit
                        ? 0
                        : db->submit_transaction_sync(synct);
		
		// 放入finalize线程队列，并通知其处理。
		std::unique_lock<std::mutex> m(kv_finalize_lock);
		kv_committing_to_finalize.swap(kv_committing);
		deferred_stable_to_finalize.swap(deferred_stable);
		kv_finalize_cond.notify_one();
	}
}
```

**涉及延迟指标**：`state_kv_queued_lat`、`state_kv_committing_lat`、`kv_lat`

#### kv\_finalize\_thread

清理线程，包含两个队列：

* `kv_committing_to_finalize`：再次调用\_txc\_state\_proc进入状态机，设置状态为STATE\_KV\_DONE，并执行回调函数通知用户io操作完成。
* `deferred_stable_to_finalize`：遍历deferred\_stable中的dbh，调用\_txc\_state\_proc进入状态机，设置状态为STATE_FINISHING，继续调用\_txc\_finish，设置状态为STATE\_DONE，状态机结束，事物完成。

```
void BlueStore::_kv_finalize_thread() {
	while (true) {
		// 交换指针
		kv_committed.swap(kv_committing_to_finalize);
		deferred_stable.swap(deferred_stable_to_finalize);
		
		// 处理kv_committing_to_finalize队列
		while (!kv_committed.empty()) {
			TransContext *txc = kv_committed.front();
			_txc_state_proc(txc);
			kv_committed.pop_front();
		}
		
		// 处理deferred_stable_to_finalize
		for (auto b : deferred_stable) {
			auto p = b->txcs.begin();
			while (p != b->txcs.end()) {
				TransContext *txc = &*p;
					p = b->txcs.erase(p);  // unlink here because
					_txc_state_proc(txc);  // this may destroy txc
				}
				delete b;
		}
		deferred_stable.clear();
	}
}
```

**涉及延迟指标**：`state_deferred_cleanup_lat`、`state_finishing_lat`

### <a name="chapter5"></a>IO状态

主要分为SimpleWrite、DeferredWrite、SimpleWrite+DeferredWrite。

已有文章写的不错，这里便不再详细描述(写文章+画图真是体力活: )，可参见：[Ceph BlueStore](http://blog.wjin.org/posts/ceph-bluestore.html) Write Type章节。

### <a name="chapter6"></a>最后YY

BlueStore源码分析的系列文章就到此结束了，也算是学习BlueStore的一个总结。貌似还差BlueFS的分析，已有大神写出[BlueStore-先进的用户态文件系统《二》-BlueFS](https://zhuanlan.zhihu.com/p/46362124)。

由于个人水平有限，对BlueStore也仅限于粗浅的理解，如有错误，还请指出。

### 参考资源

* [Ceph BlueStore](http://blog.wjin.org/posts/ceph-bluestore.html)
* [ceph bluestore工作流程](http://www.sysnote.org/2016/08/25/bluestore-work-flow/)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [BlueStore源码分析之事物状态机](https://shimingyah.github.io/2019/11/BlueStore%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%B9%8B%E4%BA%8B%E7%89%A9%E7%8A%B6%E6%80%81%E6%9C%BA/)