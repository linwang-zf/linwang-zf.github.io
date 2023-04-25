---
title: MySQL日志
author: linWang
date: 2022-05-017 20:11:28
tags: MySQL日志
categories:	MySQL系列
---

> MySQL日志是MySQL的核心功能之一，Crash-Safe、数据备份 & 恢复、数据一致性等能力都是依赖日志完成的，今天就来介绍下MySQL的日志系统。

<!--more-->

## 日志类型

MySQL日志有错误日志、查询日志、慢查询日志、事务日志、二进制日志等几类，当然最为重要的日志还是二进制日志binlog（归档日志）、事务日志redo log（重做日志），undo log（回滚日志）。

其中binlog是MySQL自带的日志文件，是运行在Server层，而redo log日志是InnoDB存储引擎特有的日志。

![image-20230422172837788](image-20230422172837788.png)

### binlog日志

binlog是逻辑日志，记录了SQL语句的原始逻辑，类似与“某个字段的值加一”，它属于MySQL Server层，所以每个存储引擎只要涉及到数据的变化，都会记录到binlog日志中。

#### 日志格式

binlog日志有三种格式，通过binlog_format参数来指定：

* statement
* row
* mixed

在statement模式下，binlog日志会将SQL语句的原文记录下来，如下图：

![image-20230422174305732](image-20230422174305732.png)

当备库要从主库同步数据时，只要应用binlog日志即可。但是这里有个问题，time = now()会获取系统当前时间，这样操作会导致备库和主库的数据不一致。

为了解决这个问题，可以将binlog格式指定为row，这时记录的内容不再是原始的SQL值，还包含了具体的值，记录内容如下：

![image-20230422174822766](image-20230422174822766.png)

可以看到，row格式下，会记录当前的时间，并且把表所操作的数据的原始值也记录了下来（图中的@1、@2），这样就保证了主备数据的一致性。

但是这种模式需要更多的存储空间来记录，比较占用空间，恢复与同步时会更消耗`IO`资源，影响执行速度。

所以就有了mixed模式，MySQL会根据**SQL语句是否可能引起数据的不一致**来判断采用哪种格式存储语句。

#### 写入进制

上面了解了binlog的日志格式后，我们再来看下binlog日志写入磁盘的机制

这里我们先介绍下为啥MySQL不直接把数据记录到磁盘中，而是要先写到日志文件中（WAL技术：write-ahead-logging），这是因为磁盘写是随机写，如果每次数据更新都直接更新到磁盘，那么IO成本、查找成本都是很高的，而日志是顺序写，速度会很快。

binlog的写入进制如下：

![image-20230422183205076](image-20230422183205076.png)

binlog日志的写入步骤如下：

（1）在事务执行过程中，将SQL语句记录到binlog cache中（因为事务的binlog不能分开，一个事务不管多大，binlog需要确保一次性写入，所以系统给每个线程分配一个binlog cache缓存）

（2）当事务提交时，将binlog cache的内容write到文件系统缓存page cache中

（3）最终调用fsync刷入磁盘中

binlog_cache_size可以控制binlog cache的大小，如果一个事务执行过程中存储的内容超过了这个值，需要暂存到磁盘中。

write 和 fsync的时机可以通过sync_binlog来控制：

* 当sync_binlog = 0时，每次事务提交只进行write，fsync由系统决定何时执行（这种方式下，如果MySQL突然宕机，可能造成page cache中的数据丢失）
* 当sync_binlog = 1时，每次事务提交先write，然后立马fsync（这种方式不会造成数据丢失）
* 当sync_binlog = N时，每次事务提交先write，累积N个事务后，执行一次fsync（同第一种方式一样，MySQL宕机时，可能造成N个事务数据的丢失）

### redo log日志

redo log日志是InnoDB存储引擎特有的能力，正是因为redo log，才让MySQL具备了崩溃恢复的能力（Crash-safe），保证了数据的完整性。

#### 日志格式

redo log在磁盘中不止一个文件，而是几个文件共同组成一个日志文件，每个文件的大小均相同。

并且redo log日志采用循环写的方式，类似与循环数组，它有两个很重要的参数：

* write_pos：当前记录的位置
* checkpoint：当前要擦除的位置

[write_pos, checkpoint)之间的位置就是可以用来写新的redo log的记录。而[checkpoint, write_pos)之间的位置则是需要往表中刷数据的内容。

如果write_pos追上了checkpoint，那么此时必须暂停手中的工作，清空一些redo log中的记录，让checkpoint往前走一段。

