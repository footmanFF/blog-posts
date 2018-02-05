---
layout: post
title: 如何查看MySQL加锁状态
date: 2018-01-30
tags: db
---

## information_schema.innodb_locks 

这个表罗列了两类锁：

- 事务请求了但是还没有获取到的锁
- 事务持有的锁，并且这个锁阻塞了其他事务的执行

> The INNODB_LOCKS table contains information about each lock that an InnoDB transaction has requested but not yet acquired, and each lock that a transaction holds that is blocking another transaction.

<!-- more -->

| Column name | 描述   | Description                              |
| ----------- | ---- | ---------------------------------------- |
| LOCK_ID     |      | Unique lock ID number, internal to InnoDB. Treat it as an opaque string. Although LOCK_ID currently contains TRX_ID, the format of the data in LOCK_ID is subject to change at any time. Do not write applications that parse the LOCK_ID value. |
| LOCK_TRX_ID |      | ID of the transaction holding the lock. To obtain details about the transaction, join this column with the TRX_ID column of theINNODB_TRX table. |
| LOCK_MODE   |      | How the lock is requested. Permitted values are S[,GAP], X[,GAP], IS[,GAP], IX[,GAP], AUTO_INC, and UNKNOWN. Lock modes other than AUTO_INC and UNKNOWN indicate gap locks, if present. For information about S, X, IS, IX, and gap locks, refer to Section 14.8.1, “InnoDB Locking”. |
| LOCK_TYPE   |      | The type of lock. Permitted values are `RECORD` for a row-level lock, `TABLE` for a table-level lock. |
| LOCK_TABLE  |      | Name of the table that has been locked or contains locked records. |
| LOCK_INDEX  |      | Name of the index, if LOCK_TYPE is RECORD; otherwise NULL. |
| LOCK_SPACE  |      | Tablespace ID of the locked record, if `LOCK_TYPE` is `RECORD`; otherwise `NULL`. |
| LOCK_PAGE   |      | Page number of the locked record, if `LOCK_TYPE` is `RECORD`; otherwise `NULL`. |
| LOCK_REC    |      | Heap number of the locked record within the page, if LOCK_TYPE is RECORD; otherwise NULL. |
| LOCK_DATA   |      | The data associated with the lock, if any. Values are primary key values of the locked record if LOCK_TYPE is RECORD, otherwise NULL. This column contains the values of the primary key columns in the locked row, formatted as a valid SQL string (ready to be copied to SQL statements). If there is no primary key, LOCK_DATA is the unique InnoDB internal row ID number. If a gap lock is taken for key values or ranges above the largest value in the index, LOCK_DATA reports supremum pseudo-record. When the page containing the locked record is not in the buffer pool (in the case that it was paged out to disk while the lock was held), InnoDB does not fetch the page from disk, to avoid unnecessary disk operations. Instead, LOCK_DATA is set to NULL. |

## information_schema.innodb_lock_waits

这个表罗列了所有当前是阻塞的事务，指出

> The INNODB_LOCK_WAITS table contains one or more rows for each blocked InnoDB transaction, indicating the lock it has requested and any locks that are blocking that request.

| Column name       | 描述            | Description                              |
| ----------------- | ------------- | ---------------------------------------- |
| REQUESTING_TRX_ID | 请求事务Id        | ID of the requesting (blocked) transaction. |
| REQUESTED_LOCK_ID | 请求事务申请的锁id    | ID of the lock for which a transaction is waiting. To obtain details about the lock, join this column with the LOCK_ID column of theINNODB_LOCKS table. |
| BLOCKING_TRX_ID   | 阻塞请求事务的其他事务id | ID of the blocking transaction.          |
| BLOCKING_LOCK_ID  | 阻塞请求事务的锁的id   | ID of a lock held by a transaction blocking another transaction from proceeding. To obtain details about the lock, join this column with the LOCK_ID column of the INNODB_LOCKS table. |

## 举个例子

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

另外注意，只有在存在锁冲突时，这两个表才会记录对应的数据。

## show engine innodb status

输出 innodb 的状态信息，其中有事务的加锁情况

