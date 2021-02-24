---
layout: post
title: 在Regin间对象相互引用时G1如何完成Root-Tracing
date: 2017-03-01
tags: JVM
---

### 在Regin间对象相互引用时G1如何完成Root-Tracing

假如在对一个 Region 进行 GC，称他为 S1，上面有一个对象 O1 需要进行可达性分析。 如果没有其他 Region 上的对象引用 O1，那么皆大欢喜， 只在 S1 上就能确定 O1 是否可达（不可达就是需要清理）。如果有其他 Region 上的对象引用 O1，如果还不只一个，那么处理起来就比较麻烦了，笨办法就是去遍历，这当然不是一个好办法。

<!-- more -->

Remembered set 就是处理这个问题的。又假如有个 Region 叫 R2，上面有一个对象叫 O2，O2 有对 O1 的应用。G1 收集器的做法是当把 O2 对象实例上的一个对象引用指向 O1 时，产生一次 writer barrier （我的理解是一次拦截动作），这个拦截动作做的是检查应用所在对象也就是 O2 所属的 Region 是否与被引用对象（也就是 O1）所在的 Region 是否是同一个 Region。如果是同一个，啥也不做。如果不是，就去把这个引用关系（O2 有引用 O1）记录到 O1 的 Region 上（也就是 R1）。在任何设置对象引用的时候都如上述处理，那么最终的结果是，一个 Region 上的所有对象要么没被其他 Region 的对象引用，如果有被引用的就一定能在这个 Region 的 Remembered set 中找到其引用关系。

有了些引用关系能做啥呢？回到上面的例子，如果对 O1 做 Root tracing，只要把 Remembered set 里的外部引用指针全部加入到 Root 集合中，那么利用这个 Root  集合就能完成这个 Region 上所有对象的 Root tracing（可达性分析），即能判断所有对象是否是需要收集，不再去别的 Region 上遍历查看时有有对象应用这个 Regin 上的对象了。效率提升是杠杠的。

### 2021-02-24 更新

Remembered set 一般可以用卡表（CardTable）来实现，CardTable 是一个 bit 数组，每一位标记一个范围内存上的对象是否有针对当前 GC 区域的对应引用，如果有标记 1，如果没有标记 0。有了这个 CardTable，在 root-tracing 时，把 CardTable 标记为存在引用的内存区域的对象都加进 GC roots，就能收集到完整的 GC roots 了。

### 引用

* 《深入Java虚拟机第2版》
* http://tech.meituan.com/g1.html
* https://www.infoq.com/articles/tuning-tips-G1-GC
* http://hllvm.group.iteye.com/group/topic/44381
* http://hllvm.group.iteye.com/group/topic/21468