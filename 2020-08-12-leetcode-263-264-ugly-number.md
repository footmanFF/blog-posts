---
layout: post
title: Ugly Number系列
date: 2020-08-12
tags: leetcode
---

问题 1：

https://leetcode-cn.com/problems/ugly-number/

解：https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/P263.java

问题 2：

https://leetcode-cn.com/problems/ugly-number-ii/

<!-- more -->

解 1，小根堆：

https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/P264_2.java

> 执行用时：77 ms, 在所有 Java 提交中击败了17.41%的用户
> 内存消耗：39.6 MB, 在所有 Java 提交中击败了10.64%的用户

解 2，动态规划：

https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/P264_3.java

设 fn 为第 n 个丑数，如果第 n 个丑数前有 3 个丑数分别是 a b c 他们分别乘以 2、3、5 以后，得出的结果刚好是 fn 的下三个丑数，那么取其中最小的那个就是 fn 的下一个丑数。

这是已知第 n-1 个丑数，计算第 n 个丑数的思路。

每次已知一个丑数求下一个丑数，明显 a b c 是需要不断维护到新值的，那么怎么去维护 a b c 呢？a b c 最初肯定都是 1。

现在设 fa fb fc 分别是 a\*2 b\*3 c\*5 以后的丑数，如果 a b c 的大小顺序刚好使得 fa fb fc 是下三个连续的丑数。并且 fa 是最小的丑数，也就是下一个丑数，那么选定 fa 为下一个丑数，此时原来的 a\*2 已经太小了（已经为已经被拿出来作为丑数了）需要重新维护一个「乘以 2 最近的比较小的丑数」，那么需要把 a 维护为「a 之后的第一个丑数」。这么描述估计过几天自己都看不懂 ...

核心的思路是，已知若干个丑数以后，求下一个丑数，这个新丑数必定是从前面所有丑数乘以 2、3、5 以后的，大于上一个丑数的最小丑数。

从这个基础上思考，其实不需要对所有历史丑数去做乘 2、乘 3、乘 5，以为一定存在三个数，每个数选择乘以 2、3、5 中的一个以后，刚好称为下面三个连续的丑数，取这之中的第一个丑数即可。

如何去准确的维护这三个刚好成为下三个丑数的「基数」？因为这个三个基数必定要分别选择乘以 2、3、5 中的一个，最初必定三个数都为 1。每次从三个基数中选择一个生成丑数以后，比如选择乘以 2 的基数为新丑数，那么将乘以 2 的基数，往后取下一个更大一点的丑数作为乘以 2 的新基数即可。 

> 执行用时：2 ms, 在所有 Java 提交中击败了93.41%的用户
> 内存消耗：37.7 MB, 在所有 Java 提交中击败了70.45%的用户