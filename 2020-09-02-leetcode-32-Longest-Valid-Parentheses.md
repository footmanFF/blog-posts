---
layout: post
title: Longest Valid Parentheses
date: 2020-09-02
tags: leetcode
---

问题 ：https://leetcode-cn.com/problems/longest-valid-parentheses/

解：https://gitee.com/footmanff/leetcode/blob/master/src/main/java/com/footmanff/leetcode/dp/P32.java

<!-- more -->

解：使用动态规划。

问题实际上是求最长的「有效括号集」。

**子问题定义：**将输入定义为一个 char 数组 s，每个子问题定义为「已某个数组元素为结尾有效括号序列」。

定义数组 dp，dp[i] 为以 s[i] 为结尾的有效括号序列，比如 s 为 「(」「(」「)」「)」，那么 dp[3] 就是 2，因为数组第三个元素作为结尾的话，只有一组括号对有效，长度为 2。

**转移函数：**思考如果已知 dp[1] .. dp[i -1]，需要求 dp[i]，如何求解。

dp[i] 相对于 dp[i -1] 只是在末尾附加一个括号而已，是可以求解的：

- 如果 s[i] 为 ‘(’ 直接设 dp[i] 为 0；

- 如果 s[i] 为 ')'，取 s [i - 1]，如果 s [i - 1] 为 '('，刚好和新添加的括号组成对，dp[i] 即为 dp[i-2] + 2，新组成的括号对长度为 2；

  - 如果 s [i - 1] 为 ')'，并且 dp[i- 1] 如果非 0 的话，取数组 s 的下标为 i - dp[i- 1] - 1 位置的字符，如果为 '(' ，那么刚好也和新添加的括号组成括号对，dp[i] = dp[i - 1] + dp[i - dp[i - 1] - 2] + 2。

    其中 dp[i - dp[i - 1] - 2] 是指配对的 '(' 之前的有效括号长度，有可能连接上的之前的有效括号。

还有其他的方式，比如栈。

#### 资料

思路：https://leetcode-cn.com/problems/longest-valid-parentheses/solution/dong-tai-gui-hua-si-lu-xiang-jie-c-by-zhanganan042/