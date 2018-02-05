---
layout: post
title: MySQL文档翻译：幻影行
date: 2018-01-31
tags: db
---

> The so-called phantom problem occurs within a transaction when the same query produces different sets of rows at different times. For example, if a [SELECT](https://dev.mysql.com/doc/refman/5.7/en/select.html) is executed twice, but returns a row the second time that was not returned the first time, the row is a “phantom” row.

所谓的幻影读问题是在同一个事务内两次执行一个相同的查询，最终查询到的结果不一致。举个例子，如果一个 select 执行两次，第二次的查询结果返回了一行第一次查询所没有返回的结果，就说这行数据是一行「幻影」。

> Suppose that there is an index on the `id` column of the `child` table and that you want to read and lock all rows from the table having an identifier value larger than 100, with the intention of updating some column in the selected rows later:
>
> ```
> SELECT * FROM child WHERE id > 100 FOR UPDATE;
> ```

<!-- more -->

假设在 id 列上有索引，然后你想读取并锁住 child 表上所有 id > 100 的行，以用来后续对查询到的数据进行一些更新操作。

> The query scans the index starting from the first record where `id` is bigger than 100. Let the table contain rows having `id` values of 90 and 102. If the locks set on the index records in the scanned range do not lock out inserts made in the gaps (in this case, the gap between 90 and 102), another session can insert a new row into the table with an `id` of 101. If you were to execute the same [`SELECT`](https://dev.mysql.com/doc/refman/5.7/en/select.html) within the same transaction, you would see a new row with an `id` of 101 (a “phantom”) in the result set returned by the query. If we regard a set of rows as a data item, the new phantom child would violate the isolation principle of transactions that a transaction should be able to run so that the data it has read does not change during the transaction.

查询会扫描索引，直到从索引上扫到第一个 id > 100 的索引项。如果当期表里的数据包含 id 的区间为 90 到 102。如果在这些索引项（index record）上加的锁没有阻止在 id 90 - 102 这个区间上新增数据，如果其他事务能够新增新的数据（比如 id = 101）行到表中。之后如果你在同一个事务中执行同样的查询，你会在结果集中发现一行新的数据（id = 101），就像幻影一样。如果我们把这些数据行作为数据来处理，新的幻影数据（第二次查询相对于第一次多出的行）将会违反事务的隔离性原则。

> To prevent phantoms, `InnoDB` uses an algorithm called next-key locking that combines index-row locking with gap locking. `InnoDB` performs row-level locking in such a way that when it searches or scans a table index, it sets shared or exclusive locks on the index records it encounters. Thus, the row-level locks are actually index-record locks. In addition, a next-key lock on an index record also affects the “gap” before that index record. That is, a next-key lock is an index-record lock plus a gap lock on the gap preceding the index record. If one session has a shared or exclusive lock on record `R` in an index, another session cannot insert a new index record in the gap immediately before `R` in the index order.

为了阻止这种现象，innodb 使用一种称为 next-key 锁的算法，这种算法由 index-row locking 和 gaplock locking 组成（行锁和间隙锁）。innodb 通过这样一种方式实现行级锁，他在扫描索引时，他在索引项（index record）上加一个共享或者排他锁。所以，行级锁其实是索引项上的锁。此外，一个索引项上的 next-key 锁还会在这个索引项前施加一个间隙锁。next-key 锁其实是一个索引项上的锁外加一个锁住这个索引项之前一段间隙的「间隙锁」。如果一个事务在一个索引项上拥有一个共享或者排他的锁，其他事务不能再在这个索引项前的一段间隙中新增一个新的索引项。

> When `InnoDB` scans an index, it can also lock the gap after the last record in the index. Just that happens in the preceding example: To prevent any insert into the table where `id` would be bigger than 100, the locks set by `InnoDB` include a lock on the gap following `id` value 102.

当 innodb 扫描索引，他能同时锁住最后一个索引项后面的一段间隙。前述的例子中就有发生：为了阻止任何新 id > 100 的数据写入到表，innodb 会施加一个在 id 102 后间隙的锁。

> You can use next-key locking to implement a uniqueness check in your application: If you read your data in share mode and do not see a duplicate for a row you are going to insert, then you can safely insert your row and know that the next-key lock set on the successor of your row during the read prevents anyone meanwhile inserting a duplicate for your row. Thus, the next-key locking enables you to “lock” the nonexistence of something in your table.

你能利用 next-key 在你的应用中实现一个唯一检查逻辑：如果你在共享锁模式下没有读取到你要写入的唯一数据，然后你就可以安全的写入你的数据了，同时施加的 next-key 锁会阻值其他事务写入重复的数据进表。因此，next-key 锁就好像控制可表并让他满足「一些数据不该存在」的要求。

> Gap locking can be disabled as discussed in [Section 14.5.1, “InnoDB Locking”](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html). This may cause phantom problems because other sessions can insert new rows into the gaps when gap locking is disabled.

间隙锁是可以被禁用的，但是这会导致幻读问题。

## 资料：

https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html