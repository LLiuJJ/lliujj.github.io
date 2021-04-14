
title: CMU-DBMS-COURSE-03-NOTES (Database Storage I)
date: 2019-04-06 17:18:53
tags: Database Storage I

**存储器架构**

![StorageHierarchy](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/StorageHierarchy.png)

- 存储器根据其存储特性可以分为多层，上层存储器容量小，运行速度快。
- 寄存器，CPU缓存，DRAM是易失性存储，一旦断电，存储数据将丢失。
- SSD，HDD等是非易失性存储，断电数据仍然保留。
- Non-Volatile Memory(NVM) 是位于分界线上的一种存储，它有着比SSD更高的性能，并能提供持久化存储。

**访问这些存储器的时延**

![ACCESSTIMES](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/ACCESSTIMES.png)

**数据库设计的目标**

- 允许DBMS去管理超过可用内存大小的存储量。
- 读写磁盘的开销是很大的，所以必须要谨慎管理磁盘，避免出现性能下降。

**顺序和随机访问的区别**

- 随机访问HDD要比顺序访问慢很多。
- 传统的数据库管理系统都是以最大化顺序访问为准则的。
- 算法尝试减少写入随机页面的次数，将数据存在连续的块中。

**mmap**

- 可以使用mmap映射一个文件的内容到一个进程的地址空间
- 操作系统的责任是将文件的页换入换出内存

![mmap1](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/mmap1.png)

![mmap2](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/mmap2.png)

**为什么不使用操作系统的调用管理文件？**

![mmap3](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/mmap3.png)

![mmap4](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/mmap4.png)

![mmap5](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/mmap5.png)

- 如果我们允许多个线程访问mmap文件去隐藏缺页错误呢？
- 这对只读的请求是正常工作的，但是当面对多个写请求的时候情况就复杂了。
- madvise：告诉操作系统，你期待怎么样去读某些页面？
- mlock：告诉操作系统一定范围内的内存不能被还出
- msync：告诉操作系统把内存的一定范围页面刷如磁盘

**DBMS做的一些事情**

- 以正确的顺序将脏页面写回磁盘
- 专业的预读
- 缓冲区的替换策略
- 线程/进程调度

**DBMS关注的两个问题**

- DBMS如何在磁盘文件中表示数据库？
- DBMS如何管理内存和磁盘之间来回的移动数据？

**文件存储**

- DBMS在一个或者多个磁盘文件中存储数据
- 存储管理器(storage manager)的责任是管理数据库文件，它将文件组织成页集合的形式，跟踪读写到页面的数据，跟踪可用的存储空间

**数据库页面**

- 一个页面是固定大小的数据块，它可以包括元组(tuples)，元数据(meta-data)，索引(indexes)，日志记录(log records)
- 大多数系统不回混合页面类型
- 有些系统需要一个独立的页面
- 每个页面都有一个唯一的标识符，DBMS使用一个间接层将页ID映射到实际的物理存储位置
- 对于数据库系统中的页，有三种不同的概念：硬件页(Hardware Page)通常是4KB；操作系统页(OS Page)通常是4KB；数据库页面(Database Page)1-16KB

**常见数据库页面大小**

![pages](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/pages.png)

**页面存储架构**

- 不同的DBMSs用不同的方式管理磁盘上面的页
  - 堆文件组织(Heap File Organization)
  - 顺序/排序文件组织(Sequential/Sorted File Organization)
  - 哈希文件组织(Hash File Organization)

**数据库Heap**

- 堆文件(Heap file)是无序的页面集合，其中是按随机顺序存储的元组。作用：获取/删除页面；必须支持对所有页面进行迭代。
- 需要元数据来跟踪存在的页面，以及哪些有空闲的空间
- Heap file的形式有两种: LinkedList和Page Directory

**LINKED LIST结构Heap file 特点**

![PageLinkList](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/PageLinkList.png)

- 在文件开头存储了两个指针：空闲页list的头指针；数据页list的头指针
- 每一页都跟踪它自己空闲slots的序号

![DirPages](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/DirPages.png)

- DBMS维护了一个特殊的页面来跟踪数据页的位置在数据库文件中
- 这个目录也记录了每个页面空闲slot序号
- DBMS确保目录页与数据页同步

**页面头部**

![page_header](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/page_header.png)

- 每个页面都包含一个头部，里面包含了页面的元数据信息
  - 页面大小
  - 校验和
  - DBMS的版本
  - 事务的可见性
  - 压缩信息
- 有些系统需要页面变得独立，比如Oracle

**页面的布局**

- 对于任何页面存储架构来说，我们现在需要知道怎样在页面里面组织数据存储
- 两种方法：
  - 面向元组
  - 日志结构

**元组的存储**

- 记录页面中远足的数量，然后往尾部增加新的元组

![tuple1](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/tuple1.png)

![tuple2](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/tuple2.png)

![tuple3](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/tuple3.png)

![tuple4](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/tuple4.png)

**页面槽(Slotted pages)**

- 最常见的页面部署方案称为槽页(slotted pages)
- slot数组映射了slots到元组开始位置的偏移量、
- header存储了使用了的slots，最后一个被使用slot的起始位置

![slot-page1](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/slot-page1.png)

![slot-page2](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/slot-page2.png)

**日志文件型数据组织方式**

![log1](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/log1.png)

- DBMS只存储日志记录，而不是将元组存储在页面中
- 系统将数据库如何修改的日志追加到文件
  - 插入存储整个元组
  - 将元组标记为已删除
  - 更新属性包含被修改的增量

![log2](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/log2.png)

- 要读取记录，DBMS反向记录并重新创建元组，并找到它需要的东西
- 建立索引允许跳到log的指定位置
- 定期的压缩日志

![log3](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/log3.png)

**日志的压缩**

- 通过删除不必要的记录来将较大的日志文件合并成较小的文件
- compaction的过程

![compaction1](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/compaction1.png)

![compaction2](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/compaction2.png)

![compaction3](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/ompaction3.png)

![compaction4](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/compaction4.png)

![compaction5](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/compaction5.png)

![compaction6](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/compaction6.png)

![compaction7](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/compaction7.png)

**元组布局**

- 元组本质上是一个字节序列
- DBMS的工作就是解释这些字节属性类型和值

![tuple-header](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/tuple-header.png)

- 每个元组前面都有一个头包含有关它的元数据：可见性信息（用于并发控制）；为NULL值提供的BitMap
- 我们为什么不需要存储schema的元数据信息呢？
- 元组里面的数据按你创建表时候的命令组织
- 组织这些数据是软件工程师要做的事情

![tuple-data](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/tuple-data.png)

**非规范化元组数据(denormalized tuple data)**

- 可以在物理上取消规范化相关元组并存储它们在同一页中
  - 可以减少常见工作负载的I/O请求数量
  - 可以使得update操作更高效

![denormalized1](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/denormalized1.png)

![denormalized2](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/denormalized2.png)

![denormalized3](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/denormalized3.png)

**记录的ID**

- DBMS需要一种跟踪单个元组的方法
- 每个元组都被分配一个唯一的记录标识符
  - 最常见：page_id + offset/slot
  - 也可以包含文件的位置信息

![record-ids](https://learnbycoding.oss-cn-beijing.aliyuncs.com/CMU-DBMS-COURSE-03-NOTES/record-ids.png)

