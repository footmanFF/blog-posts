---
layout: post
title: 对text类型字段如何避免使用nested
date: 2019-06-19
tags: elasticsearch
---

### 背景

最近在优化 ES 的搜索性能，其中有一个优化点是避免对 nested 结果的使用，对 nested 结构内的字段查询时候，相对不非 nested 结构的字段，性能会有损耗。本文不去具体说这个损耗的原理，只说解决办法。感兴趣的可以看看这篇文章：https://www.jianshu.com/p/f0a15e21f61b。 

<!-- more -->

### 解决

解决办法很简单，将父子结构打平即可。本来在 nested 结构内的字段上提到父级。这里分几种情况：

1. 如果只会针对单个 nested 内字段进行查询，直接将字段提到父级，字段内容存成数组即可。

2. 如果会针对多个 nested 内字段进行查询，这种暂时没有找到特别好的解决办法。如果是只有两个字段，那么可以将两个字段值拼接以后形成一个新字段。比如新字段 field1-field2，字段值存所有 field1 和 field2 的组合。当然查询的时候如果同时查 field1 和 field2，那么需要对查询的值拼接起来以后在 field1-field2 字段上做查询。

   另一种办法是新增一个父级的 combine_field 字段，类型为 text，在索引每个子文档的时候，将 field1 和 field2 拼接起来并做成一个数组索引到 combine_field 字段中，需要注意的是，这里对拼接以后组合值的分词，只能分成拼接前的两个值。这种情况的查询和上一种办法一样，查询的值也是拼接以后的值。查询使用短语查询，也就是 phase_match，这样就能避免跨值匹配。这一块的原理可以看这两个连接：[understand-elasticsearch-multivalue-fields](https://stackoverflow.com/questions/49522118/understand-elasticsearch-multivalue-fields)、[position-increment-gap](https://www.elastic.co/guide/en/elasticsearch/reference/current/position-increment-gap.html)。

3. 如果只会针对单个 nested 内字段进行查询，并且字段类型是 text，即走分词和搜索。那么和第一种方式一样，也是将字段提到上级，并且组成一个数组。只是在查询的时候需要用短语查询，即 phase_match。原理见上述贴的两个文档。

### 最后

在优化索引期间，已经将所有 nested 结构都去掉，最终只剩下一个 text 字段，因为开始不知道 phase_match 的特性，测试的时候碰到跨值匹配问题，没办法去掉。有 phase_match 以后就能完美解决了。

### 资料

https://www.elastic.co/guide/en/elasticsearch/reference/current/position-increment-gap.html
https://stackoverflow.com/questions/49522118/understand-elasticsearch-multivalue-fields
https://www.elastic.co/guide/en/elasticsearch/reference/current/index-options.html