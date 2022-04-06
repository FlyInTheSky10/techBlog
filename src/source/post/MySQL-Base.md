---
title: MySQL实战(一): 基础
date: 2022-04-01 21:38
categories:
- 数据库
tags:
- InnoDB
- MySQL
- 技术
---

本文基本上为 [MySQL 实战 45 讲](https://time.geekbang.org/column/intro/100020801) 1-15 节内容，为学习笔记，主要涉及 MySQL 基础部分内容如锁、事务、索引等。

# 架构

![MySQL 架构](../static/images/mySQL1.png)

MySQL 分为 **Server 层**和**储存引擎层**。

- Server 层：包括 MySQL 大多数核心服务功能，不同引擎共用一个 Server 层。
- 储存引擎层：数据的存储和读取，如 InnoDB、MylSAM、Memory 引擎。

## 连接器

连接器负责与客户端建立连接、获取权限、维持和管理连接。

如果管理员修改了权限表中某个用户的权限，那么即使这个用户当前存在成功的连接，也不会被影响，只有新建的连接才会受到影响。

可以使用 `show processlist` 命令看到当前连接，连接不作操作则为空闲状态 (Sleep)。

参数 `wait_timeout` 控制空闲多长时间将连接断开，默认为 8 小时。

> 长连接过多影响 MySQL 内存性能怎么办？

1、定期断开长连接

2、可以在每次执行一个比较大的操作后，通过执行 `mysql_reset_connection` 来重新初始化连接资源。这个过程不需要重连和重新做权限验证， 但是会将连接恢复到刚刚创建完时的状态。<!-- more -->

## 查询缓存

MySQL select 的查询缓存。

**大多数情况下不使用查询缓存。**查询缓存只要有对一个表的更新，都会清理这个表的所有查询缓存。除非是使用一张静态表。

将参数`query_cache_type`设置成 `DEMAND` ，可以默认不使用查询缓存。可在每条 select 语句用 `SQL_CACHE` 显示地指定使用查询缓存，如 

```sql
mysql> select SQL_CACHE * from T where ID=10;
```

## 分析器

分析器对 MySQL 语句进行**语法分析**。语法错误将在此抛出错误。

## 优化器

在执行前，要进行优化，即在表中有多个索引时，决定使用哪个**索引**。

## 执行器

开始执行任务，先检查用户权限再进行。

数据库的慢查询日志中有一个`rows_examined`的字段，表示这个语句执行过程中扫描了多少行。这个值就是在执行器每次调用引擎获取数据行的时候累加的。

# 日志

## redo log 与 binlog

redo log 即重做日志，binlog 即归档日志。

**redo log 记录这个页 "做了什么修改" (MySQL 中数据是以页为单位)，bin log 可以记录 SQL 语句，也可能是记录修改前后行数据。**

### redo log

InnoDB 的 redo log 是固定大小的，是一个循环结构。

![redo log 循环结构](../static/images/mySQL2.png)

- write pos: 当前记录的位置，是循环写的。
- checkpoint: 当前需要擦除的位置，擦除后将记录更新写入磁盘。
- write pos 与 checkpoint 之间为可记录新操作的位置，如果 write pos 到了 checkpoint 的位置，则 redo log 满了，需要等待擦除后 checkpoint 后移释放位置。

**crash-safe**：redo log 使得 MySQL 即使在数据库异常重启后依然能保证提交的记录不丢失。

`innodb_flush_log_at_trx_commit`这个参数设置成1的时候， 表示每次事务的 redo log 都直接持久化到磁盘。**建议设置为1。**

redo log 一般设置为 4 个文件、每个文件 1GB

#### 刷脏页

当内存数据页和硬盘数据页数据不一致时，称内存中的数据页为**脏页 (Dirty Page)**。

> **刷脏页 (flush)** 的时机？

1、redo log 满了，停止所有更新操作，checkpoint 往前推，走过区域 redo log 的脏页 flush到磁盘，留出空间写新的 redo log

2、系统内存不足。需要新的数据页而内存不足，则应该将旧的数据页淘汰。

3、空闲时期。

4、MySQL 正常关闭时。

正确设置 `innodb_io_capacity` 参数 (磁盘 I/O 能力指标，(建议设置成磁盘的IOPS。磁盘的IOPS可以通过 fio 工具来测试)) 可以使得不会刷脏页的时候刷得特别慢。

InnoDB的刷盘速度参考两个因素：一个是脏页比例，一个是 redo log 写盘速度。

参数 `innodb_max_dirty_pages_pct` 是脏页比例上限，默认值是75%。**平时要多关注脏页比例，不要让它经常接近75%。**脏页比例是通过`Innodb_buffer_pool_pages_dirty`/`Innodb_buffer_pool_pages_total`得到的

`innodb_flush_neighbors` 参数控制 flush 脏页是否影响邻近页的 flush。**建议设置为0。**

脏页刷到内存实质上与 redo log 无关。redo log 不能直接去更新硬盘数据页。

### bin log

Server 层的日志即为 bin log。

使用两个 log 的原因：最开始 MySQL 里并没有 InnoDB 引擎。MySQL自带的引擎是 MyISAM，但是 MyISAM 没有 crash-safe 的能力，binlog 日志只能用于归档。

**redo log 为物理日志， bin log 为逻辑日志**。

bin log 不是循环结构，是追加写的。

`sync_binlog`这个参数设置成1的时候，表示每次事务的binlog都持久化到磁盘。**建议设置为1。**

### WAL 技术

WAL 技术即 Write-Ahead Logging，即先写日志，再写磁盘。(磁盘 I/O 效率极低，要减少磁盘 I/O 次数)

当一条记录被更新后，MySQL 先将其写入 redo log，等到一定时间或者空闲时再将其一起一次写入磁盘。

### update 的两阶段提交

![update 的两阶段提交](../static/images/mySQL3.png)

由图可见，最后三部将 redo log 写入拆成两个步骤，**prepare 和 commit，即两阶段提交。**

两阶段提交可以保证两份日志同步，逻辑一致，即**一致性**。可用反证证明不拆分的两种情况都无法保证。

**只保证一致性，不保证这一小段时间 redo log 还在 redo log buffer 里没有被刷到磁盘上的数据丢失。**

> MySQL怎么知道 bin log 是完整的?

一个事务的 bin log 是有完整格式的： 

- statement 格式的 bin log，最后会有 COMMIT
- row 格式的 bin log，最后会有一个 XID event

# 锁

## 全局锁

全局锁即对整个数据库加锁。

MySQL 提供了一个加全局读锁的方法，命令是 `Flush tables with read lock (FTWRL)`。

可以用作全局备份，但是不建议这么做。

- 如果你在**主库**上备份，那么在备份期间都不能执行更新，业务基本上就得停摆； 
- 如果你在**从库**上备份，那么备份期间从库不能执行主库同步过来的binlog，会导致**主从延迟**。

**可以在可重复读隔离级别下开启一个事务备份。**

MyISAM 不支持事务，则其也不支持上述操作。

`set global readonly=true`的方式也可以让全库为只读模式。

> 为什么不建议使用 readonly？

1、readonly 在某些系统会用作其他逻辑，比如判断该库是否为备库

2、异常处理机制不同。如果异常端口，全局锁会直接释放，而 readonly 将会持续

## 表锁

### 表锁

表锁语法：`lock tables …read/write`，可主动使用 `unlock tables`释放锁，也可以在客户端断开后自动释放。

**InnoDB 一般不使用。**

### MDL 锁

MDL (MetaData Lock)，**不需要显示调用，会在访问一个表时自动加上。**

MDL 保证读写的正确性。即控制 DDL (改结构) 和 DML (查改数据) 的冲突。**此锁为读写锁。**

事务中的MDL锁，在语句执行开始时申请，但是语句结束后并不会马上释放，而会等到整个事务提交后再释放。

> 如何安全地给小表加字段？

一个 DDL 长期得不到 MDL 锁，则会阻塞后面的所有操作，导致session 爆满。

防止 session 爆满，占用内存空间。

1、解决长事务

在MySQL的 `information_schema` 库的 `innodb_trx` 表中，你可以查到当前执行中的事务。如果你要做 DDL 变更的表刚好有长事务在执行，要考虑先暂停 DDL，或者 kill 掉这个长事务。

2、在 `alter table` 语句里面设定等待时间

```sql
ALTER TABLE tbl_name NOWAIT add column ...
ALTER TABLE tbl_name WAIT N add column ...
```

## 行锁

在 InnoDB 中，行锁按需添加，但是要到事务结束才释放，这就是**两阶段锁协议**。

**所以要把加锁后影响并发度最大的锁尽量往后加锁。**

### 死锁

A 事务等待行 1 的锁，并且持有行 2 的锁，而 B 事务持有行 2 的锁，却等待着行 1 的锁，就造成了**死锁**。

> 如何处理行锁死锁？

1、直接进入等待

等待超时时间，通过参数 `innodb_lock_wait_timeout` 来设置，默认 50s

2、死锁检测

将参数`innodb_deadlock_detect`设置为`on`，表示开启这个逻辑。

可以通过这个检测锁等待是否成环，成环则将此事务终止，即可破环，类似拓扑排序。

3、对于热点行更新

由于死锁检测复杂度为 $O(n)$，则死锁判断会消耗大量 CPU 资源。

解决方法：

- 临时关闭死锁
- 限制并发量：1、对应相同行的更新，在进入引擎之前排队 2、将热更新的行数据拆分成逻辑上的多行来减少锁冲突

# 事务隔离

事务特性 ACID (Atomicity、Consistency、Isolation、Durability)，即原子性、一致性、隔离性、持久性。

- 原子性 (Atomicity): 一个事务中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节，而且事务在执行过程中发生错误，会被回滚到事务开始前的状态，就像这个事务从来没有执行过一样。

- 一致性 (Consistency): 数据库的完整性不会因为事务的执行而受到破坏。

- 隔离性 (Isolation)：数据库允许多个并发事务同时对其数据进行读写和修改的能力，隔离性可以防止多个事务并发执行时由于交叉执行而导致数据的不一致。

- 持久性 (Durability): 事务处理结束后，对数据的修改就是永久的，即便系统故障也不会丢失。

其中

- 持久性是通过 redo log （重做日志）来保证的；
- 原子性是通过 undo log（回滚日志） 来保证的；
- 隔离性是通过 MVCC（多版本并发控制） 或锁机制来保证的；
- 一致性则是通过持久性+原子性+隔离性来保证；

这里接下来讲其中的 I，即隔离性。

## 读

- 脏读 (dirty read)：如果一个事务**读到了另一个未提交事务修改过的数据**，就意味着发生了脏读。
- 不可重复读 (non-repeatable read)：在一个事务内多次读取同一个数据，如果出现**前后两次读到的数据不一样的情况**，就意味着发生了不可重复读现象。
- 幻读 (phantom read)：在一个事务内多次查询某个符合查询条件的记录数量，如果出现**前后两次查询到的记录数量不一样**的情况，就意味着发生了幻读现象。**可重复读出现幻读原因是可重复读除了 select 操作是视图读，其他操作都是当前读。**

## 隔离级别

- 读未提交 (read uncommitted): 一个事务还未提交，就能被其他事务看到变更。
- 读提交 (read committed): 一个事务只有提交了，其他事务才能看见他的变更。
- 可重复读 (repeatable read): 一个事务执行过程中看见的数据，与事务刚启动时的一致 (启动时的一个视图 (read-view) )，未提交的事务也是不可见的。
- 串行化 (serializable): 对于同一记录，写加写锁，读加读锁。

启动参数`transaction-isolation`的值设置成`READ-COMMITTED`可以设置为读提交隔离级别，可以用 `show variables` 来查看当前的值。

## 实现

**MySQL 每次更新数据的时候会记录一条回滚操作 (回滚日志，undo log)。**通过回滚操作能回到前几个状态的值。可用于建立不同的视图 (read-view)。

同一条记录在系统中中可能存在多个版本，即为**数据库的多版本并发控制 (MVCC)**。

undo log 在没有比这个回滚日志更早的 read-view 的时候删除。

**因此，尽量不用长事务。**

> 为什么不建议使用长事务？

1、保留大量 undo log，占用储存空间

长事务会导致系统中会存在很多老视图，使得 undo log 大量保留，占用储存空间。

undo log 是跟数据字典一起放在 ibdata 文件里的，即使长事务最终提交，回滚段被清理，文件也不会变小。

2、长事务占用锁资源

可以在`information_schema`库的`innodb_trx`这个表中查询长事务，如下 (查询大于 60s 的事务)

```sql
select * from information_schema.innodb_trx where TIME_TO_SEC(timediff(now(),trx_started))>60
```

## 事件启动方式

### commit 方式

- 显式启动事务语句， begin 或 start transaction。配套的提交语句是commit，回滚语句是 rollback。 
- `set autocommit=0`，这个命令会将这个线程的自动提交关掉。意味着如果你只执行一个 select 语句，这个事务就启动了，而且并不会自动提交。这个事务持续存在直到你主动执行 commit 或 rollback 语句，或者断开连接。

**建议使用 `set autocommit=1`, 通过显式语句的方式来启动事务。**

### 视图创建时机

`begin/start transaction` 启动事务在执行到它们之后的第一个操作 InnoDB 表的语句，事务才真正启动，**一致性视图**是在执行第一个快照读语句时创建的。

`start transaction with consistent snapshot` 启动事务在执行此条语句即创建**一致性视图**。

## 视图工作原理 (undo log)

InnoDB 里每个事务 **(只读事务不分配)** 都会分配一个事务 ID，即 transaction id，在事务开启前向系统申请得到，按申请顺序严格递增。

每行数据有多个版本。每次事务更新时，都将生成一个新的版本，把 transaction id 赋值给这个数据版本的事务 ID，记为 `row trx_id`。保存的链，即为 **undo log**。

InnoDB 为每个事务创建一个数组储存启动瞬间启动了没有提交的事务的 ID。

数组里面事务 ID 的最小值记为**低水位**，当前系统里面已经创建过的事务 ID 的最大值加 1 记为**高水位**。

这个视图数组和高水位，就组成了当前事务的**一致性视图（read-view）**，可重复读的核心就是一致性读。

这样创建视图即非常快速。

对于**可重复读**，除了自己的更新总是可见以外，有三种情况：

- 版本未提交，不可见；
- 版本已提交，但是是在视图创建后提交的，不可见；
- 版本已提交，而且是在视图创建前提交的，可见。

**对于可重复读，更新数据都是先读后写的，而这个读，只能读当前的值，称为当前读（current read）。**所以会产生幻读的情况。当前读需要加行锁。

除了 update 语句外，select 语句如果加锁，也是当前读。

> 为什么表结构不支持可重复读？

表结构没有对应的行数据，也没有 `row trx_id`，只能当前读。

# 索引

InnoDB 使用 B+ 树索引模型，B+ 树存储单位为**页**。每一张表包含多棵 B+ 树。

B+ 树高取决于叶子树（数据行数）和 N 叉树的 N 。 **而 N 是由页大小和索引大小决定的**。

索引分为主键索引和非主键索引。

- 主键索引 (又称聚簇索引，clustered index)：叶子结点存的是整行数据。
- 非主键索引 (又称二级索引，secondary index)：叶子结点存的是主键值。

**回表**指查询时使用二级索引查询后，再到主键索引树搜索一次找到数据的过程。

一般情况下表要有**自增主键**，并且**要求主键长度尽可能小。**

没有主键的表，会自动加上一个 Rowid 做主键。

## 覆盖索引

如果查找的刚好是主键，那么使用二级索引作为 where 条件时，不需要回表，这就是**覆盖索引**。覆盖索引减少对树的搜索，提高性能。

可以对高频请求单独创建一个**联合索引**。

## 最左前缀原则

不只是索引的全部定义，只要满足最左前缀，**由于 B+ 树的索引结构**，都可以利用索引来加速检索。

所以当已经有了 (a, b) 这个联合索引后，一般就不需要单独在 a 上建立索引了。

**第一原则是，如果通过调整顺序，可以少维护一个索引，那么这个顺序往往就是需要优先考虑采用的。**再者，空间问题也要考虑。

## 索引下推

MySQL 5.6 引入的**索引下推优化**（index condition pushdown)， 可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，**减少回表次数**。

