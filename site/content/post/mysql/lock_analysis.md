---
title: "MySQL各种情况的加锁分析"
date: 2020-10-27T02:46:48+08:00
draft: false
tags:
    - MySQL
---

首先介绍了事务与隔离级别，引入了加锁的复杂性以及InnoDB的锁类型。最后对于各种索引、数据的加锁情况进行了相应的分析。

# 加锁是为了数据库能够在并发下正确运行

MySQL是一个支持**事务**的数据库。如何处理多个事务之间的关系，就需要引入一个新的概念：**隔离级别**。

## 四个隔离级别

*注意，不同数据库对于事务的隔离级别实现略有不同。*

### READ UNCOMMITED （可能）读到未提交的事务

存在的问题：**脏读**

一个事务可能读取到另一个事务作出的、**未提交**的修改。

实际生产中，这种隔离级别通常没什么使用场景。

### READ COMMITTED 只能读到已经提交的事务

为了解决脏读，该隔离级别限制一个事务中的任意修改，在提交之前，都不会被其他事务读取到。

存在的问题：**不可重复读**

考虑一种情景：

1. 事务A首先读取数据x值记为x1
1. 事务B修改x为x2并提交
1. 此时事务A再次读取x获得x2与x1不同

这种情况称为**不可重复读**。

### REPEATABLE READ 可重复读 **MySQL的默认隔离级别**

通常可重复读解决了不可重复读问题。

存在的问题：**幻读**

类似不可重复读的形成原因，将特定的数据的查询，改成范围查询。

事务B创建一条新的记录，那么事务A第二次查询的时候，就会有一条新的记录被查到。这种现象成为**幻读**。

### SERIALIZABLE 串行化

以上的问题均为多个事务并发执行导致的。串行化隔离级别将事务串行执行，自然不会产生问题。

# 为了更高的性能，锁的粒度需要尽可能的小

数据的安全、准确与能承载的并发量是跷跷板的两端。

在能够满足业务对数据安全的情况下，为了提高性能，减少锁的粒度，是提高性能的有效方法。

# InnoDB的锁类型

行锁，意向锁(Intention Lock)，记录锁（Record Lock），间隙锁，Next-key锁，插入意向锁，自增数值锁。

## 行锁：共享锁与排他锁

InnoDB实现了标准的行锁。

其中包含了共享锁与排他锁。

## 意向锁

意向锁是用来兼容表锁与行锁而设计的。

意向锁是一个表锁。分为读写两种，IX和IS。

意向锁有两个约定：

1. 对于尝试申请行锁读锁的事务，必须先申请到IS锁。排他锁亦然。
1. 意向锁会阻塞读写表锁，如IX阻塞读表锁，不会阻塞读行锁。

不严谨的讲，持有一个意向锁，表明该事务持有“部分”该种表锁。

## 记录锁

记录锁是加在**索引的对应条目**上的。用于实现行锁，如避免其他事务修改对应条件的记录。

我认为记录锁就是行锁的实现。

## Gap Lock 

间隙锁是加在**索引的两个条目之间或到正/负无穷**上的锁。

用来避免其他事务对本事务修改的（指写操作）需要加锁的区间的插入。

```mysql
# id是主键
# transaction A
begin
select * from t1 where id >= 10 and id <= 20 for upate # 在id索引上会有10 - 20的索引
### 不要提交

# transaction B
insert into t1 values (`val`) (15) # 会被阻塞
```

**如果查询的条件没有在索引上呢？**

(RR下)会退化锁住全表的行。*这真的不是我试验哪里出问题了🏇？*

## Next-Key Lock

Next-Key锁是一个记录锁和锁定该记录前的间隙的间隙锁的组合。

在RR隔离级别下，InnoDB利用Next-Key锁避免**幻读**的产生。

## 插入意向锁

插入意向锁用来增加并发度。

插入意向锁是一种由`Insert`发起的间隙锁。

不同于间隙锁的排他，插入意向锁是一种意向排他锁。

对于两个事务，如果是因为`Insert`导致需要加间隙锁的时候，会尝试加一个插入意向锁。

相较于间隙锁不允许任何插入，插入意向锁之间，允许不同位置的插入。

## 自增锁

对于自增的列，可以通过配置来对严格顺序生成和并发性能之间进行权衡。

# 场景试验

## 0. 环境

以下试验如无特殊说明，均在RR隔离级别下，数据库版本5.7。

