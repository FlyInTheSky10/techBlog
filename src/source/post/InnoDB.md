---
title: InnoDB 深入理解
date: 2021-12-25 14:38
categories:
- 数据库
tags:
- InnoDB
---

# InnoDB 体系结构

InnoDB 支持独立表空间，支持MVCC，行锁设计，提供一致性非锁定读
支持外键，插入缓冲，二次写，自适应哈希索引，预读
使用聚集的方式存储数据，每张表的存储都是按主键顺序存放。

# 架构

![](../static/images/innoDB1.jpg)
<!-- more -->

https://zhuanlan.zhihu.com/p/158978012

## 内存池

主要工作：

- 维护所有进程/线程需要使用的多个内部数据结构
- 缓存磁盘上的数据，方便快速地读取，同时对磁盘文件数据修改之前在这里缓存
- 重做日志 (redo log) 缓存

![](../static/images/innoDB2.jpg)

### 缓冲池

InnoDB是基于磁盘存储的，并将其中的记录按照页的方式进行管理。
而缓冲池就是一块内存区域，主要缓冲数据页和索引页。
InnoDB中对页的读取操作，首先判断该页是否在缓冲池中，若在，直接读取该页，若不在则从磁盘读取页数据，并存放在缓冲池中。
对页的修改操作，首先修改在缓冲池中的页，再以一定的频率（**Checkpoint机制**）刷新到磁盘。

#### LRU

缓冲池通过**LRU（Latest Recent Used，最近最少使用）算法**进行管理。最频繁使用的页在LRU列表前端，最少使用的页在尾端，当缓冲池不能存放新读取的页时，首先释放LRU列表尾端的页（页数据刷新到磁盘，并从缓冲次中删除）。
InnoDB对于新读取的页，**不是放到LRU列表最前端，而是放到midpoint位置（默认为5/8处）**。
这是因为一些SQL操作会访问大量的页（如全表扫描），读取大量非热点数据，如果直接放到首部，可能导致真正的热点数据被移除。

在LRU列表中的页被修改后，称该页为**脏页 (dirty page)**，即缓冲池中的页和磁盘上的页的数据产生了不一致。

### 重做日志 (Redo Log) 缓冲

重做日志先放到这个缓冲区，然后按一定频率刷新到重做日志文件。

## Checkpoint 机制

InnoDB对于对于**DML语句操作**（如Update或Delete），事务提交时只需在缓冲池中中完成操作，然后再通过Checkpoint将修改后的脏页数据刷新到磁盘。

两种Checkpoint
Sharp Checkpoint：数据库关闭时将所有脏页刷新会磁盘
Fuzzy Checkpoint：

- Master Thread Checkpoint
  Master Thread每个1秒或10秒按一定比例将缓存池的脏页列表刷新会磁盘
- FLUSH LRU LIST Checkpoint
  Page Cleaner线程发现LRU列表中可用页数量少于innodb_lru_scan_depth(1024)，就将LRU列表尾端移除，如果这些页中有脏页，就需要Checkpoint
- Async/Sync Flush Checkpoint
  重做日志文件空间不可以用时，将一部分脏页刷新到磁盘。
- Dirty Page too much Checkpoint:
  脏页数量太多（超过比例innodb_max_dirty_pages_pct,默认75），执行Checkpoint。

## 线程

主要作用：
- 负责刷新内存池中的数据，保证缓冲池的内存缓冲的是**最近的数据**
- 已修改的数据文件**刷新到磁盘文件**
- 保证数据库**发生异常**的情况下InnoDB能恢复到正常状态。

### Master Thread
负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，包括脏页的刷新，合并插入缓冲（INSERT BUFFER），UNDO页的回收等。
Master Thread具有最高的线程优先级别，内部由多个循环组成：主循环（loop），后台循环(backgroup loop)，刷新循环(flush loop)，暂停循环(suspend loop)，Master Thread根据数据库运行状态在以上循环切换。

### IO Thread
负责AIO请求的回调处理。

### Purge Thread
事务提交后，undo log可能不再需要，由Purge Thread负责回收并重新分配的这些已经使用的undo页。
注意：Purge Thread需要离散地读取undo页。

### Page Cleaner Thread
InnoDB 1.2.x引入，将Master Threader中刷新脏页的工作移至该线程，如上面说的FLUSH LRU LIST Checkpoint以及Async/Sync Flush Checkpoint。

