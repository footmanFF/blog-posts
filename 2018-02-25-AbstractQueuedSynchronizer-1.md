---
layout: post
title: AbstractQueuedSynchronizer排他锁分析
date: 2018-02-25
tags: AbstractQueuedSynchronizer
---

ThreadPoolExecutor.Worker 继承自 AbstractQueuedSynchronizer，并实现了一些锁相关的方法：

```java
public void lock()        { acquire(1); }
public boolean tryLock()  { return tryAcquire(1); }
public void unlock()      { release(1); }
public boolean isLocked() { return isHeldExclusively(); }
```

锁的实现：

```java
public final void acquire(int arg) {
    // 利用CAS尝试去更新state，如果失败就要进入等待队列
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

<!-- more -->

tryAcquire 方法通过 unsafe 提供的 CAS 支持实现：

```java
protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}
```

```Java
    protected final boolean compareAndSetState(int expect, int update) {
        // See below for intrinsics setup to support this
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
```

```java
  // 在队列中增加一个节点，即新增一个等待这个锁的线程
	private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        // 如果队尾不为null就去将node添加到队尾
        if (pred != null) {
            node.prev = pred;
            // 如果CAS失败，进入下面的enq，enq用循环保证添加队尾操作会成功
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
```

enq 方法将 node 加进等待队列中，用 CAS + 循环重试去避免并发问题。

```java
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            // 队尾为null，就CAS设置一个新的队尾
            if (t == null) { // Must initialize
                // 成功和失败都会进入下一次循环
                // 失败则在下一次循环中已存队尾后增加节点
                // 成功则在下一次循环中，在新增加节点的后面加上参数中的节点node(此处看下面的详细备注)
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                // 在已存队尾后增加节点
                node.prev = t;
                // 如果CAS失败，进入下一次循环，在最新的队尾增加节点(node)
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

**备注：**如果有 2 个线程在等待锁，那么锁的队列中会有 3 个节点。从头到尾分别是 N1 N2 N3。这里 N1 会是一个空节点，用来表示已经获取锁的线程。从第二个节点开始两个等待线程的节点。上面的代码反映了这点：

```java
// 如果尾巴为null，即队列为空，那么就去原子新增一个空节点(new Node())，并且tail和head都指向这个新建的空节点
Node t = tail;
if (t == null) {
    if (compareAndSetHead(new Node()))
        tail = head;
}
```

然后看 acquireQueued 方法：

```java
  // 对队列中的某个节点，去尝试获取这个节点的排他且不可中断的锁
	final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 获取前置的节点，当前节点可能在等待前置节点释放锁
                final Node p = node.predecessor();
                // tryAcquire(arg)为true代表获得了CAS设置state成功(成功获得锁)
                if (p == head && tryAcquire(arg)) {
                    // 如果node的前置节点是head，那说明node是队列中第一个处于等待中的节点
                    // 队列的首个节点其实是已经获取锁的线程的节点
                    // 如果是上述情况，就去尝试获取锁(调用tryAcquire)，如果获取成功就将node设为head，并返回
                    // 补充：前置节点为head，并且CAS获取state可以成功，意味着当前线程等待的锁已经被释放，可以去获取了
                    // 因为tryAcquire方法CAS更新state，下面的逻辑只会有一个线程进入，因此是线程安全的
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                // 获取锁失败会执行到这里，理论上应该会进入阻塞(猜测->已确定)
                // shouldParkAfterFailedAcquire判断是否需要阻塞线程
                // parkAndCheckInterrupt阻塞，并且返回线程是否中断(在阻塞被唤醒以后)
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

**这里有个疑问：**shouldParkAfterFailedAcquire 对 pred.waitStatus 的判断如何理解

```java
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        // 取消状态，不需要被唤醒
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            // 将被取消的节点从链表中剔除
            do {
                /* 相当于
                 * pred = pred.prev;
                 * node.prev = pred;
                 */
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
            // 当前节点node前面的一个或多个节点被取消，剔除掉这些被取消的节点以后，让方法返回false
            // 让外层调用继续在循环检测是否能获取锁，外层是指acquireQueued方法
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

Node 的 waitStatus 的四个状态：

```java
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
```

**SIGNAL：**标记为 SIGNAL 的节点表示他后面的节点需要被从阻塞中恢复过来（unpack），相当于当前获得线程释放锁或取消锁定，在他之后加锁的线程就可以从阻塞中恢复过来，并获得锁。

**CANCELLED：**节点因为超时或者线程中断而进入的取消状态，线程一点进入取消状态，就不能再从这个状态进入其他状态了。尤其有一点：线程从阻塞进入取消状态就不会再重新进入阻塞状态。

**CONDITION：**TODO

**PROPAGATE：**一次 releaseShared 请求必须传播给其他的节点。PROPAGATE 会在 releaseShared 的执行中被设置，用来保证他的传播，即使有其他操作进行干预。这个是在共享模式下会使用到。并且只会被设置到队列的头部。

waitStatus 是被初始化为 0 的，0 应该代表的是 node 代表的线程处在阻塞状态。

**获取和释放逻辑的伪代码：**

```

Acquire:
     while (!tryAcquire(arg)) {
        enqueue thread if it is not already queued;
        possibly block current thread;
        // 此处被唤醒，如果唤醒后还是不成功就会接着阻塞
     }

Release:
     if (tryRelease(arg))
        unblock the first queued thread;
```

### 接着看释放

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            // h.waitStatus!=0如何理解？
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```

tryRelease 以 ReentrantLock 为例：

```java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            // 当前线程如果和锁定的线程不是同一个线程是要抛出异常的
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

```java
    private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // unpark头结点的下一个节点，即唤醒下一个等待中的线程
        // unpark调用会让上面的parkAndCheckInterrupt方法返回false，具体见LockSupport类
        if (s != null)
            LockSupport.unpark(s.thread);
    }
```

如果有 N 个线程在等待锁，队列中就会有 N + 1 个节点，首个节点是当前获得了锁的线程。每个等待的节点都在 acquireQueued 方法的大循环中，被 parkAndCheckInterrupt 方法阻塞。当前获取锁的线程一旦释放锁，就会去唤醒他的下一个节点（下一个等待线程）。等待线程被唤醒以后又开始执行 acquireQueued 方法的循环，重新检查「自己前面的节点是不是首个节点，并且是不是能够获取锁」，如果通过检查就自己成为头部，并且方法返回，获得锁。

### 引用

[深度解析Java 8：JDK1.8 AbstractQueuedSynchronizer的实现分析（上）](http://www.infoq.com/cn/articles/jdk1.8-abstractqueuedsynchronizer)

[深度解析Java 8：AbstractQueuedSynchronizer的实现分析（下）](http://www.infoq.com/cn/articles/java8-abstractqueuedsynchronizer)

[AbstractQueuedSynchronizer的介绍和原理分析](http://ifeve.com/introduce-abstractqueuedsynchronizer/)



