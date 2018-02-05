---
layout: post
title: 数据库索引结构
date: 2018-01-25
tags: db
---

### 聚簇索引和辅助索引

如果数据按照索引的顺序排列存储在磁盘上，称这个索引为聚簇索引：

![](http://note-1255449501.file.myqcloud.com/2018-01-25-132327.png)

<!-- more -->

如果数据不按照索引的顺序排列存储在磁盘上，称这个索引为辅助索引

![](http://note-1255449501.file.myqcloud.com/2018-01-25-132357.png)

### B+树的结构

B+树是平衡树(balanced tree)，根到每个叶子节点的路径长度相同，每个非叶子节点持有一定数量范围的子节点。

B+树的叶子节点和非叶节点都具有的基本结构：

![](http://note-1255449501.file.myqcloud.com/2018-01-25-134854.png)

K1… Kn-1 是索引值，P1 … Pn 是指针。

如果是叶子节点，指针指向 TODO。

如果是非叶节点，P1 指向 (无穷 , K1] 区间的节点，P2 指向 (K1, K2)区间的节点.

B+ 树的结构：

![](http://note-1255449501.file.myqcloud.com/2018-01-26-031853.png)

#### B+ 树查找

![](http://note-1255449501.file.myqcloud.com/2018-01-27-Screen%20Shot%202018-01-27%20at%208.31.07%20PM.png)

#### B+ 树新增

![](http://note-1255449501.file.myqcloud.com/2018-01-27-Screen%20Shot%202018-01-27%20at%208.20.35%20PM.png)

### MySQL 索引：

http://blog.codinglabs.org/articles/theory-of-mysql-index.html