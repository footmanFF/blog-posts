---
layout: post
title: AtomicInteger的lazySet探究
date: 2018-03-27
tags: AtomicInteger,lazySet,putOrderedInt
---

研究 ForkJoinPool 的时候碰到如下代码：

```java
final boolean tryUnpush(ForkJoinTask<?> t) {
    ForkJoinTask<?>[] a; int s;
    if ((a = array) != null && (s = top) != base &&
        U.compareAndSwapObject
        (a, (((a.length - 1) & --s) << ASHIFT) + ABASE, t, null)) {
        U.putOrderedInt(this, QTOP, s);
        return true;
    }
    return false;
}
```

U.putOrderedInt 这个方法有点迷惑，他和 AtomicInteger 的 lazySet 是一个机制。然后查了些资料：

https://www.zhihu.com/question/43691809
https://www.zhihu.com/question/60888757
https://blog.csdn.net/ITer_ZC/article/details/40744485
https://stackoverflow.com/questions/1468007/atomicinteger-lazyset-vs-set
http://bugs.sun.com/bugdatabase/view_bug.do?bug_id=6275329
https://stackoverflow.com/questions/7557156/atomicxxx-lazyset-in-terms-of-happens-before-edges/7561051#7561051
http://ifeve.com/juc-atomic-class-lazyset-que/
https://tech.meituan.com/java-memory-reordering.html

> The semantics are that the write is guaranteed not to be re-ordered with any previous write, but may be reordered with subsequent operations (or equivalently, might not be visible to other threads) until some other volatile write or synchronizing action occurs).

putOrderedInt 之前的写不会被重排序，之后的写会被重排序。之后会被重排序意味着执行以后对其他线程是没有可见性保证的。

> The main use case is for nulling out fields of nodes in non-blocking data structures solely for the sake of avoiding long-term garbage retention; it applies when it is harmless if other threads see non-null values for a while, but you'd like to ensure that structures are eventually GCable. In such cases, you can get better performance by avoiding the costs of the null volatile-write. There are a few other use cases along these lines for non-reference-based atomics as well, so the method is supported across all of the AtomicX classes.

当想设置一个 volatile 变量为 null，让这个对象被 GC 掉，volatile write 是消耗比较大（store-load 屏障）的，但是 putOrderedInt 只会加 store-store 屏障，损耗会小一些。

> For people who like to think of these operations in terms of machine-level barriers on common multiprocessors, lazySet provides a preceeding store-store barrier (which is either a no-op or very cheap on current platforms), but no store-load barrier (which is usually the expensive part of a volatile-write).

putOrderedInt 加的是 store-store 屏障，但是没有 store-load，volatile-write 两种屏障都加。

> **StoreStore Barriers**
>
> The sequence: Store1; StoreStore; Store2
> ensures that Store1's data are visible to other processors (i.e., flushed to memory) before the data associated with Store2 and all subsequent store instructions. In general, StoreStore barriers are needed on processors that do not otherwise guarantee strict ordering of flushes from write buffers and/or caches to other processors or main memory.

store1 写入的数据相对于 store2 可见意味着 store1 和 store2 无法重排序，在写入 volatile 时会在写入前插入StoreStore，用来保证对相同数据块的写入， volatile 写是最后一次写入。

> **StoreLoad Barriers**
>
> The sequence: Store1; StoreLoad; Load2
> ensures that Store1's data are made visible to other processors (i.e., flushed to main memory) before data accessed by Load2 and all subsequent load instructions are loaded. StoreLoadbarriers protect against a subsequent load incorrectly using Store1's data value rather than that from a more recent store to the same location performed by a different processor. Because of this, on the processors discussed below, a StoreLoad is strictly necessary only for separating stores from subsequent loads of the *same* location(s) as were stored before the barrier.StoreLoad barriers are needed on nearly all recent multiprocessors, and are usually the most expensive kind. Part of the reason they are expensive is that they must disable mechanisms that ordinarily bypass cache to satisfy loads from write-buffers. This might be implemented by letting the buffer fully flush, among other possible stalls.

StoreLoad 保证 Store1 写入的数据块相对于 Load2 可见。

#### 回到 tryUnpush 方法

putOrderedInt 写入的是 WorkQueue 的 top 属性，这个写入没有立刻保证后续的可见性。