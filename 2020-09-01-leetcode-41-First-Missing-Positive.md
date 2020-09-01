---
layout: post
title: First Missing Positive
date: 2020-09-01
tags: leetcode
---

问题 ：https://leetcode-cn.com/problems/first-missing-positive/

解：https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/P41_2.java

要求：

> 间复杂度应为O(*n*)，并且只能使用常数级别的额外空间

<!-- more -->

解：不申请新的内存，那么就去复用给的 nums 数组。思路是将每个数组元素根据其值维护到对应的数组位置。负数，0，还有超出数组大小的数不处理。然后再起一个循环，取第一个数组元素和下标不等的即可得到结果。

第一版用 map 做，申请了内存。

Java 的实现，就算复用 nums 数组，执行结果的内存使用也差别不大，都是 37 M左右，用 C 的话应该区别很明显。