---
layout: post
title: 1.8版本的ConcurrentHashMap分析
date: 2018-03-13
tags: ConcurrentHashMap
---

## 先看看文档

[ConcurrentHashMap jdk 1.8](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html) 

> Retrievals reflect the results of the most recently *completed* update operations holding upon their onset. (More formally, an update operation for a given key bears a *happens-before* relation with any (non-null) retrieval for that key reporting the updated value.) 

happens-before 是重点。

<!-- more -->

#### 内部的注释

```
* We use the top (sign) bit of Node hash fields for control
* purposes -- it is available anyway because of addressing
* constraints.  Nodes with negative hash fields are specially
* handled or ignored in map methods.
```

Node 节点的整形的 Hash 属性，二进制的高几位用来作为状态使用。hash 为负的 Node 表示 Node 需要被特殊处理。负数的状态：

```java
static final int MOVED     = -1; // hash for forwarding nodes
static final int TREEBIN   = -2; // hash for roots of trees
static final int RESERVED  = -3; // hash for transient reservations
```

```
* Maintaining API and serialization compatibility with previous
* versions of this class introduces several oddities. Mainly: We
* leave untouched but unused constructor arguments refering to
* concurrencyLevel. We accept a loadFactor constructor argument,
* but apply it only to initial table capacity (which is the only
* time that we can guarantee to honor it.) We also declare an
* unused "Segment" class that is instantiated in minimal form
* only when serializing.
```

构造器里的 concurrencyLevel 和 loadFactor 没啥用了，后者只是用来初始化最初的 table。

## 还是看代码吧

### 构造器

