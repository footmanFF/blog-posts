---
layout: post
title: ThreadPoolExecutor原理
date: 2018-02-26
tags: ThreadPoolExecutor
---

### ThreadPoolExecutor 源码

#### The main pool control state

属性 ctl 用来表示表示他内部的两个状态：工作线程数（workerCount），运行状态（runState）：

```java
/**
 * The main pool control state, ctl, is an atomic integer packing
 * two conceptual fields
 *   workerCount, indicating the effective number of threads
 *   runState,    indicating whether running, shutting down etc
 */
 private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

<!-- more -->

ctlof 方法是执行两个参数的或运算：

```java
private static int ctlOf(int rs, int wc) { 
    return rs | wc; 
}
```

这里 ctl 的初始值是 RUNNING 和 0 进行或运算，RUNNING 的值二进制是：

```Java
// -1左移29位
11100000 00000000 00000000 00000000
```

为什么是 29 位，因为设计者把整形的前 3 位用作 runState，后 29 位用作 workerCount。看看其他几个状态：

```Java
// 11100000 00000000 00000000 00000000
private static final int RUNNING    = -1 << COUNT_BITS;
// 00000000 00000000 00000000 00000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
// 00100000 00000000 00000000 00000000
private static final int STOP       =  1 << COUNT_BITS;
// 01000000 00000000 00000000 00000000
private static final int TIDYING    =  2 << COUNT_BITS;
// 01100000 00000000 00000000 00000000
private static final int TERMINATED =  3 << COUNT_BITS;
```

这几个状态其实 workerCount 位为都为 0，但是 runState 都有值。所以其实上面的 ctlof 方法 rs 参数是 runState，wc 是 workerCount。两者求或运算，刚好是最终合并了这两个状态的 ctl。

然后可以看 execute 方法：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    /*
    * Proceed in 3 steps:
    *
    * 1. If fewer than corePoolSize threads are running, try to
    * start a new thread with the given command as its first
    * task.  The call to addWorker atomically checks runState and
    * workerCount, and so prevents false alarms that would add
    * threads when it shouldn't, by returning false.
    *
    * 2. If a task can be successfully queued, then we still need
    * to double-check whether we should have added a thread
    * (because existing ones died since last checking) or that
    * the pool shut down since entry into this method. So we
    * recheck state and if necessary roll back the enqueuing if
    * stopped, or start a new thread if there are none.
    *
    * 3. If we cannot queue task, then we try to add a new
    * thread.  If it fails, we know we are shut down or saturated
    * and so reject the task.
    */
  
    // 如果当前工作线程数少于corePoolSize就增加工作线程(核心线程)，并且任务直接在新工作线程中跑
    if (workerCountOf(c) < corePoolSize) {
        // 如果增加成功，直接提交运行并返回
        if (addWorker(command, true))
            return;
        // 如果增加失败(失败原因见下文的addWorker方法注释)，就去将任务放进队列中
        c = ctl.get();
    }
    // 如果线程池处于running状态，且任务入队成功
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 重新检查ctl的runState，如果此时线程池已经被关闭，就去从队列删除任务
        // 此处其实没有这段代码应该也可以，加了这段可以有助于在线程池被请求关闭后
        // 可以减少一些新的任务进入队列，算是种优化，可以让线程池尽快退出，不被新的任务拖累
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 判断如果工作线程数为0，就去新增一个非核心线程，任务已经进入队列，就不需要设定初始任务参数了
       	else if (workerCountOf(recheck) == 0)   //  (1)
            addWorker(null, false);
    }
    // 如果线程池不是running状态的，进入addWorker最终也会返回false，一样会拒绝任务
    // 如果任务入队失败(比如对列满了)，就去尝试新增一个非核心线程，并在这个线程上执行入队失败的任务
    // 如果还是失败，证明池满了，那就只能拒绝了
    else if (!addWorker(command, false))
        reject(command);
}
```

**备注：**execute 方法标记 （1）处，可能会个疑问「前面不是刚判断过 workCount 小于核心线程数就去新增一个线程，为什么这里还会存在工作线程数为 0 的情况」。这里是因为线程池中线程时刻都在消费队列中的任务，如果队列中的任务消费空了，初始化的时候有设置线程池的核心线程是可以被回收的，那么就有可能在任务队列为空的时候导致所有线程都被关闭的情况。这里判断如果工作线程数为 0，就去新增一个工作线程，**保证在增加任务以后线程池中有线程可用**。

