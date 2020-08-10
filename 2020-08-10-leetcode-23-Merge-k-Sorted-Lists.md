---
layout: post
title: Merge k Sorted Lists
date: 2020-08-10
tags: leetcode
---

问题：https://leetcode.com/problems/merge-k-sorted-lists/

解：https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/P23_2.java

> Runtime: 6 ms, faster than 54.03% of Java online submissions for Merge k Sorted Lists.
>
> Memory Usage: 41.4 MB, less than 59.41% of Java online submissions for Merge k Sorted Lists.

解法是对 ListNode array 头部数字排序，然后每次对第一个位置的 array 执行出队，出队以后，利用重新对所有 array 排序（冒泡），并重复执行上述过程。

要再找个更优解法。