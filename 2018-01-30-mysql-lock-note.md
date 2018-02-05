---
layout: post
title: MySQL一例锁分析
date: 2018-01-30
tags: db
---

```
show create table banner;
CREATE TABLE `banner` (
  `id` int(11) NOT NULL AUTO_INCREMENT COMMENT '元素ID',
  `name` varchar(20) NOT NULL COMMENT '元素名称',
  `city` varchar(20) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `index_` (`name`)
) ENGINE=InnoDB AUTO_INCREMENT=40 DEFAULT CHARSET=utf8;

mysql> select * from banner;
+----+------+------+
| id | name | city |
+----+------+------+
|  3 | a    | 2    |
| 33 | xx   | 1    |
| 34 | xx   | 1    |
| 35 | xx   | 1    |
| 36 | a4   | 1    |
| 37 | a4   | 1    |
| 38 | a4   | 0    |
| 41 | a    | 0    |
+----+------+------+
```

<!-- more -->

T1：

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into banner values (null, 'a', '0');
Query OK, 1 row affected (0.00 sec)
```

T2：

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)
mysql> select count(*) from banner where name='a' LOCK IN SHARE MODE;
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```

锁的情况：

```
mysql> select * from information_schema.innodb_locks;
+-------------------------+-----------------+-----------+-----------+--------------------+------------+------------+-----------+----------+-----------+
| lock_id                 | lock_trx_id     | lock_mode | lock_type | lock_table         | lock_index | lock_space | lock_page | lock_rec | lock_data |
+-------------------------+-----------------+-----------+-----------+--------------------+------------+------------+-----------+----------+-----------+
| 281479545797304:362:4:3 | 281479545797304 | S         | RECORD    | `db_test`.`banner` | index_     |        362 |         4 |        3 | 'a', 42   |
| 99154:362:4:3           | 99154           | X         | RECORD    | `db_test`.`banner` | index_     |        362 |         4 |        3 | 'a', 42   |
+-------------------------+-----------------+-----------+-----------+--------------------+------------+------------+-----------+----------+-----------+
2 rows in set, 1 warning (0.00 sec)

mysql> select * from information_schema.innodb_lock_waits;
+-------------------+-------------------------+-----------------+------------------+
| requesting_trx_id | requested_lock_id       | blocking_trx_id | blocking_lock_id |
+-------------------+-------------------------+-----------------+------------------+
| 281479545797304   | 281479545797304:362:4:3 | 99154           | 99154:362:4:3    |
+-------------------+-------------------------+-----------------+------------------+
1 row in set, 1 warning (0.00 sec)
```

### 问题：

这是什么锁在起作用？

T1 的 insert 在 42 这个 id 上加了 X 行锁，T1 的查询会去加 S 行锁，然后就冲突了。delete 加 X 锁可以理解，T1 的查询为什么会加行锁，是回表吗？从辅助索引（name列）扫到数据以后回表去找查最终的数据行，这个查找会加 S 行锁？看起来是这个样子的 … TODO

##### 2018-02-05 更新：

```
TRANSACTIONS
------------
Trx id counter 99320
Purge done for trx's n:o < 99319 undo n:o < 0 state: running but idle
History list length 12
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 281479545798208, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 99319, ACTIVE 68 sec fetching rows
mysql tables in use 1, locked 1
LOCK WAIT 3 lock struct(s), heap size 1136, 5 row lock(s)
MySQL thread id 57, OS thread handle 123145382178816, query id 594 localhost root Sending data
select count(*) from banner where name='a' LOCK IN SHARE MODE
------- TRX HAS BEEN WAITING 2 SEC FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 362 page no 4 n bits 80 index index_ of table `db_test`.`banner` trx id 99319 lock mode S waiting
Record lock, heap no 11 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 1; hex 61; asc a;;
 1: len 4; hex 8000002b; asc    +;;

------------------
TABLE LOCK table `db_test`.`banner` trx id 99319 lock mode IS
RECORD LOCKS space id 362 page no 4 n bits 80 index index_ of table `db_test`.`banner` trx id 99319 lock mode S
Record lock, heap no 3 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 1; hex 61; asc a;;
 1: len 4; hex 8000002a; asc    *;;

Record lock, heap no 7 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 1; hex 61; asc a;;
 1: len 4; hex 80000003; asc     ;;

Record lock, heap no 10 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 1; hex 61; asc a;;
 1: len 4; hex 80000029; asc    );;

RECORD LOCKS space id 362 page no 4 n bits 80 index index_ of table `db_test`.`banner` trx id 99319 lock mode S waiting
Record lock, heap no 11 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 1; hex 61; asc a;;
 1: len 4; hex 8000002b; asc    +;;

---TRANSACTION 99314, ACTIVE 74 sec
2 lock struct(s), heap size 1136, 1 row lock(s), undo log entries 1
MySQL thread id 56, OS thread handle 123145383014400, query id 571 localhost root
TABLE LOCK table `db_test`.`banner` trx id 99314 lock mode IX
RECORD LOCKS space id 362 page no 4 n bits 80 index index_ of table `db_test`.`banner` trx id 99314 lock_mode X locks rec but not gap
Record lock, heap no 11 PHYSICAL RECORD: n_fields 2; compact format; info bits 0
 0: len 1; hex 61; asc a;;
 1: len 4; hex 8000002b; asc    +;;
```

insert 加 X 锁的原理是啥？是加在辅助索引还是主键索引上？select count() 加锁被阻塞是阻塞在主键索引还是辅助索引上？

理解的时候跟 insert intention lock 混了。[innodb-locking](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html) 这个页面的 Insert Intention Locks 是：insert 之前会在间隙上增加一个意向锁，和这里的例子不同。T1 的 insert 在未提交之前，在未提交之前应该是在辅助索引的 'a' 上加 X 锁，在主键索引上加 X 锁。T2 想通过辅助索引加共享锁（next-key lock），自然被阻塞了（这部分更多属于基于当时理解的猜测，待日后确定）。

### 资料：

http://www.fanyilun.me/2017/04/20/MySQL%E5%8A%A0%E9%94%81%E5%88%86%E6%9E%90/

http://hedengcheng.com/?p=771