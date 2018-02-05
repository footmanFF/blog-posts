---
layout: post
title: 数据库隔离级别和各个级别下的缺点
date: 2017-03-23
tags: db
---

# 2018-01-15 更新

事务的隔离性定义：

> 尽管多个事务可能并发执行，但系统保证，对于任何一对事务 T1和 T2，在 T1 看来，T2或者在 T1开始之前已经完成执行，或者在 T1完成之后开始执行。因此，每个事务都感觉不到系统中有其他事务在并发地执行。-------《数据库系统概念》

要完全遵守上面的定义可能只有让系统每个瞬间只有一个事务在运行才行。如果要让事务能够并行执行，事务之间一定会存在冲突。一个事务内对数据的操作只有读取和写入两种。首先写入冲突（脏写）一定是不被允许的。两次同时运行的事务，如果都对一块数据进行写入，如果一个事务的写操作覆盖了另一个事务的写操作，那被覆盖的那个事务就会蒙圈。明明我事务还没提交，才刚写入成 A，怎么再执行查询就变成 B 了（其他事务改成了 B）。这是不被允许的。

然后就是对数据的读取。如果一个数据没有被改动，那么再多的事务同时对他进行读取都是不会有问题的，问题发生在不同事务同时对数据进行读取和写入。事务的隔离级别其实是通过控制事务内读操作可以选取的数据范围来满足事务的隔离性要求。

<!-- more -->

当对并发事务不做任何事务隔离机制，读取操作可以读取任何其他事务操作中的数据。这样会发生如下这些违反隔离性的问题：

### dirty read

读取到其他事务未提交的数据。

### nonRepeatable read

同一个事务内，先后读取过两次同一块数据。这块数据在两次读取之间被其他事务修改过（并提交），最终两次的读取结果不一致。

这种情况读到内容虽然不脏，都是数据实际有过的状态（不是事务未执行完时的中间状态），隔离性要求事务之间不得相互影响，其他事务的写入操作都影响到了本事务内的两次读取操作的结果，显然是不隔离的。

### phantom read：

事务**无法查询到其他事务提交的数据**，而导致的写失败，具体如下：

> users： id 主键
>
> 1、T1：select * from users where id = 1;
>
> 2、T2：insert into users(id, name`) values (1, 'big cat');
>
> 3、T1：insert into users(id, name`) values (1, 'big cat');
>
> T1 ：主事务，检测表中是否有 id 为 1 的记录，没有则插入，这是我们期望的正常业务逻辑。
>
> T2 ：干扰事务，目的在于扰乱 T1 的正常的事务执行。
>
> 在 RR 隔离级别下，1、2 是会正常执行的，3 则会报错主键冲突，对于 T1 的业务来说是执行失败的，这里 T1 就是发生了幻读，因为T1读取的数据状态并不能支持他的下一步的业务，见鬼了一样。
>
> 引自：https://www.zhihu.com/question/47007926

另外这个链接关于幻读的例子也比较有价值：http://www.cnblogs.com/zhoujinyi/p/3437475.html

以上两个例子其实是一样的，都是因为插入事先检查不存在的数据时，数据库反馈插入的数据已经存在了。「幻」体现在开始的查询返回没有，后来执行写入又说有，最初的查询像幻觉一样。

关于幻读，还有这个资料提到了：

> Phantom reads do not occur if you’re simply doing a SELECT. They only occur if you do UPDATE or DELETE or SELECT FOR UPDATE
>
> http://www.cnblogs.com/zuoxingyu/archive/2012/09/27/2705285.html

### 其他

脏读只需要限制事务只能读取到其他事务提交的结果就能解决了。

不可重复读需要在事务开始时给所操作数据定一个快照，事务内不管怎么读，都只能读取事务初定下的数据的快照版本，看不到后来快照时间戳之后发生的任何新增和更新。这个时候就不管怎么重复读，结果都是一致的。当然快照的版本不能在其他事务有未提交数据时选取。

```
这里有一个问题，快照机制如果说是对当前数据的一个快照，事务内对这块的数据查询的结果一直是一致的。按照这个思路，对数据的更新确实是可以一致，那新增数据呢。这个看起来是间隙锁干的事，是吗？
```

应该是的，见：http://www.cnblogs.com/zuoxingyu/archive/2012/09/27/2705285.html，**TODO** ...

幻读的话用序列化的隔离级别一定能解决。此外 MySQL 的话在可重复读级别下有个办法：

> 在默认的可重复读的隔离级别里，可以使用加锁读去查询最新的数据（提交读）
>
> 见这篇的最后：http://www.cnblogs.com/zhoujinyi/p/3437475.html