```
set global innodb_status_output_locks=ON; // 输出的内容包含锁的详细信息
show engine innodb status;
```

一个结果：

```
| InnoDB |      |
=====================================
2018-01-31 10:29:10 0x700004c6f000 INNODB MONITOR OUTPUT
=====================================
Per second averages calculated from the last 22 seconds
-----------------
BACKGROUND THREAD
-----------------
srv_master_thread loops: 53 srv_active, 0 srv_shutdown, 17501 srv_idle
srv_master_thread log flush and writes: 17554
----------
SEMAPHORES
----------
OS WAIT ARRAY INFO: reservation count 156
OS WAIT ARRAY INFO: signal count 102
RW-shared spins 0, rounds 441, OS waits 144
RW-excl spins 0, rounds 1074, OS waits 2
RW-sx spins 6, rounds 136, OS waits 4
Spin rounds per wait: 441.00 RW-shared, 1074.00 RW-excl, 22.67 RW-sx
------------
TRANSACTIONS
------------
Trx id counter 99177
Purge done for trx's n:o < 99174 undo n:o < 0 state: running but idle
History list length 149
LIST OF TRANSACTIONS FOR EACH SESSION:
---TRANSACTION 281479545796400, not started
0 lock struct(s), heap size 1136, 0 row lock(s)
---TRANSACTION 99172, ACTIVE 1520 sec inserting
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

---TRANSACTION 99171, ACTIVE 1541 sec
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

--------
FILE I/O
--------
I/O thread 0 state: waiting for i/o request (insert buffer thread)
I/O thread 1 state: waiting for i/o request (log thread)
I/O thread 2 state: waiting for i/o request (read thread)
I/O thread 3 state: waiting for i/o request (read thread)
I/O thread 4 state: waiting for i/o request (read thread)
I/O thread 5 state: waiting for i/o request (read thread)
I/O thread 6 state: waiting for i/o request (write thread)
I/O thread 7 state: waiting for i/o request (write thread)
I/O thread 8 state: waiting for i/o request (write thread)
I/O thread 9 state: waiting for i/o request (write thread)
Pending normal aio reads: [0, 0, 0, 0] , aio writes: [0, 0, 0, 0] ,
 ibuf aio reads:, log i/o's:, sync i/o's:
Pending flushes (fsync) log: 0; buffer pool: 0
734 OS file reads, 732 OS file writes, 514 OS fsyncs
0.00 reads/s, 0 avg bytes/read, 0.00 writes/s, 0.00 fsyncs/s
-------------------------------------
INSERT BUFFER AND ADAPTIVE HASH INDEX
-------------------------------------
Ibuf: size 1, free list len 0, seg size 2, 0 merges
merged operations:
 insert 0, delete mark 0, delete 0
discarded operations:
 insert 0, delete mark 0, delete 0
Hash table size 34679, node heap has 1 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 2 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 0 buffer(s)
Hash table size 34679, node heap has 1 buffer(s)
0.00 hash searches/s, 0.00 non-hash searches/s
---
LOG
---
Log sequence number 405931474
Log flushed up to   405931474
Pages flushed up to 405931474
Last checkpoint at  405931465
0 pending log flushes, 0 pending chkp writes
313 log i/o's done, 0.00 log i/o's/second
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 370468
Buffer pool size   8192
Free buffers       7657
Database pages     531
Old database pages 211
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 478, created 53, written 344
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
Buffer pool hit rate 1000 / 1000, young-making rate 0 / 1000 not 0 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 531, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
1 read views open inside InnoDB
Process ID=4112, Main thread ID=123145376792576, state: sleeping
Number of rows inserted 19, updated 10, deleted 3, read 278
0.00 inserts/s, 0.00 updates/s, 0.00 deletes/s, 0.00 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
```

## 资料：

https://dev.mysql.com/doc/refman/5.5/en/innodb-locks-table.html

https://dev.mysql.com/doc/refman/5.5/en/innodb-lock-waits-table.html

http://www.fanyilun.me/2017/04/20/MySQL%E5%8A%A0%E9%94%81%E5%88%86%E6%9E%90/

https://dev.mysql.com/doc/refman/5.5/en/innodb-standard-monitor.html