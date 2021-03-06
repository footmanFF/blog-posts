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
        //(4)
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

（4）m & r & SQMASK，m & r 是随机取一个 0 - m 的数，再与 SQMASK 求与获得一个将二进制最低位设为 0，得到一个偶数。外部任务提交的时候，会取 workQueues 上偶数下标的 WorkQueue，然后将任务加到队列内。

#### externalSubmit

```java
private void externalSubmit(ForkJoinTask<?> task) {
    int r;                                    // initialize caller's probe
    if ((r = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();
        r = ThreadLocalRandom.getProbe();
    }
    for (;;) {
        WorkQueue[] ws; WorkQueue q; int rs, m, k;
        boolean move = false;
        if ((rs = runState) < 0) {
            tryTerminate(false, false);     // help terminate
            throw new RejectedExecutionException();
        }
        else if ((rs & STARTED) == 0 ||     // initialize
                 ((ws = workQueues) == null || (m = ws.length - 1) < 0)) {
            int ns = 0;
            rs = lockRunState();
            try {
                if ((rs & STARTED) == 0) {
                    U.compareAndSwapObject(this, STEALCOUNTER, null,
                                           new AtomicLong());
                    // create workQueues array with size a power of two
                    int p = config & SMASK; // ensure at least 2 slots
                    int n = (p > 1) ? p - 1 : 1;
                    // 这里的位运算见下面对SMASK的解析
                    n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4;
                    n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1;
                    workQueues = new WorkQueue[n];
                    ns = STARTED;
                }
            } finally {
                unlockRunState(rs, (rs & ~RSLOCK) | ns);
            }
        }
        // 获取WorkQueues的偶数位的队列，并尝试增加任务
        else if ((q = ws[k = r & m & SQMASK]) != null) {
            if (q.qlock == 0 && U.compareAndSwapInt(q, QLOCK, 0, 1)) {
                ForkJoinTask<?>[] a = q.array;
                int s = q.top;
                boolean submitted = false; // initial submission or resizing
                try {                      // locked version of push
                    if ((a != null && a.length > s + 1 - q.base) ||
                        (a = q.growArray()) != null) {
                        int j = (((a.length - 1) & s) << ASHIFT) + ABASE;
                        U.putOrderedObject(a, j, task);
                        U.putOrderedInt(q, QTOP, s + 1);
                        submitted = true;
                    }
                } finally {
                    U.compareAndSwapInt(q, QLOCK, 1, 0);
                }
                if (submitted) {
                    signalWork(ws, q);
                    return;
                }
            }
            move = true;                   // move on failure
        }
        // 创建在WorkQueues的偶数位创建队列
        else if (((rs = runState) & RSLOCK) == 0) { // create new queue
            q = new WorkQueue(this, null);
            q.hint = r;
            q.config = k | SHARED_QUEUE;
            q.scanState = INACTIVE;
            rs = lockRunState();           // publish index
            if (rs > 0 &&  (ws = workQueues) != null &&
                k < ws.length && ws[k] == null)
                ws[k] = q;                 // else terminated
            unlockRunState(rs, rs & ~RSLOCK);
        }
        else
            move = true;                   // move if busy
        if (move)
            r = ThreadLocalRandom.advanceProbe(r);
    }
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

#### lockRunState

```java
private int lockRunState() {
    int rs;
    return ((((rs = runState) & RSLOCK) != 0 ||
             !U.compareAndSwapInt(this, RUNSTATE, rs, rs |= RSLOCK)) ?
            awaitRunStateLock() : rs);
}
```

runState 与 RSLOCK 求与运算，检测 runState 中 lock 位，不为 0 表示已经锁住。如果没锁住，就去尝试 CAS 加锁，runState 和 RSLOCK 求或运算是将 runState 的 lock 位设置为 1，即加锁。如果成功返回 runState，失败则进入  awaitRunStateLock 方法。利用 runState 的最右侧的二进制位作为加锁标识。

#### awaitRunStateLock

```java
// 加锁失败以后进入等待，等待锁变为可用
private int awaitRunStateLock() {
    Object lock;
    boolean wasInterrupted = false;
    // 自旋
    for (int spins = SPINS, r = 0, rs, ns;;) {
        // 自旋中发现锁可用
        if (((rs = runState) & RSLOCK) == 0) {
            // 重新尝试加锁
            if (U.compareAndSwapInt(this, RUNSTATE, rs, ns = rs | RSLOCK)) {
                // 处理中断标识
                if (wasInterrupted) {
                    try {
                        Thread.currentThread().interrupt();
                    } catch (SecurityException ignore) {
                    }
                }
                // 成功即可返回
                return ns;
            }
        }
        else if (r == 0)
            // 设置为下一个伪随机数
            r = ThreadLocalRandom.nextSecondarySeed();
        else if (spins > 0) {
            // 如果自旋次数大于0，利用伪随机数，随机自旋一定次数
            r ^= r << 6; r ^= r >>> 21; r ^= r << 7; // xorshift
            if (r >= 0)
                --spins;
        }
        else if ((rs & STARTED) == 0 || (lock = stealCounter) == null)
            // rs & STARTED 为0，或者stealCounter为null表示未"初始化"
            Thread.yield();   // initialization race
        else if (U.compareAndSwapInt(this, RUNSTATE, rs, rs | RSIGNAL)) {
            // 在自旋重试以后，任然无法获得锁，就去设置runState的signal状态
            synchronized (lock) {
                if ((runState & RSIGNAL) != 0) {
                    try {
                        // 双重检测以后，让线程进入wait
                        // 此处由unlockRunState方法来唤醒
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

#### unlockRunState

```java
private void unlockRunState(int oldRunState, int newRunState) {
    // (1)
    if (!U.compareAndSwapInt(this, RUNSTATE, oldRunState, newRunState)) {
        Object lock = stealCounter;
        // (2)
        runState = newRunState;              // clears RSIGNAL bit
        if (lock != null)
            synchronized (lock) { lock.notifyAll(); }
    }
}
```

「1」处的 CAS 是为了设置 runState 为 newRunState，也比较好理解，那为什么会有失败的情况呢（要不然也不会有 if 里面的代码）。失败的情况发生在当锁被获取以后，又有新的线程去尝试加锁，这个时候会进入 awaitRunStateLock 方法，这个方法的逻辑见上述，最后在尝试了自旋重试还是无法获得锁时，会去设置 runState 的 signal 位为1，相当于标识了「当前有线程处于阻塞等待中，需要被唤醒」。这样，自然 unlockRunState 方法的 CAS 操作就失败了，然后 if 块里的代码逻辑也比较清楚了。newRunState 的 signal 位没有被设置，「2」处会清除 signal 位为 0。然后在 lock 上调用 notifyAll 去唤醒阻塞的线程。

## 一些理解了的机制

### lockRunState 和 unlockRunState

runState 是一个整形，他是一个符合的状态变量，用他的二进制位表示一些状态：

```java
// runState bits: SHUTDOWN must be negative, others arbitrary powers of two
private static final int  RSLOCK     = 1;
private static final int  RSIGNAL    = 1 << 1;
private static final int  STARTED    = 1 << 2;
private static final int  STOP       = 1 << 29;
private static final int  TERMINATED = 1 << 30;
private static final int  SHUTDOWN   = 1 << 31;
```

左侧第一位表示加锁状态，1 为加锁。lockRunState 和 unlockRunState 详细的逻辑见上面的描述。lockRunState 方法利用 runState 的一个二进制位去作为加锁标识，来实现一个全局的锁。加锁失败会进行随机次数的自旋。自旋以后仍然无法获得锁时，就去在 stealCounter 上加锁并阻塞。这里的加锁失败就阻塞是为了避免很多线程尝试加锁，但是无法立刻获得锁时会导致的频繁的自旋，过多的自旋消耗了 CPU 资源。进入阻塞就释放了 CPU 时间。

unlockRunState 解锁的同时还能去更新 runState，ForkJoinPool 所有对 runState 的修改都是通过 unlockRunState 的。

那么这个全局锁是哪些地方用到呢，只要看哪些地方在调用 lockRunState 就知道了，目前看到的有一个是 externalSubmit 方法，就是从外部向 ForkJoinPool 提交任务时，如果 WorkQueues 没创建，或者在随机（通过 ThreadLocalRandom.getProbe 做 hash，参见 SMASK）选取 WorkQueues 上的 queue 时，发现选取的位置上的 queue 为 null。这两种情况一个需要新建 queue 数组，一个需要新建 queue 本身，两者都加了锁。

### ForkJoinTask 的 status

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

### WorkQueue 的 top 和 base 的维护

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

putOrdered** 系列方法类似于 volatile 写。volatile 写会在写前加 StoreStore 屏障，会在写后加 StoreLoad 屏障。putOrdered 只有 StoreStore，没有 StoreLoad。带来的影响是 putOrdered 更新的变量无法立即保证可见性。但是此处为什么能容忍可见性的延迟的呢？   TODO https://www.zhihu.com/question/270824087

### SMASK 是干什么用的

```java
static final int SMASK        = 0xffff;        // short bits == max index
```

SMASK 值为 65535，二进制位 1111111111111111。然后看构造器中对 config 的初始化。

```java
this.config = (parallelism & SMASK) | mode;
```

parallelism & SMASK 是将 parallelism 限制到最大 65535。mode 有两种情况：

```java
static final int LIFO_QUEUE   = 0;
static final int FIFO_QUEUE   = 1 << 16;   // 二进制 10000000000000000
```

parallelism & SMASK 和 0 算或运算其实还是原样，但是和 1 << 16 算或运算得到的结果是相对于 parallelism & SMASK 加 65535。因为前者最大值是 65535，那么从左算起第 17 个二进制位是 0。1 << 16 刚好是从左算起第 17 个二进制位是 1，求或运算就相当于增加 65535。 默认的构造器使用 LIFO_QUEUE，即 config 是小于等于 65535 的。如果用 FIFO_QUEUE，是大于 65535 的。

ForkJoinPool 的 workQueues 的初始容量是这么定的：

```java
// 将config从左侧17位开始的二进制位设为0
int p = config & SMASK; // ensure at least 2 slots
int n = (p > 1) ? p - 1 : 1;
n |= n >>> 1; n |= n >>> 2;  n |= n >>> 4; //(1)
n |= n >>> 8; n |= n >>> 16; n = (n + 1) << 1; //(2)
workQueues = new WorkQueue[n];
```

上述位运算的测试在 [这里（test3方法）](https://github.com/footmanFF/demos/blob/8c57e57c9606c5767c11049994abb92ad6abba0f/jdk-test/src/main/java/com/footmanff/jdktest/bit/BitTest.java)。（1）和（2）两行的目的是为了求大于 n 的最小的 2 的幂。值是 2 的幂的数是最高位为 1，其他位都为 0 的数。如果要求大于一个数的 2 的幂，就是将这个数除了最高位以外所有其他的位设为 1，然后再将这个数 + 1。按照这个思路去看上面的代码就能理解了：

```java
// 因为最高位一定为1，右移1位再与原值求或运算，相当于将最高和次最高位设置为1
n |= n >>> 1;
// 第一步已经将最高和次最高位设置为1，这次可以移动2位再求或
// 一次性将右边第3第4位置设置为1
n |= n >>> 2;  
// 依次类推
n |= n >>> 4;
n |= n >>> 8; 
n |= n >>> 16; 
// 因为入参n其实被限制了最大值为65535(与SMARK做过与运算)
// 所以执行到这里，最高位右侧的二进制位全为1了，再加1就获得大于2的最小的2的幂了
n = (n + 1);
// 最后还会左移1位，相当于乘以2
n = n << 1;
```

结合上述分析，workQueues 的初始容量是大于 p 的最小的 2 的幂的两倍。

### ForkJoinPool 中的锁