有的试验不得不改动表结构才能得出结论，所以有的试验数据不太符合。
表结构：
```mysql
CREATE TABLE `t1` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `val` bigint(20) NOT NULL DEFAULT '0',
  `str` varchar(128) NOT NULL DEFAULT '',
  `no_idx_val` int(11) NOT NULL DEFAULT '0',
  `uni_val` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uni_uni_val` (`uni_val`),
  KEY `idx_val` (`val`)
) ENGINE=InnoDB AUTO_INCREMENT=10 DEFAULT CHARSET=utf8;
```

## 1. 自增主键，并发插入

结论：不会阻塞，先执行的获得较小的自增值。

试验过程：

1. 分别开启两个事务先后执行

  ```mysql
  # transactionA
  insert into t1 (val, str, no_idx_val, uni_val) values (8, 'record8', 8, 8);
  # transactionB
  insert into t1 (val, str, no_idx_val, uni_val) values (10, 'record8', 10, 10);
  ```

1. 在提交之前的`show engine innodb status`

  ```
  ------------
  TRANSACTIONS
  ------------
  Trx id counter 2966
  Purge done for trx's n:o < 2963 undo n:o < 0 state: running but idle
  History list length 34
  LIST OF TRANSACTIONS FOR EACH SESSION:
  ---TRANSACTION 422034519421440, not started
  0 lock struct(s), heap size 1136, 0 row lock(s)
  ---TRANSACTION 422034519418704, not started
  0 lock struct(s), heap size 1136, 0 row lock(s)
  ---TRANSACTION 2965, ACTIVE 21 sec
  1 lock struct(s), heap size 1136, 0 row lock(s), undo log entries 1
  MySQL thread id 11, OS thread handle 140559053973248, query id 430 localhost root
  ---TRANSACTION 2963, ACTIVE 94 sec
  1 lock struct(s), heap size 1136, 0 row lock(s), undo log entries 1
  MySQL thread id 4, OS thread handle 140559054243584, query id 425 localhost root
  ```

1. 先提交后写入语句的事务，发现较小的id由先写入命令的事务获得。
  ```
  +----+--------+----------+------------+---------+
  | id | val    | str      | no_idx_val | uni_val |
  +----+--------+----------+------------+---------+
  ...
  | 10 |      8 | record8  |          8 |       8 |
  | 12 |     10 | record10 |         10 |      10 |
  +----+--------+----------+------------+---------+

  ```

## 2. 在非主键索引上获取排他锁，会发生什么？

1. 非主键非唯一索引相等搜索

    结论：

      1. 会为符合条件的记录在主键记录上行锁排他。
      1. 会为符合条件的记录在唯一索引记录上行锁排他。
      1. 会在该非唯一索引上加一个间隙锁，范围是该值的前后两个值，均为开区间。
          1. 即假设数据有333,4444,55555，那么间隙锁的范围是(334,55554)。
      1. 因为状态展示提示有5个行被锁了，可能是非唯一索引上的该值的记录有一个排他行锁。

    数据：

    ```ditaa
    @startuml
    ditaa
    +----+--------+----------+------------+---------+----------------+
    | id | val    | str      | no_idx_val | uni_val | val_with_idx_2 |
    +----+--------+----------+------------+---------+----------------+
    |  1 |    999 | record1  |       1111 |       1 |            999 |
    |  2 | 362324 | record2  |        222 |       2 |         362324 |
    |  3 |   3624 | record3  |          1 |       3 |           3624 |
    |  4 |    324 | record4  |          4 |       4 |            324 |
    |  5 |   9999 | record5  |          1 |       5 |           9999 |
    |  6 |   9999 | record6  |        999 |       6 |           9999 |
    |  7 |    999 | record7  |        998 |       7 |            999 |
    |  9 |      9 | record9  |          9 |       9 |              9 |
    | 10 |      8 | record8  |          8 |       8 |              8 |
    | 12 |     10 | record10 |         10 |      10 |             10 |
    | 14 |     13 | record13 |         13 |      13 |             13 |
    | 15 |     14 | record14 |         14 |      14 |             14 |
    +----+--------+----------+------------+---------+----------------+
    @enduml
    ```

    过程：
    ```mysql
    # 1. transaction A
    update t1 set val=998 where val=9999;
    # 2 transaction B
    # 2.1 数据上和A完全无关的数据或不存在的记录
    update t1 set val=222 where val=8888; # 不会阻塞，因为8888不存在对应的行
    # 2.2 相同的查询条件修改
    update t1 set val=222 where val=9999; # 阻塞在idx_val的排他锁上
    # 2.3 将其他记录的val修改改为9999/小于9999的最大val+1 即3625/大于9999的最小val-1 即362323
    update t1 set val=9999 where id=15; 
    # 阻塞，事务B试图获取idx_val上的插入意向锁，被一个gap排他锁阻塞。
    # 很奇怪，是因为RR吗？如果是RR那么MVCC应该就可以实现了，有必要加锁吗？
    # 2.4 尝试修改对应主键的记录的值
    update t1 set no_idx_val=5 where id=5; # 阻塞在主键的排他行锁上
    # 2.5 修改符合该条件的记录
    update t1 set no_idx_val = 233 where val=9999; # 阻塞在idx_val上的排他行锁

    ```

1. 非主键唯一索引相等搜索

todo

1. 非主键非唯一索引范围搜索

todo

1. 非主键唯一索引范围搜索

todo

## 3. 在无索引列上进行范围查询或更新。

todo

# 参考

- [MySQL手册-InnoDB锁与事务模型](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking-transaction-model.html)