## 特性

### 插入缓冲 (Insert Buffer)

插入聚集**索引一般是顺序**的，不需要磁盘的随机读取
但插入非聚集索引叶子节点不是顺序的，需要离散访问非聚集索引页，速度较慢。
对于非聚集索引的插入或更新，先判断插入的非聚集索引页是否在缓存池中，若在，直接插入，或不在，先放到一个**Insert Buffer对象**中，
然后根据一些算法将**Insert Buffer缓存**的记录通过后台线程慢慢**合并刷新**回辅助索引。
插入缓冲将**多次插入合并为一次操作，减少磁盘的离散操作**。

使用Insert Buffer需满足两个条件：

- 索引是辅助索引
- 索引不是唯一的（不需要查找索引页判断唯一性）

InnoDB从1.0.x引入Change Buffer，对INSERT，DELETE，UPDATE都进行缓冲。

### Redo Log, Undo Log

**重做日志 (Redo Log)**：每次操作都先记录到Redo Log中，当出现实例故障**（像断电）**，导致数据未能更新到**硬盘数据文件**，则数据库重启时须redo，重新把数据更新到**硬盘数据文件**。

**撤消日志 (Undo Log)**：当一些更改在执行一半时，发生意外，而无法完成，则**可以根据撤消日志恢复到更改之前的状态**。记录所有的前印象，用于回滚。

做 undo 的目的是使系统恢复到系统崩溃前(关机前)的状态, 再进行 redo 是保证系统的一致性。不做 undo, 系统就不会知道之前的状态, redo 就无从谈起

https://blog.51cto.com/sunwayle/107232

#### Undo + Redo事务的简化过程

 假设有A、B两个数据，值分别为1,2，开始一个事务，事务的操作内容为：把1修改为3，2修改为4，那么实际的记录如下（简化）：
```
A.事务开始.
B.记录A=1到undo log.
C.修改A=3.
D.记录A=3到redo log.
E.记录B=2到undo log.
F.修改B=4.
G.记录B=4到redo log.
H.将redo log写入磁盘。
I.事务提交
```
https://www.cnblogs.com/xinysu/p/6555082.html#_lab2_2_0

### 两次写 (Double Write)

如果说**插入缓冲是为了提高写性能**的话，那么**两次写是为了提高可靠性**，牺牲了一点点写性能。

**两次写（Double Write）**是很独特的一个功能点。因为Innodb中的日志是**逻辑**的，所谓逻辑就是比如插入一条记录时，它可能会在某一个页面（这条记录最终被插入的位置）的多个**偏移位置**写入某个长度的值，例如页头的记录数、槽数、页尾槽数据、页中的记录值等。这些本是一些**物理操作**，而InnoDB为了节省日志量及其它原因，设计为逻辑处理的方式，即在一个页面上插入一条记录时，对应的日志内容包括表空间号、页面号、将被记录的各个列的值等内容，在真正物理插入的时候，才会**将日志逻辑操作转换为前面的物理操作**。

先有逻辑日志，再有物理操作，但是这样需要有一个前提，**就是物理操作的页面是正确的**。如果那个数据页面本身是错误的，这种错误可能是上次的操作导致的写断裂（1个页面为16KB，分多次写入，后面的可能没有写成功，导致这个页面不完整）或者其它原因，那么这个逻辑操作就没办法完成了。因为如果这个页面不正确的话，里面的数据是无效的，就可能会产生各种不可预料的问题。

因此首先要保证这个页面是正确的，方法就是**两次写**。

![](../static/images/innoDB3.jpg)

https://www.cnblogs.com/xuliuzai/p/10290196.html

### 自适应哈希索引

InnoDB会监控对表上各索引页的查询执行情况，如发现建立哈希索引可以提升速度，则建立哈希索引，这是过程不需要用户干预。

### 异步IO

InnoDB使用异步IO操作磁盘，避免同步IO导致阻塞，也可以进行IO Merge操作，将多个IO操作合并为一个IO操作。

### 刷新邻接页

当刷新一个脏页时，InnoDB会检测该页所在区的所有页，如果是脏页，一起刷新，这是可以通过AIO将多个IO写入操作合并为一个IO操作。

# 储存

