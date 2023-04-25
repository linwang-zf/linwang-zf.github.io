---
title: MySQL主备一致性原理
author: linWang
date: 2022-05-21 17:06:44
tags: MySQL主备一致性原理
categories: MySQL系列
---

>   MySQL能够成为现下最流行的开源数据库，binlog功不可没，不仅可以用来归档，还可以用来做主备之间的同步。下面我们就来学习下binlog是如何做到的。

<!--more-->

## MySQL主备架构图

![image-20221010091411206](image-20221010091411206.png)

状态1中，MySQL A为主库，客户端的所有读写都直接访问该数据库，MySQL B为备库，只负责同步A的更新到本地。在状态2中，MySQL B为主库，A变成了从库。

## MySQL主备同步流程示意图

![image-20221010202839417](image-20221010202839417.png)

从图中可以看到，当客户端发起一个update或insert语句后，MySQL A首先会进行两阶段提交，写redo log和binlog。

在A和B之间有一个长连接，A中的dump线程专门通过这个连接和B进行通信，传递事务执行成功后的binlog日志。详细的步骤如下：

1.   备库B发送binlog文件名+文件的偏移地址来获取需要同步的binlog内容（此处还会包含A的ip、端口、用户名以及密码等供A来校验）
2.   主库A校验成功后会将本地binlog日志的指定内容通过长连接发送给备库B的IO线程
3.   备库B拿到binlog后写到本地的relay log中，称为中继日志
4.   备库B的SQL线程从relay log中读取内容解析为SQL命令后执行

此时，我们已经知道了MySQL主备同步的原理，通过在备库执行主库传递过来的binlog即可，那么我们就一起来探究下binlog的具体内容吧。

## binlog文件格式

binlog一共有三种存储格式，分别为statement、row、mixed，其中mixed就是statement和row两种格式的混合。

我们通过具体的例子来讲解binlog的文件格式，首先我们先创建一个表并初始化几条数据。

```sql
create table binlog_study (
  id int(11) not null,
  name varchar(32) not null,
  age tinyint not null,
  primary key(id),
  key name(name),
  key age(age)
) engine=InnoDB;

insert into binlog_study values(1,"zhangsan",20);
insert into binlog_study values(2,"lisi",21);
insert into binlog_study values(3,"wangwu",22);
insert into binlog_study values(4,"zhaoliu",22);
insert into binlog_study values(5,"chenqi",23);
```

### statement

首先设置我们MySQL的binlog格式为statement，如下

```sql
set binlog_format=STATEMENT;

# 查看设置是否成功
show variables like 'binlog_format%';
```

假设我们此时要执行一个delete语句：

```sql
delete from binlog_study where id >=3 and age < 23 limit 1;
```

执行之后我们可以用下面的命令查看binlog日志的内容：

```sql
show binlog events in 'mysql-bin.000001';
```

![image-20221010205827882](image-20221010205827882.png)

我们一起看下上述binlog的日志内容：

1.   begin 和commit表示事务的开始和结束
2.   第三行为真正的执行语句，首先是use `mysql_study`，这个是mysql根据我们的语句自动加上去的，目的是为了不管当前线程在哪个库执行，都可以保证正确的执行mysql_study下的binlog_study表；use `mysql_study`后面跟着的就是我们刚刚执行的语句了，可以发现binlog直接将我们的SQL命令原封不动的保存了下来。

其实上述语句采用statement格式存储是有点问题的，这是因为delete语句带limit可能导致主备数据不一致。具体原因如下：

1.   上述的语句如果通过id索引来定义，那么会删除id=3这一行数据
2.   如果通过age来定位， 那么会删除id=4这一行数据

这样就会导致虽然是同一个语句，但由于选择的索引不一致而出现不同的执行结果，进而可能出现主备数据不一致的情况。

下面我们看下binlog_format=ROW的情况。

### row

首先我们还是先将binlog的格式设置为row，命令如下：

```sql
set binlog_format=ROW;

# 查看设置是否成功
show variables like 'binlog_format%';
```

我们同样执行上述的delete语句，然后查看binlog中的内容如下：

![image-20221010212214939](image-20221010212214939.png)

从图中可以看到， 与statement格式的binlog相比，row格式的binlog中没有了SQL原文，而是换成了Table_map和Delete_rows两条语句。

1.   Table_map：表示接下来要操作的表是mysql_study.binlog_study
2.   Delete_rows：定义了删除的行为

我们借助mysqlbinlog来查看binlog中的详细信息，如下图所示：

![image-20221010212550492](image-20221010212550492.png)

我们可以发现，row格式的binlog中并没有存储SQL的原文，而是直接保存了删除数据的主键ID，当binlog传入备库执行时，就只会删除主键Id=3的行了，不会在出现主备数据不一致的情况了。

### mixed

我们前面说过mixed是statement和row格式的混合，那么这个格式有什么用呢？

从前面我们的分析中我们可以知道statement和row的优缺点：

1.   statement占用空间小，但是可能会造成主备数据不一致
2.   row格式需要存储删除的数据内容，不仅占用空间大，而且耗费IO资源，但是不存在主备数据不一致的情况

所以MySQL为了平衡两种格式的优缺点，取了一个折中的办法，就是mixed的格式。在该格式下，MySQL会自己判读该语句该采用何种格式来存储binlog。如果语句不会引起主备数据不一致就采用statemen 就采用row格式。

## 主备同步存在的问题

### 循环复制问题

在实际场景中，可能存在A和B为双M结果，即A和B都互为对方的备库，此时就存在一个问题：

当业务在A上更新了一条语句，然后A把生成的binlog传递给B做同步，B做完同步之后生成了自己的binlog；由于A此时也是B的备库，A需要同步B的binlog来保持数据一致，那么节点A和B之间会不断的执行这个更新语句，也就是循环复制。

解决的办法也很简单，大家应该也发现binlog中存在server id这个参数，图中为1918，这个数字可以唯一对应一个数据库（如果两个库server id一样，是无法设定为主备关系的）。对于A和B互为主备的这种架构，备库在更新binlog时，不要覆盖主库的server id即可。此时，主备同步的流程如下：

1.   A将更新语句记录在binlog中，并记录自己库的server id
2.   B将A传递过来的binlog同步到自己库中，并生成自己的binlog，并记录A的server id
3.   A在同步B的binlog时，发现该事务的server id与自己的相同，则直接返回。

## 参考文档

>   MySQL实战45讲-----第24讲 MySQL是怎么保持主备一致的?