workerCountOf 方法和 CAPACITY 变量：

```Java
// 00011111 11111111 11111111 11111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
private static int workerCountOf(int c)  {
    return c & CAPACITY;
}
```

CAPACITY 最高位都为 0，和 ctl 的值执行且运算，刚好可以得到 ctl 中的 workCount 部分的值。

**addWorker：**

```java
/**
 * 增加一个工作线程，如果线程池被stop或者shutdown返回false
 * 如果线程数达到限制返回false，线程创建失败返回false
 * @param firstTask 初始执行任务，可以不为null
 * @param core 是否创建核心线程
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && 
            !(rs == SHUTDOWN && firstTask == null && !workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            // 工作线程数超过容量就直接返回失败
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 利用AtomicInteger去尝试给ctl中的线程数+1，如果成功就退出retry循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // compareAndIncrementWorkerCount 失败就一定是有其他线程增加或减少了工作线程数
            // 这里重新获得ctl的值，判断运行状态是不是有变化，如果有变化重新执行retry循环
            // 如果没变化，重新利用AtomicInteger的CAS机制，尝试去给工作线程数+1
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }
    // 以上其实是利用了AtomicInteger实现并发下，无锁递增ctl中的workCount
    // 只要AtomicInteger的CAS执行成功，就能跳出retry循环，否则就无限尝试，或者runState状态变换就
    // 重新检测runState来判断是否继续执行

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 在mainLock上加锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                // 在线程池为running或者shutdown的时候才去增加工作线程，后者还要求任务对象firstTask为null
                // 因为shutDown状态是不允许新的任务进队列，但是允许队列内还留存未处理任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 线程的创建是交给ThreadFactory的，ThreadFactory可能由用户提供，这里检查下线程的状态
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 出现异常，或者线程创建出问题，或导致workerStarted为false
        if (!workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

**addWorkerFailed：**

```java
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);
        // 利用AtomicInteger，执行并发下无锁递减
        decrementWorkerCount();
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