## 索引选择

InnoDB 不能知道满足条件的语句有多少条，只能通过一个**基数 (cardinality) **来估计扫描行数。**基数即为一个索引上不同值的个数**，可以使用 `show index` 方法，看到一个索引的基数。

InnoDB 随机选择 N 个数据页，统计页面的不同值的平均数，然后得出这个索引的基数。

当变更的数据行数超过 1/M 的时候，会自动触发重新做一次索引统计。

`analyze table t` 命令，可以用来重新统计索引信息。

采用 `force index` 可以强行选择一个索引。

我们可以新建一个更合适的索引，来提供给优化器做选择，或删掉误用的索引。

## 字符串索引

**前缀索引**：可以定义一部分前缀作为索引。需要区分度高的前缀索引查找效率才高

前缀索引会对覆盖索引造成影响。即使前缀索引将整个字符串包括，查找都会回到主键回表一次，因为系统不知道是否包含了整个字符串。

**倒序索引**：字符串倒着存，再使用前缀索引来增加索引区分度。

**Hash索引：**在表上再创建一个整数字段，来保存身份证的校验码

## B+树空洞

`innodb_file_per_table` 设置为 ON 时，使用 `drop table` 系统就会直接删除这个文 件，否则不会。

数据删除只是打标签，会照成空洞，同理数据插入 (B+树页分裂) 和更新 (删除+插入)

