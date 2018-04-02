---
layout: post
title: volatile理解
date: 2017-03-14
tags: Java
---
## 看一段代码
```java
class VolatileExample {
  int a = 0;
  volatile boolean flag = false;
  public void writer() {
      a = 1;                   //1
      flag = true;               //2
  }
  public void reader() {
      if (flag) {                //3
          int i =  a;           //4
      }
  }
}
```
volatile的作用是让一个变量是多线程可见的，并且对他的 get 和 set 是原子的。着重考虑前一点，后一点好理解。

<!-- more -->


那么有个问题。在上面的代码里，如果线程 A 执行 writer，线程 B 并行执行 reader，变量 flag 因为有 volatile 所以是多线程可见的，但是变量 a 是可见的吗？有没有可能，reader 方法最终 i 被设置为 0，因为没有 a 变量不是「可见的」？。

这个文章有讲这个问题 <http://www.infoq.com/cn/articles/java-memory-model-4>

>当写一个 volatile 变量时，JMM 会**把该线程对应的本地内存中的共享变量刷新到主内存**。
>当读一个 volatile 变量时，JMM 会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

如上说法，那么应该是线程 A 在写 flag 时，同时让当前线程调用的方法所能使用到的变量（局部变量、类变量）都刷到主内存。当 B 读 flag 时，同时让当前线程调用的方法所能使用到的变量（局部变量、类变量）在本地内存失效，重新去主内存中读取。
以上就等价于让所有的变量都变成了多线程可见的，就如 volatile 是多线程可见的一样。

>下面对 volatile 写和 volatile 读的内存语义做个总结：
>- 线程 A 写一个 volatile 变量，实质上是线程 A 向接下来将要读这个 volatile 变量的某个线程发出了（其对共享变量所在修改的）消息。
>- 线程 B 读一个volatile变量，实质上是线程 B 接收了之前某个线程发出的（在写这个volatile变量之前对共享变量所做修改的）消息。
>- 线程 A 写一个 volatile 变量，随后线程 B 读这个 volatile 变量，这个过程实质上是线程 A 通过主内存向线程 B 发送消息。

看起来就是那么回事，volatile 的作用的可见性更体现在：volatile 的写和读取发出了一个「通知 - 接受」机制，通知其他线程是否需要重新 reload 当前线程的本地内存。

另外一点，volatile 变量是不能做递增的：

> 虽然 volatile 变量很方便，但也存在一些局限性。volatile 变量通常用做某个操作完成、发生中断或者状态的标志，例如程序清单 3-4 中的 asleep 标志。尽管 volatile 变量也可以用于表示其他的状态信息，但在使用时要非常小心。例如；volatile 的语义不足以确保递增操作（count++）的原子性，除非你能确保只有一个线程对变量执行写操作。（原子变量提供了“读一改一写”的原子操作，并且常常用做一种“更好的 volatile 变量”。请参见第 15 章）。
>
> 《深入理解 Java 虚拟机》

------

2018 - 03 - 12 更新：

## JMM 的一些理解

> Java 线程之间的通信由 Java 内存模型（本文简称为 JMM）控制，JMM 决定一个 线程对共享变量的写入何时对另一个线程可见。从抽象的角度来看，JMM 定义了 线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（main memory）中，每个线程都有一个私有的本地内存（local memory），本地内存中存储了该线程以读/写共享变量的副本。**本地内存是 JMM 的一个抽象概念，并不真实存在。它涵盖了缓存，写缓冲区，寄存器以及其他的硬件和编译器优化**。
>
> JMM 属于语言级的内存模型，它确保在不同的编译器和不同的处理器平台之上， 通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见 性保证。
>
> 引用自《Java内存模型》

内存屏障到底什么，起什么作用，如何起作用？

### 这些是啥：load-load load-store store-store store-load

> Load（载人）：作用于工作内存的变量，它把 read 操作从主内存中得到的变量值放人工作内存的变量副本中。
>
> Store（存储）：作用于工作内存的变量，它把工作内存中一个变量的值传送到主内存中，以便随后的 write 操作使用。
>
> 引用自《深入Java虚拟机》- 周志明

load 和 store 似乎是字节码指令。那么如何理解他们，为什么需要他们，他们起了什么作用？

### 所谓的内存屏障是什么

「屏障」是用来隔离的，比如洗澡的时候需要有帘子（如果没有墙），屏障让别人看不到自己洗澡的样子。内存屏障隔离的是指令的执行。比如在一段指令的中间增加了内存屏障，这个时候在屏障之前，所有的 load 都保证已经从主存取回数据，所有的 store 都保证已经写回主存。

编译器和处理器会对指令进行重排序，这会影响内存的访问顺序。CPU、CPU cache、主存之间的缓存一致性也会影响内存的访问顺序。当多核心环境下，并且是多线程时，以上两点是不能被接受的，具体例子网上很多就不多举例了。在指令之间增加内存屏障可以限制屏障前后的乱序的内存访问顺序。首先指令执行顺序不能重排序了，另外内存屏障能规避 CPU 缓存一致性带来的问题，但是做到了上面两点，就解决了问题。