```java
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

- tableSizeFor 是取大于或等于 size 的最小的 2 的乘方（此处不完全确定），但是返回结果不会大于 MAXIMUM_CAPACITY。算法见《Hackers Delight - 算法心得：高效算法的奥秘》 3.2 节，回头有时间去看看真本书。
- HashMap 的装载因子解释 [What is the significance of load factor in HashMap?](https://stackoverflow.com/questions/10901752/what-is-the-significance-of-load-factor-in-hashmap)
- concurrencyLevel 没啥用了，见上面的注释。
- loadFactor 用来计算初始化 table 大小，元素容量除以 loadFactor 可以得出 rehash 的阀值，首次初始化的 table 大小大于这个阀值，以后 loadFactor 都用不到了。

```java
public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}
```

- 初始化的 table 的容量是大于或等于 (ic + ic / 2 + 1) 的最小的 2 的乘方。

### put

##### spread

```java
static final int spread(int h) {
    // HASH_BITS二进制：1111111111111111111111111111111
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

spread 的用处：[Java HashMap工作原理及实现](https://yikun.github.io/2015/04/01/Java-HashMap%E5%B7%A5%E4%BD%9C%E5%8E%9F%E7%90%86%E5%8F%8A%E5%AE%9E%E7%8E%B0/)

##### initTable

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // lost initialization race; just spin
        // CAS设置sizeCtl为-1(table初始化中)
        else if (U.compareAndSetInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    // 去初始化table，大小为sizeCtl
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 将sizeCtl设置n-(n>>>2)
                    // 为什么设置成这个数???
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

- sizeCtl 属性：负数表示 table 正在初始化或者扩容，-1 表示正在初始化，- (1 + 扩容线程数) 表示正在扩容。当 table 为 null 时，sizeCtl 为初始化时创立的 table 长度。初始化以后，sizeCtl 的值为 next element count value upon which to resize the table（**TODO**）。此处 element count 看起来是 k-v 数。
- initTable 方法无锁实现 table 初始化。初始化 table 的操作放在临界区内，临界区由 CAS 设置 sizeCtl 保证。临界区内的逻辑会设置 table 为新的非空的数组，并且 table 是 volatile 的，其他在临界区外的线程会最终全部退出循环，从循环下面的 return 返回。
- 未进入临界区的线程不会疯狂循环消耗 CPU，通过调用 Thread.yield() 释放 CPU 时间。

##### tabAt

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

ABASE 是 Node [ ] 类型的在内存中的基础偏移，通过调用 unsafe.arrayBaseOffset 获得。ASHIFT 是 Node[] 间隔元素之间内存偏移数的二进制位数，计算方法如下：

```java
int scale = unsafe.arrayIndexScale(arrayClass);
// numberOfLeadingZeros 方法取除了符号位，从左侧开始二进制位为0的位的个数
int ashift = 31 - Integer.numberOfLeadingZeros(scale);
```

计算数组中一个元素的内存偏移直接理解应该是：基础偏移 + ( 下标索引 * 元素偏移量 )。但是因为数组元素的内存偏移都是 2 的指数，所以这个计算可以等价于：基础偏移 + 下标索引向左偏移元素偏移量的位数，等价于执行乘法。这里是一个优化，位运算的效率比做乘法要好。这里能这么做也是因为元素偏移量是 2 的指数，一个数乘以一个 2 的指数，相当于向左偏移乘数的位数。[测试代码](https://github.com/footmanFF/demos/blob/f4e9951fe6770b5b08d7e43241d8145b6545707f/jdk-test/src/main/java/com/footmanff/jdktest/concurrent/UnsafeTest.java)

##### putVal

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        // i: hash码在table表上的落点(数组下标)
        // n: table表的长度
        // fh: 局部变量，用来存Node的属性hash 
        // f: hash最终所定位到Node
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 桶不存在时，去创建一个，并原子更新进table，失败的话会继续循环
            // 去当做桶已经存在的情况处理
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        } else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            // 在桶上加锁
            synchronized (f) {
                // 进入锁以后重新去判断桶是否还在原位置
                // 扩容可能会改变桶的位置。如果此处为false，继续循环重新在扩容以后的table上处理。
                if (tabAt(tab, i) == f) {
                    // Node的hash大于0，表示是正常的数据节点
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 从链表上查找hash和key都为一致的node
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // hash或key不一致则要去遍历链表查找
                            // 如果遍历了链表都不存在，那么创建一个新节点
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    // hash小于0时，有一种状态是TREEBIN，判断出来做特殊处理
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 桶内的节点大于阈值以后，将链表转化成树
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

##### addCount

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        // counterCells是多个用数组容纳的计数器
        // 这里在counterCells数组中有可用计数器时，去尝试给计数器递增x
        // 如果失败或者没有计数器可用，执行fullAddCount
        if (as == null || (m = as.length - 1) < 0 ||
            // ThreadLocalRandom.getProbe()用来取随机数 TODO
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    }
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

- ThreadLocalRandom 干啥的，sun.misc.Contended 是干啥的，cache line 又是啥？[what-is-false-sharing](http://robsjava.blogspot.hk/2014/03/what-is-false-sharing.html) ， [Oracle JDK8在JVM上的改进有哪些意义？ - RednaxelaFX的回答 - 知乎](https://www.zhihu.com/question/30538696/answer/48464003)
- addCount 和 fullAddCount 用到了 ThreadLocalRandom，待分析
- ThreadLocalRandom.getProbe() 获取的是啥？ThreadLocalRandom.advanceProbe(h) 又是啥？[JAVA THREADLOCALRANDOM EXPLAINED](http://alvaro-videla.com/2016/10/inside-java-s-threadlocalrandom.html) ， [Xorshift](https://en.wikipedia.org/wiki/Xorshift)
- ThreadLocalRandom.getProbe() & m 的数学含义是啥？ 用一个数去和另一个数求或运算的意义是啥？

##### fullAddCount

在 CAS 递增 baseCount 时，遇到竞争时，使用 fullAddCount 去在多个统计变量上去做递增，这个多个统计变量就是 counterCells，这个数组的每一项是一个 CounterCell，是一个统计。

```java
// x: 递增数量
// wasUncontended: 是否未竞争CAS递增过CounterCell
private final void fullAddCount(long x, boolean wasUncontended) {
    int h;
    if ((h = ThreadLocalRandom.getProbe()) == 0) {
        ThreadLocalRandom.localInit();      // force initialization
        h = ThreadLocalRandom.getProbe();
        wasUncontended = true;
    }
    boolean collide = false;                // True if last slot nonempty
    for (;;) {
        CounterCell[] as; CounterCell a; int n; long v;
        if ((as = counterCells) != null && (n = as.length) > 0) {
            if ((a = as[(n - 1) & h]) == null) {
                if (cellsBusy == 0) {            // Try to attach new Cell
                    CounterCell r = new CounterCell(x); // Optimistic create
                    if (cellsBusy == 0 &&
                        U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                        boolean created = false;
                        try {               // Recheck under lock
                            // 去在counterCells上创建新的CounterCell
                            CounterCell[] rs; int m, j;
                            if ((rs = counterCells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {  // 双重检测
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0;
                        }
                        if (created)
                            // 直接初始化一个统计变量，而不是去找一个统计变量递增，所以可以直接返回
                            break;
                        // 双重检测失败，证明其他线程在counterCells同一个位置建立了CounterCell
                        continue;           // Slot is now non-empty
                    }
                }
                collide = false;
            }
            else if (!wasUncontended)       // CAS already known to fail
                // 前一个if内的逻辑是去竞争递增一个CounterCell，所以此处可以将wasUncontended设置为true
                // 此处相当于竞争过一次了，就去换一个CounterCell去尝试递增，见下面的ThreadLocalRandom.advanceProbe
                wasUncontended = true;      // Continue after rehash
            else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                break;
            else if (counterCells != as || n >= NCPU)
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true;
            else if (cellsBusy == 0 &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                try {
                    if (counterCells == as) {// Expand table unless stale
                        CounterCell[] rs = new CounterCell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        counterCells = rs;
                    }
                } finally {
                    cellsBusy = 0;
                }
                collide = false;
                continue;                   // Retry with expanded table
            }
            h = ThreadLocalRandom.advanceProbe(h);
        }
        else if (cellsBusy == 0 && counterCells == as &&
                 /* [A] */
                 U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
            	 /* [B1] */
            boolean init = false;
            try {                           // Initialize table
                if (counterCells == as) {   // 此处需要判空的原因见下面第三点
                    CounterCell[] rs = new CounterCell[2];
                    rs[h & 1] = new CounterCell(x);
                    counterCells = rs;
                    /* [B2] */
                    init = true;
                }
            } finally {
                cellsBusy = 0;    // finally回复cellsBusy为0
            }
            if (init)      // 首次进入时counterCells为null，只要初始化了就可以直接中断循环返回
                break;
        }
        // 在counterCells竞争递增失败的时候，去尝试递增baseCount，如果成功就直接可以返回
        else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
            break;                          // Fall back on using base
    }
}
```

- cellsBusy：1 表示 CounterCells 正在创建 ，0 表示其他状态。
- baseCount：用于统计 Map 中的 k-v 数，baseCount 用于在没有出现竞争的情况下统计。
- counterCells：数一个数组，每个数组项是一个 int，在 baseCount 上的递增出现竞争时会去取 counterCells 中的一个项进行递增，最终的 Map 中的 k-v 数总和是 baseCount 和 counterCells 所有计数之和，见下面的 subCount 方法。
- 三个地方都回去 CAS set cellsBusy，从 0 改成 1，并发下只有一个线程能进入临界区代码。临界区代码用 try finally 去保证 cellsBusy 最终一定会被设置回 0，相当于解锁。
- 如果 A 线程运行到「A」，另 B 线程运行到「B1」，如果 A 线程被挂起（比如 CPU 切换了执行线程），然而 B 线程继续执行，一直执行到了「B2」，这个时候 A 线程继续执行，如果没有 counterCells == as 判断，实惠重复创建 counterCells 的。这个是需要 counterCells 判断的理由。这个和并发下的单例设计模式一样，在进入锁以后需要重新判空一次。
- 这里的计数是使用了 counterCells，从源码的注释看，这个实现思路是和 java.util.concurrent.atomic.LongAdder 一致的，可以去研究下。
- fullAddCount 方法的实现思路几乎和 [LongAdder](http://footmanff.com/2018/03/21/2018-03-21-LongAdder-1/) 一致。

##### sumCount

```java
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

counterCells 和 baseCount 见上面的解释。

## 资料

https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html