可以使用 `alter table A engine=InnoDB` 命令来**重建表**。此 DDL 不是 Online 的。

重建表时 inplace 没有把数据挪动到临时表，是 一个原地操作，copy 则为强制拷贝表。

`analyze table t` 其实不是重建表，只是对表的索引信息做重新统计，没有修改数据，这个过程中加了 MDL 读锁。

# 缓存

## change buffer

需要更新数据页时，如果数据页当前不在内存中，不影响一致性的情况下，就将更新操作缓存在 **change buffer**，然后之后查询时，将数据页读入内存再通过 change buffer 来得到正确的数据页。

change buffer 是可持久化的。在**共享表空间 ibdata1**。

将 change buffer 应用到硬盘数据页的操作称为 **merge**，访问页会触发 merge，数据库也会定期 merge，数据库 shutdown 后也会 merge。

merge 流程：

- 从硬盘读入数据页
- 在内存中查找这个数据页的 change buffer，然后依次修改内存中的数据页，此时数据页为脏页
- 写 redo log，包含数据变更和 change buffer 的变更 

change buffer 减少了硬盘 I/O，同时减少了数据页读入内存，减少了内存占用。

change buffer 的大小，可以通过参数 `innodb_change_buffer_max_size` 来动态设置。这个参数设置为 50 的时候，表示 change buffer 的大小最多只能占用 buffer pool 的50%。

**尽量使用普通索引而不是唯一索引，因为唯一索引修改完马上会查询，这样 change buffer 的优化就失效了。**

## change buffer 和 redo log

change buffer 的修改会记录进 redo log

**redo log 主要节省的是随机写磁盘的IO消耗（转成顺序写），而change buffer主要节省的则是随机读磁盘的IO消耗。**

如果掉电，已经持久化的 change buffer 已经 merge，不用恢复。

对于没有持久化的 change buffer：使用 redo log 和 bin log 来恢复

## redo log buffer

redo log buffer 用来存 redo log
