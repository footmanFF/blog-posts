---
layout: post
title: MySQL一例间隙锁分析
date: 2018-02-01
tags: db, gaplock, mysql
---
## 研究间隙锁的时候碰到一个问题

```
CREATE TABLE `t2` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_a` (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

mysql> select  * from t2;
+----+------+
| id | a    |
+----+------+
|  1 |    2 |
|  3 |    4 |
|  6 |    5 |
|  8 |    5 |
| 10 |    5 |
| 13 |   11 |
+----+------+
```

<!-- more -->

T1：

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select  * from t2 where a=4 for update;
+----+------+
| id | a    |
+----+------+
|  3 |    4 |
+----+------+
1 row in set (0.00 sec)
```

T2：

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t2 values (4, 5);
// 超时

mysql> insert into t2 values (7, 5);
Query OK, 1 row affected (0.00 sec)

mysql> insert into t2 values (null, 5);
Query OK, 1 row affected (0.00 sec)
```

#### 问题：

为什么 T2 的第一个 insert 不能执行，间隙锁锁定难道不是区间 (4, 5] 吗？例子来自：

https://www.jianshu.com/p/bf862c37c4c9。

为什么 T2 第二个 insert 不能执行。第二个和第一个的 insert 只是因为 id 不同就最终的行为不同。第三条sql也可以执行。

表的索引结构是这样的：

![](http://note-1255449501.file.myqcloud.com/2018-02-04-073735.png)

间隙锁锁住的不是索引值的间隙，之前一直理解的是值的间隙。比如 where a=4，如果是值的间隙就是 (2, 4] 和 (4, 5]，这样理解不够完整。间隙锁住应该是索引的间隙，他锁住的是 B+ 树的节点，他的语义是「如果某两个索引项之间的间隙被锁住以后，这两个索引项之间在解锁前不能再写入新的索引项」，对照 t2 表在 a 上的索引，(a=2, id=1)  和 (a=5, id=6) 之间不能再写入新的索引项。

(id=4, a=5) 不能写入是因为他落在了间隙锁的间隙内了，他会被插到 (id=3, a=4) 和 (id=6, a=5) 之间。(id=7, a=5) 能写入是因为他会被插到 (id=6, a=5) 和 (id=8, a=5) 之间，这个区间不在间隙锁的间隙内，所以可以写入。 (id=null, a=5) 能写入和  (id=7, a=5)  的原因一样，id 列指定为 Null 时 mysql 会分配个自增的 id ，这个新的 id 显然不在间隙所的间隙内。

#### 如果是 t2 没有主键：

```
mysql> CREATE TABLE `t_noid` (
    ->   `a` int(11) DEFAULT NULL,
    ->   KEY `idx_a` (`a`)
    -> ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.02 sec)

mysql>
mysql> insert into t_noid values (1), (3), (5), (8), (11);
Query OK, 5 rows affected (0.01 sec)
Records: 5  Duplicates: 0  Warnings: 0
```

T1：

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from t_noid where a=4 for update;
Empty set (0.00 sec)
```

T2：

```
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into t_noid values (5);
Query OK, 1 row affected (0.00 sec)

mysql> insert into t_noid values (2);
Query OK, 1 row affected (0.00 sec)

mysql> insert into t_noid values (3);
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
mysql> insert into t_noid values (4);
^C^C -- query aborted
ERROR 1317 (70100): Query execution was interrupted
```

没有主键的辅助索引是这样的：

![](http://note-1255449501.file.myqcloud.com/2018-02-04-082221.png)

insert 5 能执行是因为隐藏的 id 会递增 7，写入 (id=7, a=5) 不在间隙内。3 和 4 都在值的间隙内，无法写入。2 不在间隙内。

## 资料

https://www.jianshu.com/p/bf862c37c4c9