load-load 等等这些个内存屏障的理解比较头疼，可以看看这个：http://ifeve.com/memory-barriers-or-fences/

| 屏障类型   | 描述     |
| ---------- | -------- |
| store-load | TODO ... |
| load-load  | TODO ... |

内存屏障的资料：http://b2e699b3.wiz03.com/share/s/2OVFCP1_wkXs20LtbT1nXNrj0EqwFC1zZAjT2bCeRi3Tzco2

### happens-before

> 要想保证执行操作 B 的线程看到操作 A 的结果（无论 A 和 B 是否在同一个线程中执行），那么在 A 和 B 之间必须满足 Happens-Before 关系。如果两个操作之间缺乏 Happens-Before 关系，那么 JVM 可以对它们**任意地重排序**。
>
> 《Java 并发编程实践》第 16 章

Happens-Before 重点在于，只有满足了 Happens-Before，那么无论谁在前，两个线程中执行的命令无法进行重排序。当无法重排序，在后面的执行的线程都能看到前面执行线程的结果，即前面执行的线程的结果对后面执行的线程可见。

关于操作的顺序，并不是一定要求一个在前一个在后：

> 注意，两个操作之间具有 happens-before 关系，并不意味着前一个操作必须要在后一个操作之前执行！happens-before 仅仅要求前一个操作（执行的结果）对后 一个操作可见，且前一个操作按顺序排在第二个操作之前（the first is visible to and ordered before the second）
>
> 《深入 Java  内存模型》 — 程晓明

那么在何时会产生 happens-before 现象？

> - 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
> - 监视器锁规则：对一个监视器的解锁，happens-before 于随后对这个监视器的加锁。
> - volatile 变量规则：对一个 volatile 域的写，happens- before 于任意后续对这个 volatile 域的读。
> - 传递性：如果 A happens- before B，且 B happens- before C，那么 A happens- before C。
>
> 《深入 Java  内存模型》 — 程晓明

以上第三条可以解释 VolatileExample 这个例子，为何只有 flag 是 volatile 的，但是变量 a 也是可见的。

《Java 并发编程实践》第 16 章 （reading），FutureTask 有一种利用了 happens-before 的并发控制方法。另外，FutureTask 类在 1.6 和 1.7+ 有了变化，前者不再依赖 AQS。如下：

> 由于 Happens-Before 的排序功能很强大，因此有时候可以“借助（Piggyback）”现有同步机制的可见性属性。这需要将 Happens-Before 的程序顺序规则与其他某个顺序规则（通常是监视器锁规则或者 volatile 变量规则）结合起来，从而对某个未被锁保护的变量的访问操作进行排序。这项技术由于对语句的顺序非常敏感，因此很容易出错。它是一项高级技术，并且只有当需要最大限度地提升某些类（例如 ReentrantLock）的性能时，才应该使用这项技术。
>
> 在 FutureTask 的保护方法 AbstractQueuedSynchronizer 中说明了如何使用这种“借助”技术。AQS 维护了一个表示同步器状态的整数，FutureTask 用这个整数来保存任务的状态：正在运行，已完成和已取消。但 FutureTask 还维护了其他一些变量，例如计算的结果。当一个线程调用 set 来保存结果并且另一个线程调用 get 来获取该结果时，这两个线程最好按照 Happens- Before 进行排序。这可以通过将执行结果的引用声明为 volatile 类型来实现，但利用现有的同步机制可以更容易地实现相同的功能。
>
> FutureTask 在设计时能够确保，在调用 tryAcquireShared 之前总能成功地调用 tryRelease- Shared。tryReleaseShared 会写人一个 volatile 类型的变量，而 tryAcquireShared 将读取这个变量。程序清单 16-2 给出了。**InnerSet 和 innerGet 等方法，在保存和获取 result 时将调用这些方法。由于 innerSet 将在调用 releaseShared（这又将调用 tryReleaseShared）之前写入 result，并且 innerGet 将在调用 acquireShared（这又将调用 tryReleaseShared）之后读取 result，因此将程序顺序规则与 volatile 变量规则结合在一起，就可以确保 innerSet 中的写人操作在 innerGet 中的读取操作之前执行。**
>
> 《Java 并发编程实践》第 16 章

上述黑体部分的伪代码：

```
// 线程 1
result = v
releaseShared()        // 写入一个volatile变量

// 线程 2
acquireShared()        // 读取一个volatile变量
var r = result
```

result 是共享变量，且没有 volatile 标注，线程 1 对 result 的写入还是对线程 2 对 result 的读取还是 happens - before 的，因为两个线程的操作中涉及了一个共享的 volatile 变量的「存」与「取」。就像上面的 VolatileExample 例子一样。

