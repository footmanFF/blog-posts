---
layout: post
title: Jump Game 2
date: 2020-09-10
tags: leetcode
---

问题 ：https://leetcode-cn.com/problems/jump-game-ii/

解：https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/greedy/P45_1.java

<!-- more -->

解：利用一个数组维护到达下标 i 需要的最小步数。

遍历数组 nums，再以每个数组元素为起点，遍历所有能跳到的位置，如果目标位置为 0，则赋予到达需要的步数，如果目标不为 0，则去取「当前跳的步数」和「目标位置现在可以到达的步数」的较小值。

一个优化点：一个目标位置一旦从 0 更新为非 0，那么更新后的值一定是到达目标位置的最小步数。因此在针对每个位置起跳的时候，可以直接从下一个为 0 的位置开始跳，减少遍历的次数。

开始用的是模拟跳跃的方式，超时严重。