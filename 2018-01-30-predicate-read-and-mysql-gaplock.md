---
layout: post
title: 间隙锁和一些引申
date: 2018-01-30
tags: db
---

最近在研究数据库的事务和并发控制，顺带把以前碰到 MySQL 的间隙锁一起研究。

## 条件查询和幻象现象（Phantom Phenomenon）

在不考虑 MVCC 的情况下考虑下面两个事务：

T1：

```
select count(*) from instructor where dept name = 'Physics'
```

T2：

```
insert into instructor values (11111,’Feynman’, ’Physics’, 94000);
```

T1 的查询和 T2 的写入是冲突的，如果事务执行序列是 T1 => T2，那么最终的调度就要求是 select 要发生在 insert 之前。如果一个调度的 insert 发生在前，那么这个调度不是一个冲突可串行化调度，这个调度的执行结果和 T1 => T2 的执行结果不一样。

<!-- more -->

解决这个的办法是加锁。但是有个问题是没有个具体的数据，或者说实体可以用来加锁。如果是两个事务更新同一行数据，分别执行 update，那是可以在这行数据上加 X 锁。这里 T1 执行的是 select count，如果能够在某个实体上加锁，这种锁也必须要满足：加了这个锁以后，不会有新的数据满足 name = 'Physics'，同时现有的数据也不会被变更并满足 name = 'Physics' 。要锁住的是这个关系。

如果是 B+ 树，可以在索引上做文章。继续用上面的例子，如果在 name 列上加辅助索引，T1 就能直接在 B+ 树索引上跑。B+ 树的查找是要根据查找值找到一个叶子节点，再在叶子节点上确定出最终符合要求的数据行。T2 的写入也类似，如果要求数据写入之前必须先维护索引，那么要写入也要先查找 B+ 树，找到符合要求的叶子节点。这个时候 T1 和 T2 的冲突就可以通过在这个叶子节点上加锁解决了。select count 执行以后在 B+ 树的叶子节点上获取共享锁，insert 获取同一个叶子节点的排他锁。

另外一点是，select count 加在叶子节点上的共享锁封锁了其他事务想要在这个叶子节点上进行的写入操作。B+ 树的叶子节点其实记录了一个排序好了的 key 节点，每个节点有一个指针指向真实的数据。如图：

