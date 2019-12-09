---
layout: post
title: "浅谈Linux内核IO体系之磁盘IO"
date: 2019-12-06
description: "浅谈Linux内核IO体系之磁盘IO"
tag: IO体系
---

### 前言

Linux I/O体系是Linux内核的重要组成部分，主要包含网络IO、磁盘IO等。基本所有的技术栈都需要与IO打交道，分布式存储系统更是如此。本文主要简单分析一下磁盘IO，看看一个IO请求从发起到完成到底经历了哪些流程。

### 目录

* [名词解释](#chapter1)
* [IO体系](#chapter2)
* [VFS层](#chapter3)
* [PageCache层](#chapter4)
* [映射层](#chapter5)
* [通用块层](#chapter6)
* [IO调度层](#chapter7)
* [设备驱动层](#chapter8)
* [物理设备层](#chapter9)
* [FAQ](#chapter10)

### <a name="chapter1"></a>名词解释

* `Buffered I/O`：缓存IO又叫标准IO，是大多数文件系统的默认IO操作，经过PageCache。
* `Direct I/O`：直接IO，By Pass PageCache。offset、length需对齐到block_size。
* `Sync I/O`：同步IO，即发起IO请求后会阻塞直到完成。缓存IO和直接IO都属于同步IO。
* `Async I/O`：异步IO，即发起IO请求后不阻塞，内核完成后回调。通常用内核提供的Libaio。
* `Write Back`：Buffered IO时，仅仅写入PageCache便返回，不等数据落盘。
* `Write Through`：Buffered IO时，不仅仅写入PageCache，而且同步等待数据落盘。

### <a name="chapter2"></a>IO体系

我们先看一张总的Linux内核存储栈图片：

![](http://img-ys011.didistatic.com/static/anything/io_stack.jpeg)

Linux IO存储栈主要有以下7层：

![](http://img-ys011.didistatic.com/static/anything/linux_io_stack.png)

### <a name="chapter3"></a>VFS层

我们通常使用open、read、write等函数来编写Linux下的IO程序。接下来我们看看这些函数的IO栈是怎样的。在此之前我们先简单分析一下VFS层的4个对象，有助于我们深刻的理解IO栈。

**VFS层的作用是屏蔽了底层不同的文件系统的差异性，为用户程序提供一个统一的、抽象的、虚拟的文件系统，提供统一的对外API，使用户程序调用时无需感知底层的文件系统，只有在真正执行读写操作的时候才调用之前注册的文件系统的相应函数。**

VFS支持的文件系统主要有三种类型：

* 基于磁盘的文件系统：Ext系列、XFS等。
* 网络文件系统：NFS、CIFS等。
* 特殊文件系统：/proc、裸设备等。

VFS主要有四个对象类型(不同的文件系统都要实现)：

* superblock：整个文件系统的元信息。对应的操作结构体：`struct super_operations`。
* inode：单个文件的元信息。对应的操作结构体：`struct inode_operations`。
* dentry：目录项，一个文件目录对应一个dentry。对应的操作结构体：`struct dentry_operations`。
* file：进程打开的一个文件。对应的操作结构体：`struct file_operations`。

关于VFS相关结构体的定义都在[include/linux/fs.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/fs.h)里面。

#### superblock

superblock结构体定义了整个文件系统的元信息，以及相应的操作。

[https://github.com/torvalds/linux/blob/v4.16/fs/xfs/xfs_super.c#L1789](https://github.com/torvalds/linux/blob/v4.16/fs/xfs/xfs_super.c#L1789)

```
static const struct super_operations xfs_super_operations = {
	......
};

static struct file_system_type xfs_fs_type = {
	.name			= "xfs",
	......
};
```

#### inode

inode结构体定义了文件的元数据，比如大小、最后修改时间、权限等，除此之外还有一系列的函数指针，指向具体文件系统对文件操作的函数，包括常见的open、read、write等，由`i_fop`函数指针提供。

文件系统最核心的功能全部由inode的函数指针提供。主要是inode的`i_op`、`i_fop`字段。

```
struct inode {
	......
	// inode 文件元数据的函数操作
	const struct inode_operations	*i_op;
	
	// 文件数据的函数操作，open、write、read等
	const struct file_operations	*i_fop;
	......
}
```

在设置inode的`i_fop`时候，会根据不同的inode类型设置不同的`i_fop`。我们以xfs为例：

[https://github.com/torvalds/linux/blob/v4.16/fs/xfs/xfs_iops.c#L1266](https://github.com/torvalds/linux/blob/v4.16/fs/xfs/xfs_iops.c#L1266)

[https://github.com/torvalds/linux/blob/v4.16/fs/inode.c#L1980](https://github.com/torvalds/linux/blob/v4.16/fs/inode.c#L1980)

**如果inode类型为普通文件的话，那么设置XFS提供的`xfs_file_operations`。**

**如果inode类型为块设备文件的话，那么设置块设备默认提供的`def_blk_fops`。**

```
void xfs_setup_iops(struct xfs_inode *ip)
{
	struct inode *inode = &ip->i_vnode;

	switch (inode->i_mode & S_IFMT) {
	case S_IFREG:
		inode->i_op = &xfs_inode_operations;
		// 在IO栈章节会分析一下xfs_file_operations
		inode->i_fop = &xfs_file_operations;
		inode->i_mapping->a_ops = &xfs_address_space_operations;
		break;
	......
	default:
		inode->i_op = &xfs_inode_operations;
		init_special_inode(inode, inode->i_mode, inode->i_rdev);
		break;
	}
}

void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev)
{
	inode->i_mode = mode;
	......
	if (S_ISBLK(mode)) {
		// 块设备相应的系列函数
		inode->i_fop = &def_blk_fops;
		inode->i_rdev = rdev;
	}
	......
}
```

#### dentry

dentry是目录项，由于每一个文件必定存在于某个目录内，我们通过路径查找一个文件时，最终肯定找到某个目录项。在Linux中，目录和普通文件一样，都是存放在磁盘的数据块中，在查找目录的时候就读出该目录所在的数据块，然后去寻找其中的某个目录项。

```
struct dentry {
	......
	const struct dentry_operations *d_op;
	......
};
```

在我们使用Linux的过程中，根据目录查找文件的例子无处不在，而目录项的数据又都是存储在磁盘上的，如果每一级路径都要读取磁盘，那么性能会十分低下。所以需要目录项缓存，把dentry放在缓存中加速。

VFS把所有的dentry放在dentry\_hashtable哈希表里面，使用LRU淘汰算法。

#### file

用户程序能接触的VFS对象只有`file`，由进程管理。我们常用的打开一个文件就是创建一个file对象，并返回一个文件描述符。出于隔离性的考虑，内核不会把file的地址返回，而是返回一个整形的fd。

```
struct file {
	// 操作文件的函数指针，和inode里面的i_fop一样，在open的时候赋值为i_fop。
	const struct file_operations *f_op;
	
	// 指向对应inode对象
	struct inode *f_inode;
	
	// 每个文件都有自己的一个偏移量
	loff_t f_pos;
	......
}
```

file对象是由内核进程直接管理的。每个进程都有当前打开的文件列表，放在files\_struct结构体中。

```
struct files_struct {
	......
	struct file __rcu * fd_array[NR_OPEN_DEFAULT];
	......
};
```

**fd\_array数组存储了所有打开的file对象，用户程序拿到的文件描述符(fd)实际上是这个数组的索引。**

#### IO栈

[https://github.com/torvalds/linux/blob/v4.16/fs/read_write.c#L566](https://github.com/torvalds/linux/blob/v4.16/fs/read_write.c#L566)

```
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
	......
	ret = vfs_read(f.file, buf, count, &pos);
	......
	return ret;
}
SYSCALL_DEFINE3(write, unsigned int, fd, char __user *, buf, size_t, count)
{
	......
	ret = vfs_write(f.file, buf, count, &pos);
	......
	return ret;
}
```

**由此可见，我们经常使用的read、write系统调用实际上是对vfs\_read、vfs\_write的一个封装。**

```
size_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
	......
	if (file->f_op->read)    
		ret = file->f_op->read(file, buf, count, pos);
	else
		ret = do_sync_read(file, buf, count, pos);
	......
}
ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
	......
	if (file->f_op->write)
		ret = file->f_op->write(file, buf, count, pos);    
	else
		ret = do_sync_write(file, buf, count, pos);
	......
}
```

我们发现，VFS会调用具体的文件系统的实现：`file->f_op->read`、`file->f_op->write`。

对于通用的文件系统，Linux封装了很多基本的函数，很多文件系统的核心功能都是以这些基本的函数为基础，再封装一层。接下来我们以XFS为例，简单分析一下XFS的read、write都做了什么操作。

[https://github.com/torvalds/linux/blob/v4.16/fs/xfs/xfs_file.c#L1137](https://github.com/torvalds/linux/blob/v4.16/fs/xfs/xfs_file.c#L1137)

```
const struct file_operations xfs_file_operations = {
	......
	.llseek		= xfs_file_llseek,
	.read		= do_sync_read,
	.write		= do_sync_write,
	// 异步IO，在之后的版本中名字为read_iter、write_iter。
	.aio_read	= xfs_file_aio_read,
	.aio_write  = xfs_file_aio_write,
	.mmap		= xfs_file_mmap,
	.open		= xfs_file_open,
	.fsync		= xfs_file_fsync,
	......
};
```

这是XFS的`f_op`函数指针表，我们可以看到read、write函数直接使用了内核提供的`do_sync_read`、`do_sync_write`函数。

```
ssize_t do_sync_read(struct file *filp, char __user *buf, size_t len, loff_t *ppos)
{
	......
	ret = filp->f_op->aio_read(&kiocb, &iov, 1, kiocb.ki_pos);
	......
}
ssize_t do_sync_write(struct file *filp, const char __user *buf, size_t len, loff_t *ppos)
{
	......
	ret = filp->f_op->aio_write(&kiocb, &iov, 1, kiocb.ki_pos);
	......
}
```

这两个函数最终也是调用了具体文件系统的`aio_read`和`aio_write`函数，对应XFS的函数为`xfs_file_aio_read`和`xfs_file_aio_write`。

`xfs_file_aio_read`和`xfs_file_aio_write`虽然有很多xfs自己的实现细节，但其核心功能都是建立在内核提供的通用函数上的：`xfs_file_aio_read`最终会调用`generic_file_aio_read`函数，`xfs_file_aio_write`最终会调用`generic_perform_write`函数，这些通用函数是基本上所有文件系统的核心逻辑。

接下来便要进入PageCache层的相关逻辑了，我们先简单概括一下读写多了哪些事情。

`generic_file_aio_read`：

1. 根据文件偏移量计算出要读取数据在PageCache中的位置。
2. 如果命中PageCache则直接返回，否则触发磁盘读取任务，会有预读的操作，减少IO次数。
3. 数据读取到PageCache后，拷贝到用户态Buffer中。

`generic_perform_write`：

1. 根据文件偏移量计算要写入的数据再PageCache中的位置。
2. 将用户态的Buffer拷贝到PageCache中。
3. 检查PageCache是否占用太多，如果是则将部分PageCache的数据刷回磁盘。

使用Buffered IO时，VFS层的读写很大程度上是依赖于PageCache的，只有当Cache-Miss，Cache过满等才会涉及到磁盘的操作。

#### 块设备文件

我们在使用Direct IO时，通常搭配Libaio使用，避免同步IO阻塞程序。而往往Direct IO + Libaio应用于裸设备的场景，尽量不要应用于文件系统中的文件，这时仍然会有文件系统的种种开销。

通常Direct IO + Libaio使用的场景有几种： 

* write back journal，journal也是裸设备。
* 不怎么依赖文件系统的绝大部分功能，仅仅是读写即可，可直接操作裸设备。

上面基本都是普通文件的读写，我们通常的使用场景中还有一种特殊的文件即块设备文件(/dev/sdx)，这些块设备文件仍然由VFS层管理，相当于一个特殊的文件系统。当进程访问块设备文件时，直接调用设备驱动程序提供的相应函数，默认的块设备函数列表如下：

```
const struct file_operations def_blk_fops = {
	......
	.open		= blkdev_open,
	.llseek		= block_llseek,
	.read		= do_sync_read,
	.write		= do_sync_write,
	.aio_read	= blkdev_aio_read,
	.aio_write	= blkdev_aio_write,
	.mmap		= generic_file_mmap,
	.fsync		= blkdev_fsync,
	......
};
```

使用Direct IO + Libaio + 裸设备时，VFS层的函数指针会指向裸设备的`def_blk_fops`。因为我们通常使用DIO+Libaio+裸设备，所以我们简单分析一下Libaio的IO流程。

Libaio提供了5个基本的方法，只能以DIO的方式打开，否则可能会进行Buffered IO。

```
io_setup, io_cancal, io_destroy, io_getevents, io_submit
```

Linux内核AIO的实现在[https://github.com/torvalds/linux/blob/v4.16/fs/aio.c](https://github.com/torvalds/linux/blob/v4.16/fs/aio.c)，我们简单分析一下io_submit的操作。

```
SYSCALL_DEFINE3(io_submit, aio_context_t, ctx_id, long, nr,
		struct iocb __user * __user *, iocbpp)
{
	return do_io_submit(ctx_id, nr, iocbpp, 0);
}
long do_io_submit(aio_context_t ctx_id, long nr,struct iocb __user *__user *iocbpp, bool compat){
    ...
    for (i=0; i<nr; i++) {
        ret = io_submit_one(ctx, user_iocb, &tmp, compat);
    }
    ...
}
static int io_submit_one(struct kioctx *ctx, struct iocb __user *user_iocb,struct iocb *iocb, bool compat){
    ...
    ret = aio_run_iocb(req, compat);
    ...
}
static ssize_t aio_run_iocb(struct kiocb *req, bool compat){
    ...
    case IOCB_CMD_PREADV:
		rw_op = file->f_op->aio_read;
    case IOCB_CMD_PWRITEV:
    	rw_op = file->f_op->aio_write;
    ...
}
```

可以发现，最终也是调用`f_op`的`aio_read`函数，对应于文件系统的文件就是`xfs_file_aio_read`函数，对应于块设备文件就是`blkdev_aio_read`函数，然后进入通用块层，放入IO队列，进行IO调度。由此可见Libaio的队列也就是通用块层之下的IO调度层中的队列。

### <a name="chapter4"></a>PageCache层

在HDD时代，由于内核和磁盘速度的巨大差异，Linux内核引入了页高速缓存(PageCache)，把磁盘抽象成一个个固定大小的连续Page，通常为4K。对于VFS来说，只需要与PageCache交互，无需关注磁盘的空间分配以及是如何读写的。

当我们使用Buffered IO的时候便会用到PageCache层，与Direct IO相比，用户程序无需offset、length对齐。是因为通用块层处理IO都必须是块大小对齐的。

Buffered IO中PageCache帮我们做了对齐的工作：如果我们修改文件的offset、length不是页大小对齐的，那么PageCache会执行`RMW`的操作，先把该页对应的磁盘的数据全部读上来，再和内存中的数据做Modify，最后再把修改后的数据写回磁盘。虽然是写操作，但是非对齐的写仍然会有读操作。

Direct IO由于跳过了PageCache，直达通用块层，所以需要用户程序处理对齐的问题。

#### 脏页刷盘

如果发生机器宕机，位于PageCache中的数据就会丢失；所以仅仅写入PageCache是不可靠的，需要有一定的策略将数据刷入磁盘。通常有几种策略：

* 手动调用fsync、fdatasync刷盘，可参考[浅谈分布式存储之sync详解](https://shimingyah.github.io/2019/10/%E6%B5%85%E8%B0%88%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E4%B9%8Bsync%E8%AF%A6%E8%A7%A3/)。
* 脏页占用比例超过了阈值，触发刷盘。
* 脏页驻留时间过长，触发刷盘。

Linux内核目前的做法是为每个磁盘都建立一个线程，负责每个磁盘的刷盘。

#### 预读策略

从VFS层我们知道写是异步的，写完PageCache便直接返回了；但是读是同步的，如果PageCache没有命中，需要从磁盘读取，很影响性能。如果是顺序读的话PageCache便可以进行预读策略，异步读取该Page之后的Page，等到用户程序再次发起读请求，数据已经在PageCache里，大幅度减少IO的次数，不用阻塞读系统调用，提升读的性能。

### <a name="chapter5"></a>映射层

映射层是在PageCache之下的一层，由多个文件系统(Ext系列、XFS等，打开文件系统的文件)以及块设备文件(直接打开裸设备文件)组成，主要完成两个工作：

* 内核确定该文件所在文件系统或者块设备的块大小，并根据文件大小计算所请求数据的长度以及所在的逻辑块号。
* 根据逻辑块号确定所请求数据的物理块号，也即在在磁盘上的真正位置。

由于通用块层以及之后的的IO都必须是块大小对齐的，我们通过DIO打开文件时，略过了PageCache，所以必须要自己将IO数据的offset、length对齐到块大小。

我们使用的DIO+Libaio直接打开裸设备时，跳过了文件系统，少了文件系统的种种开销，然后进入通用块层，继续之后的处理。

### <a name="chapter6"></a>通用块层

通用块层存在的意义也和VFS一样，屏蔽底层不同设备驱动的差异性，提供统一的、抽象的通用块层API。

通用块层最核心的数据结构便是`bio`，描述了从上层提交的一次IO请求。

[https://github.com/torvalds/linux/blob/v4.16/include/linux/blk_types.h#L96](https://github.com/torvalds/linux/blob/v4.16/include/linux/blk_types.h#L96)

```
struct bio {
	......
	// 要提交到磁盘的多段数据
	struct bio_vec *bi_io_vec;
	
	// 有多少段数据
	unsigned short bi_vcnt;
	......
}
struct bio_vec {
	struct page	*bv_page;
	unsigned int	bv_len;
	unsigned int	bv_offset;
};
```

所有到通用块层的IO，都要把数据封装成`bio_vec`的形式，放到`bio`结构体内。

在VFS层的读请求，是以Page为单位读取的，如果改Page不在PageCache内，那么便要调用文件系统定义的read\_page函数从磁盘上读取数据。

```
const struct address_space_operations xfs_address_space_operations = {
	......
	.readpage		= xfs_vm_readpage,
	.readpages		= xfs_vm_readpages,
	.writepage		= xfs_vm_writepage,
	.writepages		= xfs_vm_writepages,
	......
};
```

### <a name="chapter7"></a>IO调度层

Linux调度层是Linux IO体系中的一个重要组件，介于通用块层和块设备驱动层之间。IO调度层主要是为了减少磁盘IO的次数，增大磁盘整体的吞吐量，会队列中的多个bio进行排序和合并，并且提供了多种IO调度算法，适应不同的场景。

Linux内核为每一个块设备维护了一个IO队列，item是`struct request`结构体，用来排队上层提交的IO请求。一个request包含了多个bio，一个IO队列queue了多个request。

```
struct request {
	......
	// total data len
	unsigned int __data_len;
	
	// sector cursor
	sector_t __sector;
	
	// first bio
	struct bio *bio;
	
	// last bio
	struct bio *biotail;
	......
}
```

上层提交的bio有可能分配一个新的request结构体去存放，也有可能合并到现有的request中。

Linux内核目前提供了以下几种调度策略：

* Deadline：默认的调度策略，加入了超时的队列。适用于HDD。
* CFQ：完全公平调度器。
* Noop：No Operation，最简单的FIFIO队列，不排序会合并。适用于SSD、NVME。

### <a name="chapter8"></a>块设备驱动层

每一类设备都有其驱动程序，负责设备的读写。IO调度层的请求也会交给相应的设备驱动程序去进行读写。大部分的磁盘驱动程序都采用DMA的方式去进行数据传输，DMA控制器自行在内存和IO设备间进行数据传送，当数据传送完成再通过中断通知CPU。

通常块设备的驱动程序都已经集成在了kernel里面，也即就算我们直接调用块设备驱动驱动层的代码还是要经过内核。

[spdk](https://github.com/spdk/spdk)实现了用户态、异步、无锁、轮询方式NVME驱动程序。块存储是延迟非常敏感的服务，使用NVME做后端存储磁盘时，便可以使用spdk提供的NVME驱动，缩短IO流程，降低IO延迟，提升IO性能。

### <a name="chapter9"></a>物理设备层

物理设备层便是我们经常使用的HDD、SSD、NVME等磁盘设备了。

### <a name="chapter10"></a>FAQ

#### 1、write返回成功数据落盘了吗？

Buffered IO：write返回数据仅仅是写入了PageCache，还没有落盘。

Direct IO：write返回数据仅仅是到了通用块层放入IO队列，依旧没有落盘。

此时设备断电、宕机仍然会发生数据丢失。需要调用fsync或者fdatasync把数据刷到磁盘上，调用命令时，磁盘本身缓存(DiskCache)的内容也会持久化到磁盘上。

#### 2、write系统调用是原子的吗？

write系统调用不是原子的，如果有多线程同时调用，数据可能会发生错乱。可以使用`O_APPEND`标志打开文件，只能追加写，这样多线程写入就不会发生数据错乱。

#### 3、mmap相比read、write快在了哪里？

mmap直接把PageCache映射到用户态，少了一次系统调用，也少了一次数据在用户态和内核态的拷贝。

mmap通常和read搭配使用：写入使用write+sync，读取使用mmap。

#### 4、为什么Direct IO需要数据对齐？

DIO跳过了PageCache，直接到通用块层，而通用块层的IO都必须是块大小对齐的，所以需要用户程序自行对齐offset、length。

#### 5、Libaio的IO栈？

```
write()--->sys_write()--->vfs_write()--->通用块层--->IO调度层--->块设备驱动层--->块设备
```

#### 6、为什么需要 by pass pagecache？

当应用程序不满Linux内核的Cache策略，有更适合自己的Cache策略时可以使用Direct IO跳过PageCache。例如Mysql。

#### 7、为什么需要 by pass kernel？

当应用程序对延迟极度敏感时，由于Linux内核IO栈有7层，IO路径比较长，为了缩短IO路径，降低IO延迟，可以by pass kernel，直接使用用户态的块设备驱动程序。例如spdk的nvme，阿里云的ESSD。

#### 8、为什么需要直接操作裸设备？

当应用程序仅仅使用了基本的read、write，用不到文件系统的大而全的功能，此时文件系统的开销对于应用程序来说是一种累赘，此时需要跳过文件系统，接管裸设备，自己实现磁盘分配、缓存等功能，通常使用DIO+Libaio+裸设备。例如Ceph FileStore的Journal、Ceph BlueStore。

### 参考资源

* [linux io过程自顶向下分析](https://my.oschina.net/fileoptions/blog/3058792/print)
* [Linux 中直接 I/O 机制的介绍](https://www.ibm.com/developerworks/cn/linux/l-cn-directio/)
* [Linux I/O 调度器](https://www.ibm.com/developerworks/cn/linux/l-lo-io-scheduler-optimize-performance/index.html)
* [Linux通用块设备层](http://www.ilinuxkernel.com/files/Linux.Generic.Block.Layer.pdf)
* [Linux虚拟文件系统](http://www.ilinuxkernel.com/files/Linux.Virtual.Filesystem.pdf)
* [Linux Kernel Exploration IO系统](http://ilinuxkernel.com/?cat=7)
* [Linux kernel source code](https://github.com/torvalds/linux)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [浅谈Linux内核IO体系之磁盘IO](https://shimingyah.github.io/2019/12/%E6%B5%85%E8%B0%88Linux%E5%86%85%E6%A0%B8IO%E4%BD%93%E7%B3%BB%E4%B9%8B%E7%A3%81%E7%9B%98IO/)