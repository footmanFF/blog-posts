---
layout: post
title: Java的内存可见性
date: 2018-03-14
tags: Java、JMM
---

已知的 Java 内能保证内存可见性（能做到线程间通信）现在知道大致有这些：

- synchronized  [深入理解Java内存模型（五）——锁](http://www.infoq.com/cn/articles/java-memory-model-5)
- volatile
- Unsafe 内有内存屏障相关的方法，应该也能保证可见性。举个例子 compareAndSwapInt 方法有 volatile 的读和写的内存语义。详细见《Java 内存模型  - 程晓明》「锁内存语义的实现」一节对 ReentrantLock 的分析。（2018 - 03 - 28 更新，另外见 http://www.infoq.com/cn/articles/java-memory-model-5，这个语义是从程晓明的文章中看到的，那么是不是 CAS 变更一个没有用 volatile 申明的变量也是具有 volatile 读和写的语义的？现在看到的代码中一般都是 CAS 去改 volatile 的变量，比如 AQS 中。**TODO**）
- final 的属性的发布，有点不同，见下面的引用 [深入理解Java内存模型（六）——final](http://www.infoq.com/cn/articles/java-memory-model-6)

<!-- more -->

## 问题

##### 重排序、happens-before、可见性三者的关系是什么？

重排序是一种现象，编译器、CPU等都会造成指令的重排序。内存的在多线程环境下存在的可见性问题的原因之一就是指令重排序。而 happens-before 描述一种是关系，happens-before 约束下的多线程并行执行能解决重排序，并间接保证了内存的可见性。

##### 重排序

> 在执行程序时为了提高性能，编译器和处理器常常会对指令做重排序。重排序分三种类型：
>
> 1. 编译器优化的重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序。
>
>
> 2. 指令级并行的重排序。现代处理器采用了指令级并行技术（Instruction-LevelParallelism, ILP）来将多条指令重叠执行。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序。
> 3. 内存系统的重排序。由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。
>
> 《Java 内存模型》 —  程晓明

##### ReentrantLock（AQS）是不是能保证可见性？

是的，AQS内部是用了 volatile 和 CAS 去做的，都能保证内存可见性。

##### final 属性

> 写 final 域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的 final 域已经被正确初始化过了，而普通域不具有这个保障。
>
> 读 final 域的重排序规则可以确保：在读一个对象的 final 域之前，一定会先读包含这个 final 域的对象的引用。在这个示例程序中，如果该引用不为 null 那么引用对象的 final 域一定已经被 A 线程初始化过了。
>
> 通过为 final 域增加写和读重排序规则，可以为 java 程序员提供初始化安全保证：只要对象是正确构造的（被构造对象的引|用在构造函数中没有“逸出”），那么不需要使用同步（指 lock 和 volatile 的使用），就可以保证任意线程都能看到这个 final 域在构造函数中被初始化之后的值。
>
> 《Java 内存模型》 —  程晓明

## 资料

[Java CAS 原理深度分析](http://zl198751.iteye.com/blog/1848575)