**tryTerminate：**

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;
        if (workerCountOf(c) != 0) { // Eligible to terminate
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

**interruptIdleWorkers：**

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 循环所有worker并去中断每个worker
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

Worker.tryLock 这里加的是什么锁，为什么要加锁？Worker 类继承自 [AbstractQueuedSynchronizer](https://docs.oracle.com/javase/7/docs/api/)？见 [AbstractQueuedSynchronizer排他锁分析](http://footmanff.com/2018/02/25/2018-02-25-AbstractQueuedSynchronizer-1/)

```java
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if (
                    // 这里有疑问，第二个runStateAtLeast是为啥?
                    (
                        runStateAtLeast(ctl.get(), STOP) || 
                        (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))
                    ) 
                    &&
                    !wt.isInterrupted()
                ) {
                    wt.interrupt();
                }
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            // completedAbruptly为true即发生了异常, abruptly：突然的
            // 工作线程结束，后续可能要重启线程，或者去执行一些工作线程关闭相关的逻辑，这部分在processWorkerExit处理
            processWorkerExit(w, completedAbruptly);
        }
    }
```

getTask：

```java
  // 返回Null代表工作线程需要结束掉
	private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;  // (1)

            // 看看是不是线程已经过剩（线程数超过maximumPoolSize）了，如果是就把线程关闭掉
            // 如果工作线程数超过最大线程数或者从队列获取任务超时
            if ((wc > maximumPoolSize || (timed && timedOut))   //(2)
                // 如果任务队列为空
                && (wc > 1 || workQueue.isEmpty())) {
                // 满足上述意味着当前工作线程已经是多于的了：既然连任务都没有，而且线程数也过多(超过最大线程数)
                // 然后就去递减工作线程数，并返回Null(意味着停止当前工作线程)
                if (compareAndDecrementWorkerCount(c))
                    return null;
                // 如果CAS递减失败，意味着有其他线程在并发的递减workCount
                // 进入下一次循环，重新尝试递减
                continue;
            }

            try {
                // 从队列获取一个任务
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                // 返回null就意味着超时了，如果调用的是take()是会一直等待的，不会超时
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }
```

**备注：**getTask 方法中标注的 (1) 和 (2) 的 timed 变量，控制的是是否空闲线程（IDLE）会被回收，如果设置了 allowCoreThreadTimeOut 为 true，那么所有线程都可能会被回收，如果 allowCoreThreadTimeOut 为 false，那就只有核心线程以外的线程会被回收。回收还是通过 getTask 返回 null，然后由外层的循环控制工作线程的退出（回收）。

如果当前池里的线程全是核心线程，且队列中的任务为空，那么 getTask 会一直空转，每次循环在 workQueue.poll 处等待 keepAliveTime 纳秒时间，等待完就继续进行下一次循环，直到任务队列（workQueue）中有新的任务进来，poll 会返回任务，整个方法会退出。

processWorkerExit：

```java
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
		// completedAbruptly为true代表意外结束，工作线程意外结束时需要取维护ctl中的workCount的值
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        // shutdown或者running时，这两个状态下队列内是可能有任务的，所以是需要继续维持工作线程的存在
        // 从而让队列内的任务可以被消费处理掉
        if (runStateLessThan(c, STOP)) {
            // 在非意外结束情况下
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                // 如果当前线程数小于最小线程限制，是需要去重建工作线程的，见下面的addWorker
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            // 意外结束情况下，直接重启工作线程
            addWorker(null, false);
        }
    }
```

#### 线程状态表

| runState   | value            | desc                                     |
| ---------- | ---------------- | ---------------------------------------- |
| RUNNING    | -1 << COUNT_BITS | 接受新任务，也处理队列中的任务           |
| SHUTDOWN   | 0 << COUNT_BITS  | 关闭，接受新任务，但是不处理队列中的任务 |
| STOP       | 1 << COUNT_BITS  | 停止，不接受新任务，不处理队列中的任务   |
| TIDYING    | 2 << COUNT_BITS  | 整理中，所有任务已经终止，工作线程数为0  |
| TERMINATED | 3 << COUNT_BITS  | terminated() 被调用                      |

#### Future 的实现解析

FutureTask 的 get  方法

```java
    public V get() throws InterruptedException, ExecutionException {
        int s = state;
        if (s <= COMPLETING)
            s = awaitDone(false, 0L);
        return report(s);
    }
```

state：

```java
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
```

awaitDone 方法：

```java
    private int awaitDone(boolean timed, long nanos)
        throws InterruptedException {
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        WaitNode q = null;
        boolean queued = false;
        for (;;) {
            if (Thread.interrupted()) {
                removeWaiter(q);
                throw new InterruptedException();
            }

            int s = state;
            if (s > COMPLETING) {
                if (q != null)
                    q.thread = null;
                return s;
            }
            else if (s == COMPLETING) // cannot time out yet
                Thread.yield();
            else if (q == null)
                q = new WaitNode();
            else if (!queued)
                queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                     q.next = waiters, q);
            else if (timed) {
                nanos = deadline - System.nanoTime();
                if (nanos <= 0L) {
                    removeWaiter(q);
                    return state;
                }
                LockSupport.parkNanos(this, nanos);
            }
            else
                LockSupport.park(this);
        }
    }
```

```java
    /**
     * Removes and signals all waiting threads, invokes done(), and
     * nulls out callable.
     */
    private void finishCompletion() {
        // assert state > COMPLETING;
        for (WaitNode q; (q = waiters) != null;) {
            if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
                for (;;) {
                    Thread t = q.thread;
                    if (t != null) {
                        q.thread = null;
                        LockSupport.unpark(t);
                    }
                    WaitNode next = q.next;
                    if (next == null)
                        break;
                    q.next = null; // unlink to help gc
                    q = next;
                }
                break;
            }
        }

        done();

        callable = null;        // to reduce footprint
    }
```

awaitDone 会将当前等待队列推进一个链表 WaitNode。最终会将线程进入阻塞，通过 LockSupport.park。唤醒是通过 finishCompletion 方法，任务执行完以后，无论是正常结束还是异常都会调这个方法。finishCompletion 回去唤醒这些方法。

WaitNode 的结构：

```java
    static final class WaitNode {
        volatile Thread thread;
        volatile WaitNode next;
        WaitNode() { thread = Thread.currentThread(); }
    }
```

待续 … 

### 引用

[sum.msic.Unsafe](http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/#concurrency)

[AQS](http://ifeve.com/aqs-2/)

[AbstractQueuedSynchronizer排他锁分析](http://footmanff.com/2018/02/25/2018-02-25-AbstractQueuedSynchronizer-1/)

https://zhuanlan.zhihu.com/p/123328822