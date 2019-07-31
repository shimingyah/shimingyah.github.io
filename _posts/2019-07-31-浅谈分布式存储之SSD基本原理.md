---
layout: post
title: "浅谈分布式存储之SSD基本原理"
date: 2019-07-31
description: "浅谈分布式存储之SSD基本原理"
tag: 分布式存储
---

### 前言

随着制造工艺的不断进步，SSD(Solid State Drive)性能和容量不断突破，价格不断降低，迎来了快速的发展，目前已经是商用服务器、高性能存储服务中非常流行的存储介质。作为开发人员，需要了解SSD的基本原理，以便开发时能更好地发挥其优势，规避其劣势。本文章基于末尾所列参考文献整理而来。

### 目录

* [SSD简介](#chapter1)
* [闪存基础](#chapter2)
* [存储原理](#chapter3)
* [读写流程](#chapter4)
* [GC机制](#chapter5)
* [Trim机制](#chapter6)
* [Bit-Error](#chapter7)
* [Read-Disturb](#chapter8)
* [Program-Disturb](#chapter9)
* [Wear-Leveling](#chapter10)
* [IO抖动因素](#chapter11)

### <a name="chapter1"></a>SSD简介

SSD诞生于上世纪70年代，最早的SSD使用RAM作为存储介质，但是RAM掉电后数据就会丢失，同时价格也特别贵。后来出现了基于闪存(Flash)的SSD，Flash掉电后数据不会丢失，因此Flash-SSD慢慢取代了RAM-SSD，但是此时HDD已经占据了大部分的市场。到本世纪初，随着制造工艺的不断进步，SDD迎来了长足的发展，同时HDD在工艺和技术上已经很难有突破性的进展，SSD在性能和容量上还在不断突破，相信在不久的将来，SSD在在线存储领域会取代HDD，成为软件定义存储(SDS)的主流设备。

### <a name="chapter2"></a>闪存基础

SSD主要由SSD控制器，Flash存储阵列，板上DRAM(可选)以及跟HOST接口(SATA、SAS、PCIe等)组成。

Flash的基本存储单元是`浮栅晶体管`，同时根据制造工艺分为`NOR`型和`NAND`型。`NAND`容量大，按照Page进行读写，适合进行数据存储，基本上存储使用的SSD的Flash都是`NAND`。

Flash的工作原理和场效应管类似都是利用电压控制源极和漏极之间的通断来工作的。

`写操作`**是在控制极加正电压，使电子通过绝缘层进入浮栅极。因此写操作无法将电子从浮栅极吸出，所以覆盖写入前必须擦除**。

`擦除(erase)操作`**正好相反，是在衬底加正电压，把电子从浮栅极中吸出来**。

`读操作`**给控制栅加读取电压，判断漏极-源极之间是否处于导通状态，进而可以判断浮置栅有没有存储电荷，进而判断该存储单元是`1`还是`0`**。

图片引用自[https://www.ssdfans.com](https://www.ssdfans.com)
![](http://img-ys011.didistatic.com/static/anything/flash.png)

### <a name="chapter3"></a>存储原理

如第二节-闪存基础，SSD内部一般都是使用NAND-Flash作为存储介质，逻辑结构如下图：

![](http://img-ys011.didistatic.com/static/anything/nand-flash.png)

SSD中一般有多个NAND-Flash，每个NAND-Flash包含多个Block，每个Block包含多个Page。由于NAND的特性，存取都必须以Page为单位，即每次读写至少是一个Page。通常地，每个Page的大小为4K或者8K。NAND的另一个特性是只能读写单个Page，不能覆盖写某个Page，如果要覆盖写，必须先要清空里面的内容，再写入。由于清空内容的电压较高，必须是以Block为单位，因此，没有空闲的Page时，必须要找到没有有效内容的Block，先擦除再选择空闲的Page写入。理论上也是可以设计成为按字节擦除，但是NAND容量一般很大，按字节擦除效率低，速度慢，所以就设计为按Block擦除了。

SSD中也会维护一个mapping table，维护逻辑地址到物理地址的映射。每次读写时，可以通过逻辑地址直接查表计算出物理地址，与传统的机械磁盘相比，省去了寻道时间和旋转时间。

### <a name="chapter4"></a>读写流程

从NAND-Flash的原理可以看出，其和HDD的主要区别为：

* **定位数据快**：HDD需要经过寻道和旋转，才能定位到要读写的数据块，而SSD通过mapping table直接计算即可。
* **读取速度快**：HDD的速度取决于旋转速度，而SSD只需要加电压读取数据，一般而言，要快于HDD。

在**顺序读测试**中，由于定位数据只需要一次，定位之后，则是大批量的读取数据的过程，此时，HDD和SSD的性能差距主要体现在读取速度上，HDD能到200M左右，而普通SSD是其两倍。

在**随机读测试**中，由于每次读都要先定位数据，然后再读取，HDD的定位数据的耗费时间很多，一般是几毫秒到十几毫秒，远远高于SSD的定位数据时间(一般0.1ms左右)，因此，随机读写测试主要体现在两者定位数据的速度上，此时，SSD的性能是要远远好于HDD的。

SSD的写分为**新写入**和**覆盖写**两种，处理流程不同。

#### 新写入
![](http://img-ys011.didistatic.com/static/anything/write-flow.png)

1. 找到一个空闲Page。
2. 数据写入到空闲Page。
3. 更新mapping table。

#### 覆盖写
![](http://img-ys011.didistatic.com/static/anything/update-flow.png)

1. SSD不能覆盖写，因此先找到一个空闲页H。
2. 读取Page-G中的数据到SSD内部的buffer中，把更新的字节更新到buffer。
3. buffer中的数据写入到H。
4. 更新mapping table中G页，置为无效页。
5. 更新mapping table中H页，添加映射关系。

如果在覆盖写操作比较多的情况下，会产生较多的无效页，类似于磁盘碎片，此时需要SSD的GC机制来回收这部分空间了。
### <a name="chapter5"></a>GC机制

在讨论GC机制之前，我们先了解一下`Over-Provisioning`是指SSD实际的存储空间比可写入的空间要大。比如一块SSD实际空间128G，可用容量却只有120G。为什么需要Over-Provisioning呢？请看如下例子：
![](http://img-ys011.didistatic.com/static/anything/over-provising.png)

如上图所示，假设系统中只有两个Block，最终还剩下两个无效的Page。此时要写入一个新Page，根据NAND原理，必须要先对两个无效的Page擦除才能用于写入。而擦除的粒度是Block，需要读取当前Block有效数据到新的Block，如果此时没有额外的空间，便做不了擦除操作了，那么最终那两个无效的Page便不能得到利用。所以需要SSD提供额外空间即`Over-Provisioning`，保证GC的正常运行。

![](http://img-ys011.didistatic.com/static/anything/gc.png)

GC流程如下：

1. 从Over-Provisoning空间中，找到一个空闲的Block-T。
2. 把Block-0的ABCDEFH和Block-1的A复制到空闲Block-T。
3. 擦除Block-0。
4. 把Block-1的BCDEFH复制到Block-0，此时Block0就有两个空闲Page。
5. 擦除BLock-1。

SSD的GC机制会带来两个问题：

* SSD的寿命减少。NAND-Flash中每个原件都有擦写次数限制，超过一定擦写次数后，就只能读取不能写入了。
* 写放大(Write Amplification)。即内部真正写入的数据量大于用户请求写入的数据量。

如果频繁的在某些Block上做GC，会使得这些元件比其他部分更快到达擦写次数限制。因此，需要损耗均衡控制(Wear-Leveling)算法，使得原件的擦写次数比较平均，进而延长SSD的寿命。

### <a name="chapter6"></a>Trim机制

`Trim`指令也叫`Disable Delete Notify`(禁用删除通知)，是微软联合各大SSD厂商所开发的一项技术，属于ATA8-ACS规范的技术指令。

**Trim(Discard)的出现主要是为了提高GC的效率以及减少写入放大的发生，最大作用是清空待删除的无效数据**。在SSD执行读、擦、写步骤的时候，预先把擦除的步骤先做了，这样才能发挥出SSD的性能，通常SSD掉速很大一部分原因就是待删除的无效数据太多，每次写入的时候主控都要先做清空处理，所以性能受到了限制。

在文件系统上删除某个文件时候，简单的在逻辑数据表内把存储要删除的数据的位置标记为可用而已，而并不是真正将磁盘上的数据给删除掉。使用机械硬盘的系统根本就不需要向存储设备发送任何有关文件删除的消息，系统可以随时把新数据直接覆盖到无用的数据上。固态硬盘只有当系统准备把新数据要写入那个位置的时候，固态硬盘才意识到原来这写数据已经被删除。而如果在这之前，SSD执行了GC操作，那么GC会把这些实际上已经删除了的数据还当作是有效数据进行迁移写入到其他的Block中，这是没有必要的。

在没有Trim的情况下，SSD无法事先知道那些被‘删除’的数据页已经是‘无效’的，必须到系统要求在相同的地方写入数据时才知道那些数据可以被擦除，这样就无法在最适当的时机做出最好的优化，既影响GC的效率(间接影响性能)，又影响SSD的寿命。

Trim和Discard的支持，不仅仅要SSD实现这个功能，而是整个数据链路中涉及到的文件系统、RAID控制卡以及SSD都需要实现。要使用这个功能必须要在mount文件系统时，加上discard选项。如果自己管理SSD裸设备就需要通过`ioctl`函数`BLKDISCARD`命令来操作了。

```
int block_device_discard(int fd, int64_t offset, int64_t len)
{
	uint64_t range[2] = {(uint64_t)offset, (uint64_t)len};
  	return ioctl(fd, BLKDISCARD, range);
}
```

### <a name="chapter7"></a>Bit-Error

在分析Bit-Error之前，我们先回顾一下`闪存基础`章节的知识。Bit-Error是磁盘的一种静默错误。造成Nand-Error的因素有：

* 电荷泄漏：长期不使用，会发生电荷泄漏，导致电压分布往左移，例如00漂移到01,10漂移到11。
* 读干扰(Read-Disturb)：读干扰小节会详细介绍。
* 写干扰(Program-Disturb)：写干扰小节会详细介绍。

不同因素造成的错误类型也不同： 

* Erase-Error：erase操作未能将cell复位到erase状态时，称为erase error。可能是制造问题，或者多次P/E引起的栅极氧化层缺陷所致。
* Program-Interference-Error：由Program-Disturb所导致的错误，会使电压分布偏移。
* Retention-Error：由电荷泄露引发的错误，会使电压分布偏移。
* Read-Error：由Read-Disturb所导致的错误，会使电压分布偏移。

retention时间越长，flash的浮栅极泄露的电子会越多，因而误码率越高，所以NAND-Error机制主要是为了减少Retention-Error。

### <a name="chapter8"></a>Read-Disturb

读取NAND的某个Page时，Block当中未被选取的Page控制极都会加一个正电压，以保证未被选中的MOS管是导通的。这样频繁的在一个MOS管控制极加正电压，就可能导致电子被吸进浮栅极，形成轻微的Program，导致分布电压右移，产生Bit-Error。注意Read-Disturb只影响同一Block中的其他Page。
![](http://img-ys011.didistatic.com/static/anything/read-disturb.png)
    
### <a name="chapter9"></a>Program-Disturb

erase之后所有bit都为1，写1不需要program，写0才需要program。如下图所示，绿色的Cell是写0，它们需要Program，红色的cell写1，并不需要Program。我们把绿色的Cell称为Programmed Cells，红色的Cell称为Stressed Cells。写某个Page的时候，会在其WordLine的控制极加一个正电压(下图是20V),对于Programmed Cells所在的String，它是接地的，对于不需要Program Cell所在的String，则接一正电压（下图为10V）。这样最终产生的后果是，Stressed Cell也会被轻微Program。与Read Disturb不同的是，Program Disturb 不但影响同一个Block当中的其它Page，自身Page也受影响。相同的是，都是不期望的轻微Program导致比特翻转，都非永久性损伤，经擦除后，Block还能再次使用。
![](http://img-ys011.didistatic.com/static/anything/write-disturb.png)

### <a name="chapter10"></a>Wear-Leveling

可参考：[经典Dual-pool 算法-高效Wear Leveling](http://www.ssdfans.com/blog/2018/12/30/%e7%bb%8f%e5%85%b8dual-pool-%e7%ae%97%e6%b3%95-%e9%ab%98%e6%95%88wear-leveling/)

### <a name="chapter11"></a>IO抖动因素

* **GC机制**：会导致内部IO，从而抢占用户的IO，导致性能抖动，甚至下降。
* **Bit-Error**：对于读操作，如果Bit-Error控制在一定范围内，那么延迟可以控制在100us内。如果超过了BCH快速解码的范围，将花费大量时间解码，延迟将增加。
* **读写冲突**：当一个读请求和写请求落在了同一个Block或者Page，会导致读延迟增加。针对这个问题，在存储系统设计过程中，需要将读写请求在空间上进行分离，从而避免读写请求在同一个Block上冲突。
* **读擦冲突**：当一个读请求和擦除请求落在了同一个Block，那么读请求将会被擦除请求block，NAND-Flash的擦除操作基本上在2ms以上，导致读延迟增加。为了解决这个问题，有些NAND-FLash也引入了Erase-Suspend的功能，让读优先于擦除操作，从而降低延迟。

### 参考文献

* [你的SSD可以用100年，你造吗?](http://www.ssdfans.com/blog/2017/08/03/%e4%bd%a0%e7%9a%84ssd%e5%8f%af%e4%bb%a5%e7%94%a8100%e5%b9%b4%ef%bc%8c%e4%bd%a0%e9%80%a0%e5%90%97%ef%bc%9f/)
* [SSD背后的秘密：SSD基本工作原理](http://www.ssdfans.com/blog/2017/08/03/ssd%e8%83%8c%e5%90%8e%e7%9a%84%e7%a7%98%e5%af%86%ef%bc%9assd%e5%9f%ba%e6%9c%ac%e5%b7%a5%e4%bd%9c%e5%8e%9f%e7%90%86/)
* [程序员需要知道的SSD基本原理](http://oserror.com/backend/ssd-principle/)
* [SSD内部的IO抖动因素](https://blog.51cto.com/alanwu/1876910)
* [闪存问题之Read Disturb](https://www.ssdfans.com/blog/2016/04/07/%E9%97%AA%E5%AD%98%E9%97%AE%E9%A2%98%E4%B9%8Bread-disturb/)
* [SSD Trim 详解](http://www.ssdfans.com/blog/2016/07/10/ssd-trim-%E8%AF%A6%E8%A7%A3/)
* [NAND ERROR机制解析](https://www.ssdfans.com/blog/2017/10/01/nand-error%e6%9c%ba%e5%88%b6%e8%a7%a3%e6%9e%90/)

转载请注明：[史明亚的博客](https://shimingyah.github.io) » [浅谈分布式存储之SSD基本原理](https://shimingyah.github.io/2019/07/%E6%B5%85%E8%B0%88%E5%88%86%E5%B8%83%E5%BC%8F%E5%AD%98%E5%82%A8%E4%B9%8BSSD%E5%9F%BA%E6%9C%AC%E5%8E%9F%E7%90%86/)