其实redo log和binlog一样，还有一个作用就是降低对磁盘IO的读写次数，将磁盘的随机写改为redo log的顺序写，大大提高了数据更新的性能。

#### 写入进制

![image-20230422193916202](image-20230422193916202.png)

MySQL为了降低磁盘IO的读写次数，会把涉及到的数据也加载到内存中，每次更新的时候直接修改缓存中的数据（buffer pool），同时在redo log cache中记录本次修改的内容。接着刷盘到磁盘中。

我们总结下redo log日志的写入步骤：

（1）数据更新时，将数据写入缓存buffer pool中

（2）将本次更新的记录写入redo log buffer

（3）调用write，将redo log buffer中的内容写入page cache中

（4）调用fsync，将文件系统缓存中的内容写入redo log文件中

除了上面的步骤外，InnoDB还有一个后台线程，每隔1s也会将redo log buffer的内容write到page cache中，并调用fsync（当redo log buffer的大小超过innodb_log_buffer_size的一半时，后台线程会主动刷盘）

可以write和fsync的刷盘时机可以通过innodb_flush_log_at_trx_commit参数控制：

* innodb_flush_log_at_trx_commit = 0时，每次事务提交仅将数据记录到redo log buffer后就结束了，剩下的交给后台线程完成（这种方式下，如果MySQL挂了或者宕机，都可能会丢失1s的数据）
* innodb_flush_log_at_trx_commit = 1时，每次事务提交会调用write和fsync，将数据直接刷到redo log文件中（这种方式下事务只要提交成功了，就不会出现数据丢失的情况，如果事务执行过程中MySQL宕机了，那么日志丢失也不会有任何损失）
* innodb_flush_log_at_trx_commit = 2时，每次事务提交会调用write，将数据保存到page cache中，等待后台线程调用fsync刷入redo log日志（这种方式下，如果MySQL挂了，不会造成任何数据丢失，但是如果宕机，可能会丢失1s的数据）

### undo log日志

我们知道在执行一个事务的过程中，如果发生了异常，那么该事务就会回滚，这个回滚就是通过undo log来实现的。

事务在执行的过程中，所有的修改会先记录到undo log中，然后在执行相关的操作，如果执行过程中发生了异常，则直接利用undo log日志中的信息回滚数据。并且undo log会先于数据持久化到磁盘，这样就保证了即使遇到数据库突然宕机的情况，当再次启动数据库后，依然可以通过查询undo log日志来回滚之前的事务操作。

## binlog和redo log对比

* binlog是逻辑日志，redo log是物理日志
* binlog是顺序写，redo log是循环写
* binlog是归档日志，用来实现数据恢复，主从、主备的数据一致性，redo log是事务重做日志，用来实现crash-safe（崩溃恢复）（binlog保证数据的一致性，redo log保证数据的完整性）

## 两阶段提交

binlog保证了MySQL集群架构中数据的一致性。

redo log给MySQL提供了崩溃恢复的能力。

那么在执行更新语句的过程中，这两个日志又是如何记录的呢，换句话说如何存储才能保证不会出现数据的不一致？这就不得不提到两阶段提交了。

对于两个日志的记录，我们可以总结下面几种方式：

* 先记录redo log，然后记录binlog
* 先记录binlog，然后记录redo log
* 先记录redo log，标记prepare，然后记录binlog，最后标记redo log为commit

上面三种方式，我们先看第一种：

假如redo log记录成功后，binlog还未记录，此时MySQL宕机了：

> 我们以具体的例子举例：假设id=1的记录中t=2，将其更新为3
>
> SQL语句为update T set t = 3 where id = 1;

假如redo log成功写入后，binlog日志写入时，发生了异常

* MySQL恢复后，主库根据redo log将t设置为3
* MySQL集群中的从库同步主库的binlog，由于binlog中没有这条更新语句，所以从库的t还是2，此时就造成了数据的不一致

第二种方式binlog成功写入，redo log写入异常和上述的原理一样，也会出现数据的不一致

我们来看第三种方式：

redo log写完后，标记为prepare，然后写入binlog，最后将redo log标记为commit

当MySQL宕机时，MySQL根据redo log的标识来确定下一步的操作：

* 如果redo log为commit，表示redo log和binlog全部写入成功，则将数据更新
* 如果redo log为prepare，然后根据事务Id去binlog中查看是否该事务已成功写入binlog，如果成功写入，则将redo log标记为commit，并将数据更新；否则回滚该事务

#### 更新语句的操作流程（两阶段提交）

![image-20230422203014744](image-20230422203014744.png)