```java
void innerSet(V v) {
    for (;;) {
    	int s = getState();
    	if (s == RAN)
    	    return;
        if (s == CANCELLED) {
    	    releaseShared(0);
            return;
        }
    	if (compareAndSetState(s, RAN)) {
            result = v;
            releaseShared(0);
            done();
    	    return;
        }
    }
}

V innerGet() throws InterruptedException, ExecutionException {
    acquireSharedInterruptibly(0);
    if (getState() == CANCELLED)
        throw new CancellationException();
    if (exception != null)
        throw new ExecutionException(exception);
    return result;
}
```

> **之所以将这项技术称为“借助”，是因为它使用了一种现有的 Happens-Before 顺序来确保对象 x 的可见性，而不是专门为了发布 X 而创建一种 Happens-Before 顺序。**
>
> 在 FutureTask 中使用的“借助”技术很容易出错，因此要谨慎使用。但在某些情况下，这种“借助”技术是非常合理的。例如，当某个类在其规范中规定它的各个方法之间必须遵循一种 Happens-Before 关系，基于 BlockingQueue 实现的安全发布就是一种“借助”。如果一个线程将对象置入队列并且另一个线程随后获取这个对象，那么这就是一种安全发布，因为在 BlockingQueue 的实现中包含有足够的内部同步来确保入列操作在出列操作之前执行。
>
> 在类库中提供的其他 Happens-Before 排序包括：
>
> - 将一个元素放人一个线程安全容器的操作将在另一个线程从该容器中获得这个元素的操作之前执行。
> - 在 CountDownLatch 上的倒数操作将在线程从闭锁上的 await 方法中返回之前执行。。释放 Semaphore 许可的操作将在从该 Semaphore 上获得一个许可之前执行。
> - Future 表示的任务的所有操作将在从 Future. Get 中返回之前执行。
> - 向 Executor 提交一个 Runnable 或 Callable 的操作将在任务开始执行之前执行。
> - 一个线程到达 CyclicBarrier 或 Exchanger 的操作将在其他到达该栅栏或交换点的线程被释放之前执行。如果 CyclicBarrier 使用一个栅栏操作，那么到达栅栏的操作将在栅栏操作之前执行，而栅栏操作又会在线程从栅栏中释放之前执行。
>
> 《Java 并发编程实践》第 16.1 节

《Java 并发编程实践》第 16.2 节，很值得读。另外，golang 是如何线程间通信的？GuardedBy 注解是个啥？

#### 为啥没有volatile的双重检查单例模式是不对的

```java
public class Singleton{
    private /*volatile*/ static Singleton singleton = null;
    private Singleton() {}
    public static Singleton getInstance()   {
        if (singleton== null)  {
            synchronized (Singleton.class) {
                if (singleton== null)  {
                    singleton= new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

- singleton= new Singleton() 不是原子的，指令重排时会导致方法返回的 Singleton 引用是未完成赋值的。详细见：[深入浅出单实例SINGLETON设计模式](https://coolshell.cn/articles/265.html)
- 这个实现 涉及了对象的发布，在没有做同步的情况下（没有 happens - before）的情况下，发布的 Singleton 对象引用和他属性引用的对象可能没有刷新的主存，要么 singleton 为 null，要么 Singleton 对象内部的属性为 null，即发布了一个处于错误状态的 Singleton 对象。

解决办法当然是增加 volatile。对他的写入和读取有 happens - before 语义，能防止指令的重排序。另外他能保证多线程情况下的内存可见性。

#### volatile 和重排序

最上面给出了 volatile 变量的写入和读取时主存和内存上的变化。要保证 volatile 建立的 happens-before 关系，还需要在线程内的指令执行序列中增加一些屏障，以解决 volatile 变量读写和普通变量读写在重排序时导致的问题。

![](http://note-1255449501.file.myqcloud.com/2018-03-14-065531.png)

> 从上表我们可以看出：
>
> - 当第二个操作是 volatile 写时，不管第一个操作时什么，都不能重排序。这个规则确保 volatile 写之前的操作不会被编译器重排序到 volatile 写之后。
> - 当第一个操作是 volatile 读时，不管第二个操作是什么，都不能重排序。这个规则确保 volatile 读之后的操作不会被编译器重排序到 volatile 读之前。
> - 当第一个操作是 volatile 写，第二个操作是 volatile 读时，不能重排序。
>
> 《Java 内存模型》程晓明

# 一些资料

[Java内存模型Cookbook-前言](http://ifeve.com/jmm-cookbook/)
[JMM Cookbook(一)指令重排](http://ifeve.com/jmm-cookbook-reorderings/)
[Java内存模型Cookbook（二）内存屏障](http://ifeve.com/jmm-cookbook-mb/)
[The JSR-133 Cookbook for Compiler Writers](http://gee.cs.oswego.edu/dl/jmm/cookbook.html)
The Java Language Specification, Java SE 8 Edition 17.4 Memory Model
https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/package-summary.html
https://www.kernel.org/doc/Documentation/memory-barriers.txt
《Java 并发编程实践》第 16 章
https://coolshell.cn/articles/265.html