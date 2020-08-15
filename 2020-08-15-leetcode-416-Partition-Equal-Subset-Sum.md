---
layout: post
title: Partition Equal Subset Sum
date: 2020-08-16
tags: leetcode
---

问题 ：https://leetcode-cn.com/problems/partition-equal-subset-sum/

解：https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/P416.java

动态规划

基本思想：如果前 n 个数，组成的和是 s1、s2，此时如果再加上下一个数 next，组成的和是 0 + next、s1 + next、s2 + next。

<!-- more -->

比如数列：1 5 11 5。 

已知前 0 个数，能组成和 0。

那么前 1 个数，组成 0、1

那么前 2 个数，组成 0、1、5、6

那么前 3 个数，组成 0、1、5、6、11、12、16、17，此时 11 刚好命中目标结果

用一个数组 a，a[i] 表示可以通过从数列中取若干个数，满足和为 i。

每次取一个 num，如果从前向后维护数组，后面的位会被重复维护，除非用其他的标志位过滤。另一种思路是从后往前维护。

代码：

```java
    public boolean canPartition(int[] nums) {
        int sum = 0;
        for (int num : nums) {
            sum += num;
        }
        if (sum % 2 != 0) {
            return false;
        }
        int half = sum / 2;

        int[] dp = new int[sum + 1];
        dp[0] = 1;
        for (int num : nums) {
            for (int i = sum; i >= num; i--) {
                if (dp[i - num] == 1) {
                    dp[i] = 1;
                    if (i == half) {
                        return true;
                    }
                }
            }
        }
        return false;
    }
```

资料：https://www.bilibili.com/video/BV1AW411y7So