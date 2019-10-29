---
layout: post
title: "浅谈分布式存储之sync详解"
date: 2019-10-29
description: "浅谈分布式存储之sync详解"
tag: 分布式存储
---

### 前言

由于内存比磁盘读写速度快了好几个数量级，为了弥补磁盘IO性能低，Linux内核引入了页面高速缓存(PageCache)。我们通过Linux系统调用(open--->write)写文件时，内核会先将数据从用户态缓冲区拷贝到PageCache便直接返回成功，然后由内核按照一定的策略把脏页Flush到磁盘上，我们称之为`write back`。

write写入的数据是在内存的PageCache中的，一旦内核发生Crash或者机器Down掉，就会发生数据丢失，对于分布式存储来说，数据的可靠性是至关重要的，所以我们需要在write结束后，调用`fsync`或者`fdatasync`将数据持久化到磁盘上。

`write back`减少了磁盘的写入次数，但却降低了文件磁盘数据的更新速度，会有丢失更新数据的风险。为了保证磁盘文件数据和PageCache数据的一致性，Linux提供了`sync`、`fsync`、`msync`、`fdatasync`、`sync_file_range`5个函数。

### 目录

* [posix](#chapter1)
* [sync](#chapter2)
* [fsync](#chapter3)
* [msync](#chapter4)
* [fdatasync](#chapter5)
* [o_sync](#chapter6)
* [sync\_file\_range](#chapter7)

### <a name="chapter1"></a>posix

`POSIX(Portable Operating System Interface)`是一套可移植的操作系统接口，定义了操作系统应该为应用程序提供的接口。是一套先有实现再有规范的语义接口，现今大多数的Linux操作系统都兼容POSIX标准。

### <a name="chapter2"></a>sync

函数定义：`void sync(void);`

我们知道write系统调用只是写入到PageCache，脏页不会立刻写入到磁盘，而是由内核的flusher线程在满足一定阈值(一定时间间隔、脏页达到一定比例)，调用sync函数将脏页同步到磁盘上(放入设备的IO请求队列)。

POSIX语义要求sync系统调用只需将脏页提交到块设备IO队列就可以返回。所以我们看到sync函数返回值为void。同时sync函数返回后，并不等于写入磁盘结束，仍然会出现故障，此时sync函数是无法知晓的。对于可靠性要求比较高的应用，write提供的松散的异步语义是不够的，所以我们需要内核提供的同步IO来保证，常用`fsync`以及`fdatasync`。

sync函数是针对整个PageCache的，对所有的文件更新产生的脏页都会flush。

### <a name="chapter3"></a>fsync

函数定义：`int fsync(int fd);`

fsync针对单个文件起作用，会阻塞等到PageCache的更新数据真正写入到了磁盘才会返回。fsync除了更新文件的数据外，还会更新文件的元数据(大小、修改时间等)，适用于数据库这样的应用。

通常文件的数据和元数据是存储在磁盘不同位置的，因此fsync至少需要两次IO操作，一次数据、一次元数据。根据Wikipedia的数据，当前硬盘驱动的平均寻道时间（Average seek time）大约是3~15ms，7200RPM硬盘的平均旋转延迟（Average rotational latency）大约为4ms，因此一次IO操作的耗时大约为10ms左右。所以多一次IO操作时昂贵的。

### <a name="chapter4"></a>msync

函数定义：`int msync(void *addr, size_t length, int flags);`

如果采用内存映射文件的方式进行文件IO(mmap)也有类似的系统调用来确保修改的内容完全同步到硬盘之上。msync需要指定同步的地址区间，如此细粒度的控制似乎比fsync更加高效（因为应用程序通常知道自己的脏页位置），但实际上（Linux）kernel中有着十分高效的数据结构，能够很快地找出文件的脏页，使得fsync只会同步文件的修改内容，同时内核也提供了`sync_file_range`函数。

### <a name="chapter5"></a>fdatasync

函数定义：`int fdatasync(int fd);`

上文提到fsync会同步数据以及元数据，增大延迟，因此POSIX定义了fdatasync，放宽了同步的语义，以提高性能。

fdatasync的功能与fsync类似，但是仅仅在必要的情况下才会同步metadata，因此可以减少一次IO写操作。那么什么是“必要的情况”呢？

举例来说，文件的尺寸（st_size）如果变化，是需要立即同步的，否则OS一旦崩溃，即使文件的数据部分已同步，由于metadata没有同步，依然读不到修改的内容。而最后访问时间(atime)/修改时间(mtime)是不需要每次都同步的，只要应用程序对这两个时间戳没有苛刻的要求，基本无伤大雅。

### <a name="chapter6"></a>O_SYNC

open函数的`O_SYNC`和`O_DSYNC`参数有着和`fsync`及`fdatasync`类似的含义：使每次write都会阻塞到磁盘IO完成。

O_SYNC：使每次write操作阻塞等待磁盘IO完成，文件数据和文件属性都更新。

O_DSYNC：使每次write操作阻塞等待磁盘IO完成，但是如果该写操作并不影响读取刚写入的数据，则不需等待文件属性被更新。

`O_DSYNC`和`O_SYNC`标志有微妙的区别：

文件以`O_DSYNC`标志打开时，仅当文件属性需要更新以反映文件数据变化（例如，更新文件大小以反映文件中包含了更多数据）时，标志才影响文件属性。在重写其现有的部分内容时，文件时间属性不会同步更新。

文件以`O_SYNC`标志打开时，数据和属性总是同步更新。对于该文件的每一次write都将在write返回前更新文件时间，这与是否改写现有字节或追加文件无关。相对于fsync/fdatasync，这样的设置不够灵活，应该很少使用。

实际上：Linux对`O_SYNC`、`O_DSYNC`做了相同处理，没有满足POSIX的要求，而是都实现了`fdatasync`的语义。

### <a name="chapter7"></a>sync\_file\_range

IO密集型的程序如果频繁的刷盘，会有很大的性能问题。Linux在内核2.6.17之后支持了`sync_file_range`，可以让我们在做多个更新后，一次性的刷数据，这样大大提高IO的性能。

`sync_file_range`可以将文件的部分范围作为目标，将对应范围内的脏页刷回磁盘，而不是整个文件的范围。好处是，当我们对大文件进行了修改时，如果修改了大量的数据块，我们最后fsync的时候，可能会很慢。即使fdatasync，也是有问题的，例如这个大文件的长度在我们的修改过程中发生了变化，那么fdatasync将同时写metadata，而对于文件系统来说，单个文件系统的写metadata是串行的，这势必导致影响其他用户操作metadata(如创建文件)。

`sync_file_range`是绝对不会写metadata的，所以用它非常合适，每次对文件做了小范围的修改时，立即调用`sync_file_range`，把对应的脏数据刷到磁盘，那么在结束对文件的修改后，再调用fdatasync (flush dirty data page)、fsync(flush dirty data+metadata page)都是很快的。

`sync_file_range`提供了几个flag：

* `SYNC_FILE_RANGE_WRIT`：是异步的，可以结合fsync、fdatasync使用。
* `SYNC_FILE_RANGE_WAIT_BEFORE`：写前做一次全文件范围的`sync_file_range`。从而保证在调用fdatasync或fsync前，该文件的dirty page已经全部刷到磁盘。
* `SYNC_FILE_RANGE_WAIT_AFTER`：写后做一次全文件范围的`sync_file_range`。从而保证在调用fdatasync或fsync前，该文件的dirty page已经全部刷到磁盘。

### 参考资源

* [Linux IO同步函数:sync、fsync、fdatasync](http://byteliu.com/2019/03/09/Linux-IO%E5%90%8C%E6%AD%A5%E5%87%BD%E6%95%B0-sync%E3%80%81fsync%E3%80%81fdatasync/)
* [Linux下新系统调用sync\_file\_range提高数据sync的效率](http://www.voidcn.com/article/p-vwagcekw-bkq.html)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [浅谈分布式存储之sync详解](https://shimingyah.github.io/2019/10/%E6%B5%85%E8%B0%88%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E4%B9%8Bsync%E8%AF%A6%E8%A7%A3/)