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

<!--more-->

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
        U.compareAndSwapInt(q, QLOCK, 0, 1)) { // 加锁
        ForkJoinTask<?>[] a; int am, n, s;
        if ((a = q.array) != null &&
            (am = a.length - 1) > (n = (s = q.top) - q.base)) { // (1)
            int j = ((am & s) << ASHIFT) + ABASE;  // (2)
            U.putOrderedObject(a, j, task);   // 在数组的top位置新增任务
            U.putOrderedInt(q, QTOP, s + 1);  // 给top+1
            U.putIntVolatile(q, QLOCK, 0); // 释放锁
            if (n <= 1)  // (3)
                signalWork(ws, q);
            return;
        }
        // 释放锁
        U.compareAndSwapInt(q, QLOCK, 1, 0);
    }
    externalSubmit(task);
}
```

（1） 处，q.top - q.base 表示现在队列内的任务数，q.top - q.base < a.length - 1 表示队列内任务数小于数组容量，那么就会往下执行，在数组的 top 位置增加参数里的 task，并且给 top 加 1，最后释放锁。（2）处 am & s 的含义见下面「WorkQueue 的 top 和 base 的维护」的描述，原理一样。

（3）当队列内任务数为 0个 或者 1个 或者 2个时，去执行 signalWork。

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

#### lockRunState

```java
private int lockRunState() {
    int rs;
    return ((((rs = runState) & RSLOCK) != 0 ||
             !U.compareAndSwapInt(this, RUNSTATE, rs, rs |= RSLOCK)) ?
            awaitRunStateLock() : rs);
}
```

runState 与 RSLOCK 求与运算，检测 runState 中 lock 位，不为 0 表示已经锁住。如果没锁住，就去尝试 CAS 加锁，runState 和 RSLOCK 求或运算是将 runState 的 lock 位设置为 1，即加锁。如果成功返回 runState，失败则进入  awaitRunStateLock 方法。

#### awaitRunStateLock

```java
private int awaitRunStateLock() {
    Object lock;
    boolean wasInterrupted = false;
    for (int spins = SPINS, r = 0, rs, ns;;) {
        if (((rs = runState) & RSLOCK) == 0) {
            if (U.compareAndSwapInt(this, RUNSTATE, rs, ns = rs | RSLOCK)) {
                if (wasInterrupted) {
                    try {
                        Thread.currentThread().interrupt();
                    } catch (SecurityException ignore) {
                    }
                }
                return ns;
            }
        }
        else if (r == 0)
            r = ThreadLocalRandom.nextSecondarySeed();
        else if (spins > 0) {
            r ^= r << 6; r ^= r >>> 21; r ^= r << 7; // xorshift
            if (r >= 0)
                --spins;
        }
        else if ((rs & STARTED) == 0 || (lock = stealCounter) == null)
            Thread.yield();   // initialization race
        else if (U.compareAndSwapInt(this, RUNSTATE, rs, rs | RSIGNAL)) {
            synchronized (lock) {
                if ((runState & RSIGNAL) != 0) {
                    try {
                        lock.wait();
                    } catch (InterruptedException ie) {
                        if (!(Thread.currentThread() instanceof
                              ForkJoinWorkerThread))
                            wasInterrupted = true;
                    }
                }
                else
                    lock.notifyAll();
            }
        }
    }
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

WorkQueue 是一个用数组实现的双端队列，队列内存 ForkJoinTask。每个线程内部会维护一个 WorkQueue。当一个任务扩散出新的分支任务时，新的任务就会被 push 进双端队列的有段，这个操作执行的时候只会去 push 自己线程的 WorkQueue，所以这个动作不存在竞争，是线程安全的（至少队列非空时是的）。新扩散的任务进入双端队列的右边，用 top 来维护到底新增到了哪个数组下标记。任务的窃取就从队列的左边去处理，用 base 作为下标，当一个线程执行任务的 join 进入，但是任务不能立刻执行时会去其他线程盗取一个任务，也就是从那个线程的双端队列的左边，利用 base 索引去拿出一个任务，然后自己执行，这个动作是存在竞争的，详细的内容等看到了对应的代码以后再补充。 

```java
static final class WorkQueue {
	volatile int base;         // index of next slot for poll
    int top;                   // index of next slot for push
    ForkJoinTask<?>[] array;   // the elements (initially unallocated)
}
```

#### push

push 方法是双端队列右侧的新增操作。

```java
final void push(ForkJoinTask<?> task) {
    ForkJoinTask<?>[] a; ForkJoinPool p;
    int b = base, s = top, n;
    if ((a = array) != null) {    // ignore if queue removed
        int m = a.length - 1;     // fenced write for task visibility
        U.putOrderedObject(a, ((m & s) << ASHIFT) + ABASE, task);
        U.putOrderedInt(this, QTOP, s + 1);
        if ((n = s - b) <= 1) {
            if ((p = pool) != null)
                p.signalWork(p.workQueues, this);
        }
        else if (n >= m)
            growArray();
    }
}
```

#### poll

poll 是双端队列左侧的获取操作。

```java
final ForkJoinTask<?> poll() {
    ForkJoinTask<?>[] a; int b; ForkJoinTask<?> t;
    while ((b = base) - top < 0 && (a = array) != null) {
        int j = (((a.length - 1) & b) << ASHIFT) + ABASE;
        t = (ForkJoinTask<?>)U.getObjectVolatile(a, j);
        if (base == b) {
            if (t != null) {
                if (U.compareAndSwapObject(a, j, t, null)) {
                    base = b + 1;
                    return t;
                }
            }
            else if (b + 1 == top) // now empty
                break;
        }
    }
    return null;
}
```

### 说说两个方法的 base 和 top 的维护

push 每次都需要在数组的 top + 1 个位置新增任务，pull 每次需要 base + 1 个位置去盗取任务。他们都用类似这样的方式去计算需要操作的下标：

```
(array.length - 1) & (base或top的值)
```

这里比较有意思，因为 array 被初始化的容量为 1 << n（n 是写死的 13，每次扩容的时候再左移一位），1 << n 再减去 1 的二进制表示全为 1。所以 array.length - 1 的二进制表示全为 1，那么他和 base 或者 top 执行逻辑与，在 base 或者 top 小于等于 array.length - 1 时得到的结果就是 base 或 top 本身。当 base 或者 top 大于 array.length - 1 时，从新从最小的数组下标开始（即从 0 开始）。

随着 base 和 top 的增大，( array.length - 1 ) & ( base 或 top 的值 ) 这个变量计算的结果永远不会超出数组的下标范围，并且可以循环利用数组元素。这里非常的优雅，开始看了很久没看明白，后来在 debug 的时候才看出玄机。

### WorkQueue 的 top 和数组的维护为什么要用 putOrderedObject 和 putOrderedInt

TODO