![](http://note-1255449501.file.myqcloud.com/2018-01-26-031853.png)

## MySQL 的间隙锁

http://www.cnblogs.com/zhoujinyi/p/3435982.html 这篇文章的间隙锁的第一个例子：

```
mysql> create table t(a int,key idx_a(a))engine =innodb;
Query OK, 0 rows affected (0.02 sec)

mysql> insert into t values(1),(3),(5),(8),(11);
Query OK, 5 rows affected (0.00 sec)
```

T1：

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t where a = 8 for update;
+------+
| a    |
+------+
|    8 |
+------+
1 row in set (0.01 sec)
```

T2：

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t values(6);
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

锁状态是这样的：

```
mysql> select * from information_schema.innodb_locks;
+---------------+-------------+-----------+-----------+---------------+------------+------------+-----------+----------+-------------------+
| lock_id       | lock_trx_id | lock_mode | lock_type | lock_table    | lock_index | lock_space | lock_page | lock_rec | lock_data         |
+---------------+-------------+-----------+-----------+---------------+------------+------------+-----------+----------+-------------------+
| 99172:513:4:5 | 99172       | X,GAP     | RECORD    | `db_test`.`t` | idx_a      |        513 |         4 |        5 | 8, 0x00000000020B |
| 99171:513:4:5 | 99171       | X         | RECORD    | `db_test`.`t` | idx_a      |        513 |         4 |        5 | 8, 0x00000000020B |
+---------------+-------------+-----------+-----------+---------------+------------+------------+-----------+----------+-------------------+
2 rows in set, 1 warning (0.00 sec)

mysql> select * from information_schema.innodb_lock_waits;
+-------------------+-------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------+-----------------+------------------+
| 99172             | 99172:513:4:5     | 99171           | 99171:513:4:5    |
+-------------------+-------------------+-----------------+------------------+
1 row in set, 1 warning (0.00 sec)
```

show engine innodb status 的 TRANSACTIONS 一段结果：

```
------------
TRANSACTIONS
------------
Trx id counter 99177
Purge done for trx's n:o < 99174 undo n:o < 0 state: running but idle
History list length 149
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 281479545796400, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 99172, ACTIVE 1520 sec inserting  (99172事务信息开始)
mysql tables in use 1, locked 1
LOCK WAIT 2 lock struct(s), heap size 1136, 4 row lock(s), undo log entries 3
MySQL thread id 22, OS thread handle 123145382735872, query id 258 localhost 127.0.0.1 root update
insert into t values(6)
Trx read view will not see trx with id >= 99172, sees < 99171
------- TRX HAS BEEN WAITING 25 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 513 page no 4 n bits 80 index idx_a of table `db_test`.`t` trx id 99172 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000008; asc     ;;
 1: len 6; hex 00000000020b; asc       ;;

------------------
TABLE LOCK table `db_test`.`t` trx id 99172 lock mode IX
RECORD LOCKS space id 513 page no 4 n bits 80 index idx_a of table `db_test`.`t` trx id 99172 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000008; asc     ;;
 1: len 6; hex 00000000020b; asc       ;;

---TRANSACTION 99171, ACTIVE 1541 sec   (99171事务信息开始)
4 lock struct(s), heap size 1136, 3 row lock(s)
MySQL thread id 3, OS thread handle 123145382178816, query id 233 localhost root
TABLE LOCK table `db_test`.`t` trx id 99171 lock mode IX
RECORD LOCKS space id 513 page no 4 n bits 72 index idx_a of table `db_test`.`t` trx id 99171 lock_mode X
Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 80000008; asc     ;;
 1: len 6; hex 00000000020b; asc       ;;

RECORD LOCKS space id 513 page no 3 n bits 72 index GEN_CLUST_INDEX of table `db_test`.`t` trx id 99171 lock_mode X locks rec but not gap
Record lock, heap no 5 PHYSICAL RECORD: n_fields 4; compact format; info bits 0
 0: len 6; hex 00000000020b; asc       ;;
 1: len 6; hex 00000001835e; asc      ^;;
 2: len 7; hex be00000225013d; asc     % =;;
 3: len 4; hex 80000008; asc     ;;

RECORD LOCKS space id 513 page no 4 n bits 72 index idx_a of table `db_test`.`t` trx id 99171 lock_mode X locks gap before rec
Record lock, heap no 6 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 4; hex 8000000b; asc     ;;
 1: len 6; hex 00000000020c; asc       ;;
```

有些疑问：

单从 innodb_locks 和 innodb_lock_waits 的查询结果看，innodb_lock_waits 查询到的是当前被阻塞的事务，99172 是 T1 的 insert。他请求的锁是 X,GAP。被 T1 事务的 X 锁给阻塞。有疑问啊 … select for update 加的是 X 的写锁，锁的状态查询结果看起来是没有加间隙锁（其实应该是加了的），那么后面的 insert 语句是要加间隙锁并且被阻塞，那么看起来是申请排他间隙锁并被一个排他写锁给阻塞了。排他写锁会阻塞排他间隙锁吗？select for update 难道不应该是加排他写锁和排他共享锁吗？

从 show engine innodb status 的 99171 事务的加锁情况看：

```
1：
RECORD LOCKS space id 513 page no 4 n bits 72 index idx_a of table `db_test`.`t` trx id 99171 lock_mode X 
Record lock, heap no 5 PHYSICAL RECORD: n_fields 2; compact format; info bits 0

2：
RECORD LOCKS space id 513 page no 3 n bits 72 index GEN_CLUST_INDEX of table `db_test`.`t` trx id 99171 lock_mode X locks rec but not gap
Record lock, heap no 5 PHYSICAL RECORD: n_fields 4; compact format; info bits 0

3：
RECORD LOCKS space id 513 page no 4 n bits 72 index idx_a of table `db_test`.`t` trx id 99171 lock_mode X locks gap before rec
Record lock, heap no 6 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
```

99171 事务执行的 select for update 其实加的是 next-key lock，他包含了 record lock 和 gap lock。上面的信息 2 的锁是 lock_mode X locks rec but not gap，是 record lock。上面的信息 3 是 lock_mode X locks gap before rec ，是 gap lock。所以在 RR 隔离级别下，MySQL 默认加的 X 锁不仅仅是一个 X record lock，还会加一个 gap lock，和一起就是 next-key lock。加一个 gap lock 是为了防止幻象读。

又有疑问，只考虑 RR 级别下：

1. gaplock 的间隙是如何确定的？如何联想到 B+ 树的叶子节点上？从 B+ 树的数据结构上去理解。
2. 如果是 MVCC，那么是不是就没有幻读了？至少执行 select 的时候不显示加上 for update 这样的加锁申明，是不是就不存在幻读了？
3. 假设排除 MVCC，select … for update 或者 delete from T where column = .. （delete 隐含一个查询），类似这样的「当前读」，这个名称网上都这么叫，用来说明是查询的不是 MVCC 的历史版本，而是当前版本的数据。这个时候如果没有 gaplock，幻读是会发生的。因为没有 gaplock 的话，select for update 不会去锁住间隙，同样的索引值的数据是能被写进库的。当然如果有 gaplock 的话，间隙内锁住且无法写入，那无论执行多少次 select，查询的结果都是一样的。这样理解对吗？待印证。
4. MySQL 有行锁，什么时候会启用行锁？至少，select * from T 应该是不会行锁的，？。
5. MySQL 的 RR 应该是会发生幻读的，触发幻读需要用 select for update 或者 delete from where .. 这样的语句才能发现，单纯的 select 因为存在 MVCC 的原因，是不会出现幻读的。这样理解对吗？待印证。
6. 理论上理解，严格的可重复读隔离级别应该是能避免的幻影读的，对吗？

##### 解答 1：

目前查到的资料都表明间隙是索引记录的间隙。硬要关联到索引结构（比如 B+ 树）的话，间隙锁锁住的是节点内部的索引项，可以是叶子节点和内部节点。因为区间最终是由节点内的索引项确定的。

##### 解答 2：

光 select 似乎是的，但是执行 delete from … 或者 insert （写入一个唯一列或者主键）又会发生了，因为这两个其实执行了当前读。但是网上的文章都说 MySQL 的 RR 能避免幻读 TODO。 

##### 解答 3：

引用：

> Phantom reads do not occur if you’re simply doing a SELECT. They only occur if you do UPDATE or DELETE or SELECT FOR UPDATE. InnoDB provides REPEATABLE READ for read-only SELECT, but it behaves as if you use READ COMMITTED for all write queries, in spite of your chosen transaction isolation level (considering only the two most common isolation levels, REPEATABLE READ and READ COMMITTED)
>
> https://www.percona.com/blog/2012/03/27/innodbs-gap-locks/

翻译：当只执行单纯的 select 时，幻读不会发生。update delete 和 select for update 才会导致幻读。innoDB 的可重复读只提供对只读的 select 支持。有的时候，即使选择了 RR 的隔离级别，在执行 write query 的时候，最终还是像在 RC 级别下一样（出现不可重复读）。

##### 解答 4：

不会，直接读快照。

##### 解答 5：

看解答 2

##### 解答 6：

应该是的，TODO。 

## 资料

http://www.cnblogs.com/zhoujinyi/p/3437475.html

http://www.cnblogs.com/zhoujinyi/p/3435982.html

https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html

http://hedengcheng.com/?p=771

http://www.fanyilun.me/2017/04/20/MySQL%E5%8A%A0%E9%94%81%E5%88%86%E6%9E%90/

https://www.percona.com/blog/2012/03/27/innodbs-gap-locks/

《数据库系统实现 第六版》