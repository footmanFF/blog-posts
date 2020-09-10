---
layout: post
title: Jump Game
date: 2020-09-09
tags: leetcode
---

问题 ：https://leetcode-cn.com/problems/jump-game/

解：https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/greedy/P55.java

<!-- more -->

规则 1：只要数组没有 0，一定能跳到最后。

规则 2：针对每个数组位置，只要本身能到达，那么就一定能跳到 i + (nums[i]) 位置。

维护一个最大可到达的数组下标。遍历 nums 数组，根据规则维护最大可到达的下标。然后利用规则 2 来判断出永远无法跳到最后的情况。