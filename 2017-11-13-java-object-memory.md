---
layout: post
title: Java 对象内存
date: 2017-11-13
tags: Java
---

##### 组成类的内存

一个类占用的内存包括对象头部、成员变量引用地址（包括基本数据类型和对象引用）。在 32 位的 JVM 上，一个对象引用占用 4 个字节；在 64 位上，占用 8 个字节。

对象头包含三部分。第一部分是mark word，包括hashCode、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳等。第二部分是类元信息也就是 Class 对象的引用。第三部分是数组长度，只有在数组对象的情况下会有。

<!-- more -->

##### 对象填充

Java 对象以 8 字节的倍数填充，如果对象占用 14 个字节，那么自动填充到 16 字节。

##### 指针压缩

Jdk 6 开始默认开启，启动参数是 -XX:+UseCompressedOops。64 位的寻址位数是 8 字节，32 位是 4 字节，在未压缩前 64 位 JVM 的对象引用地址是 8 字节，压缩以后是 4 字节。但也不是全部都压缩，这些是会压缩的：

> - 每个Class的属性指针（静态成员变量）；
> - 每个对象的属性指针；
> - 普通对象数组的每个元素指针。

##### 试试各种对象的大小

64 位 JDK 8 下测试，分开启和不开启指针压缩，用 java.lang.instrument.Instrumentation 计算 shallow 内存大小。

先测开启指针压缩的：

```
size of array object: 16
// 16 = 8(_mark) + 4 (compressed_klass) + 4(size)

size of array object with 1 int: 24
// 24 = 8(_mark) + 4 (compressed_klass) + 4(size) + 1 * 4(int) + 4(gap)

size of empty object: 16
// 16 = 8(_mark) + 4 (compressed_klass) + 4(gap)

size of object with int: 16
// 16 = 8(_mark) + 4 (compressed_klass) + 4(int)

size of object with 2 int: 24
// 24 = 8(_mark) + 4 (compressed_klass) + 2 * 4(int) + 4(gap)
```

然后是未开启指针压缩的，变化就是 class 引用会增大到 8 字节：

```
size of array object: 24
// 24 = 8(_mark) + 8 (compressed_klass) + 4(size) + 4(gap)

size of array object with 1 int: 32
// 24 = 8(_mark) + 8 (compressed_klass) + 4(size) + 1 * 4(int)

size of empty object: 16
// 16 = 8(_mark) + 8 (compressed_klass)

size of object with int: 24
// 24 = 8(_mark) + 8 (compressed_klass) + 4(int) + 4(gap)

size of object with 2 int: 24
// 24 = 8(_mark) + 8 (compressed_klass) + 2 * 4(int)
```

开启压缩时试试 String 类型：

```java
private final char value[];
private int hash; // Default to 0
```

```
size of empty String: 24
// 24 = 8(_mark) + 4 (compressed_klass) + 4(value[]) + 4(int) + 4(gap)

size of abc String: 24
// 24 = 8(_mark) + 4 (compressed_klass) + 4(value[]) + 4(int) + 4(gap)
```

因为统计的是浅内存大小，所以 char 数组内元素不计算在内。

##### 如何计算对象的 deep 内存大小

<http://loveshisong.cn/编程技术/2015-06-11-java对象的内存大小.html>