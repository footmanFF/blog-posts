---
layout: post
title: jdk8 fork/join 框架探究
date: 2018-03-22
tags: fork/join
---
## 直接看代码

#### join

```java
public final V join() {
    int s;
    if ((s = doJoin() & DONE_MASK) != NORMAL)
        // 如果没有正常完成去处理异常
        reportException(s);
    return getRawResult();
}
```

#### doJoin

```java
private int doJoin() {
    int s; Thread t; ForkJoinWorkerThread wt; ForkJoinPool.WorkQueue w;
    // status<0表示任务已经执行完毕
    return (s = status) < 0 ? s :
    ((t = Thread.currentThread()) instanceof ForkJoinWorkerThread) ?
        (w = (wt = (ForkJoinWorkerThread)t).workQueue).
        tryUnpush(this) && (s = doExec()) < 0 ? s :
    wt.pool.awaitJoin(w, this, 0L) :
    externalAwaitDone();
}
```

#### externalAwaitDone

```java
private int externalAwaitDone() {
    int s = ((this instanceof CountedCompleter) ? // try helping
             ForkJoinPool.common.externalHelpComplete(
                 (CountedCompleter<?>)this, 0) :
             ForkJoinPool.common.tryExternalUnpush(this) ? doExec() : 0);
    if (s >= 0 && (s = status) >= 0) {
        boolean interrupted = false;
        do {
            if (U.compareAndSwapInt(this, STATUS, s, s | SIGNAL)) {
                synchronized (this) {
                    if (status >= 0) {
                        try {
                            wait(0L);
                        } catch (InterruptedException ie) {
                            interrupted = true;
                        }
                    }
                    else
                        notifyAll();
                }
            }
        } while ((s = status) >= 0);
        if (interrupted)
            Thread.currentThread().interrupt();
    }
    return s;
}
```

#### externalPush

```java
final void externalPush(ForkJoinTask<?> task) {
    WorkQueue[] ws; WorkQueue q; int m;
    int r = ThreadLocalRandom.getProbe();
    int rs = runState;
    if ((ws = workQueues) != null && (m = (ws.length - 1)) >= 0 &&
        (q = ws[m & r & SQMASK]) != null && r != 0 && rs > 0 &&
        U.compareAndSwapInt(q, QLOCK, 0, 1)) {
        ForkJoinTask<?>[] a; int am, n, s;
        if ((a = q.array) != null &&
            (am = a.length - 1) > (n = (s = q.top) - q.base)) {
            int j = ((am & s) << ASHIFT) + ABASE;
            U.putOrderedObject(a, j, task);
            U.putOrderedInt(q, QTOP, s + 1);
            U.putIntVolatile(q, QLOCK, 0);
            if (n <= 1)
                signalWork(ws, q);
            return;
        }
        U.compareAndSwapInt(q, QLOCK, 1, 0);
    }
    externalSubmit(task);
}
```

#### tryUnpush

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

#### doExec

```java
final int doExec() {
    int s; boolean completed;
    if ((s = status) >= 0) {
        try {
            completed = exec();
        } catch (Throwable rex) {
            return setExceptionalCompletion(rex);
        }
        if (completed)
            s = setCompletion(NORMAL);
    }
    return s;
}
```

#### awaitJoin

```java
final int awaitJoin(WorkQueue w, ForkJoinTask<?> task, long deadline) {
    int s = 0;
    if (task != null && w != null) {
        ForkJoinTask<?> prevJoin = w.currentJoin;
        U.putOrderedObject(w, QCURRENTJOIN, task);
        CountedCompleter<?> cc = (task instanceof CountedCompleter) ?
            (CountedCompleter<?>)task : null;
        for (;;) {
            if ((s = task.status) < 0)
                break;
            if (cc != null)
                helpComplete(w, cc, 0);
            else if (w.base == w.top || w.tryRemoveAndExec(task))
                helpStealer(w, task);
            if ((s = task.status) < 0)
                break;
            long ms, ns;
            if (deadline == 0L)
                ms = 0L;
            else if ((ns = deadline - System.nanoTime()) <= 0L)
                break;
            else if ((ms = TimeUnit.NANOSECONDS.toMillis(ns)) <= 0L)
                ms = 1L;
            if (tryCompensate(w)) {
                task.internalWait(ms);
                U.getAndAddLong(this, CTL, AC_UNIT);
            }
        }
        U.putOrderedObject(w, QCURRENTJOIN, prevJoin);
    }
    return s;
}
```

## ForkJoinTask 的 status

ForkJoinTask 的 status 最左侧 4 位是任务状态，最右边 2 个字节是标签，最右边的 16 位 ForkJoinTask 本身不会去使用，他提供了一系列 public 方法给用户去设置和获取，提供出这个是为了让用户去给任务打标签，用来避免一些任务分发时的问题，具体见 [ForkJoinTask](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinTask.html) 。status 左侧 4 位的定义如下：

| item        | binary                                     | desc                                        |
| ----------- | ------------------------------------------ | ------------------------------------------- |
| DONE_MASK   | 11110000 - 00000000 - 00000000 - 00000000  | 用于执行位运算<br />比如 status & DONE_MASK |
| NORMAL      | 11110000 - 00000000 - 00000000 - 00000000  | 任务正常完成，负数                          |
| CANCELLED   | 11000000 - 00000000 - 00000000 - 00000000  | 任务被取消，负数，小于 NORMAL               |
| EXCEPTIONAL | 10000000 - 00000000 - 00000000 - 00000000  | 任务出现异常，负数，小于 CANCELLED          |
| SIGNAL      | 00000000 - 000000001 - 00000000 - 00000000 | 任务等待其他任务完成，处于等待状态，正数    |
| SMASK       | 00000000 - 00000000 - 11111111 - 11111111  | 用来执行位运算<br />比如 tag & SMASK        |

关于 tag 多提一下，应为 tag 是 status 的前 16 位置，在设置 tag 的方法需要用一些位运算来处理：

```java
public final short setForkJoinTaskTag(short tag) {
    for (int s;;) {
        // s & ~SMASK 结果的左侧16位和当前status一样，右侧16位全为0
        // tag & SMASK 结果的左侧16位全为0，右侧16位和参数tag一致
        // 上面两者执行或运算，正好合并tag和当前的status
        if (U.compareAndSwapInt(this, STATUS, s = status,
                                (s & ~SMASK) | (tag & SMASK)))
            return (short)s;
    }
}
public final short getForkJoinTaskTag() {
    // 获取status的右侧16位，直接强转成short，抛弃status左侧16位的二进制位
    return (short)status;
}
```

## WorkQueue 的 top 和 base 的维护

WorkQueue 的是一个用数组实现的双端队列，队列内存 ForkJoinTask，每次新建，



左侧用于窃取

```java
static final class WorkQueue {
	volatile int base;         // index of next slot for poll
    int top;                   // index of next slot for push
}
```

