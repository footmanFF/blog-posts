---
layout: post
title: Single Number
date: 2020-09-04
tags: leetcode
---

问题 ：https://leetcode-cn.com/problems/single-number/

解：https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/P136.java

<!-- more -->

解：使用位运算的异或

> 任何数和 00 做异或运算，结果仍然是原来的数。
> 任何数和其自身做异或运算，结果是 0。
> 异或运算满足交换律和结合律。

初始一个 r，直接遍历数组元素针对每个元素求 r = r ^ num，最终得出结果。

如果允许使用额外内存空间，hash 即可。