幻读的解决需要继续研究 ..

最后就可以填下这个网上见了 N 次的表格了

| 隔离级别             | Dirty read | NonRepeatable read | Phantom read |
| ---------------- | ---------- | ------------------ | ------------ |
| Read uncommitted | 可能         | 可能                 | 可能           |
| Read committed   | 不可能        | 可能                 | 可能           |
| Repeatable read  | 不可能        | 不可能                | 可能           |
| Serializable     | 不可能        | 不可能                | 不可能          |

> MySQL InnoDB的可重复读并不保证避免幻读，需要应用使用加锁读来保证。而这个加锁度使用到的机制就是[next-key locks](http://www.cnblogs.com/zhoujinyi/p/3435982.html)。
>
> 引自：http://www.cnblogs.com/zhoujinyi/p/3437475.html

Next-key locks  **TODO** …   =>  2018-01-30-predicate-read-and-mysql-gaplock 有解释

幻读的 MySQL 官方文档解释：https://dev.mysql.com/doc/refman/5.7/en/innodb-next-key-locking.html

# 旧文：

### read uncommited

这个级别中允许事务读取其他事务未提交的数据，那么如果提交的数据最终被回滚，前一次读取的数据其实最终是未持久化的数据，这就造成了问题，通常称为脏读。

### read commited

这个级别中允许事务读取其他事务已经提交了的数据，也还是有问题的。事务的隔离要求事务间相互隔离，如果一个事务能读取到其他他事务commit的数据，就违反了隔离性。这个问题通常称为不可重复读。

下一个隔离级别就是为了解决不可重读读问题。他的思路是在针对一个事务，将这个事务能访问到的所有数据都静止在一个版本，其实就是事务开始的时候有一个时间戳，这个时间戳作为版本，所有其他数据维持在这个版本不变，直到当前事务的最终提交为止。就像建立了一个平行宇宙，真正的宇宙还在推进，但是事务内有一个平行的禁止的宇宙，只供给当前事务访问，怎么读都不会重复，因为是禁止的：）。平行宇宙的说法是网上看到的，写到这里顺便用上，想想还正式贴切：）。

### repeatable read

数据库保证每个事务开始以后，在事务内读取到的数据都是在事务之初的那个版本，即在事务内多次读取同一块数据都是一样的。但是会有个问题是如果事务内有新增的数据，那么可能第一次没查询到数据，第二次又查询到。这是个反常现象，因为这违反了事务隔离性原则。事务隔离性原则要求并行的事务互相之间不影响。前面提到的反常现象通常称为「幻读」。

##### 2017-04-04 update：

上面的描述是错误的：「那么可能第一次没查询到数据，第二次又查询到」这个是不会出现的。可重复读如字面意思，保证了在一个事务内，重复读取的结果是一致。虽然能保证读取一致，但是如果在第一个事务里执行了会影响第二个事务新增行的写入动作，这个时候数据其实是存在的。如果第二个事务新增了主键为1的数据行，第一个事务再去新增主键为1的数据的时候就会报错了。但是第一个事务去查询的时候却没有主键1的数据，这才是所谓的「幻读」[http://www.cnblogs.com/zhoujinyi/p/3437475.html][1]

##### 2018-01-15 update：

**「但是会有个问题是如果事务内有新增的数据，那么可能第一次没查询到数据，第二次又查询到。这是个反常现象，因为这违反了事务隔离性原则」**这段有问题，第二次也不会查到这个数据的。但是如果其他事务新增的数据本身有约束条件（比如主键唯一），那么本事务虽然查不到数据，但是写入确会报错（违反主键约束），这个是 repeatable read 没有解决的问题，就是欢读。明明读数据的时候告诉我没有，写的时候告诉我已经存在，之前的读取是假的吗 ...

### serializable

序列化执行，但其实不是真正意义上的串行执行，如果事务是「可序列化」的（具体见《数据库系统概念》事务一章的描述），那么也是能够并行运行的，但是「可序列化」本身要求不低，不是所有种类的事务操作都支持，这个可以展开来继续研究。

### 另外

无论哪个级别，数据库都是不允许脏写的。两次并行事务同时对一块数据进行写入，必然会是后一个等待前一个写完。不会同时写入，其中一个丢失的。举个例子就是，比如 update sometable set column1=2 where version=1，数据库保证了如果有两个事务同时运行上面的语句，那么只会有一条语句返回1。

### 问题
《数据库系统概念》中，事务包含指令，并发执行事务如何协调这些指令的并行执行，其策略是什么。调度策略中的「可序列化」是什么。

[1]:	http://www.cnblogs.com/zhoujinyi/p/3437475.html