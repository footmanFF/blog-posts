---
layout: post
title: ES如何确定分片数量
date: 2018-05-29
tags: Elasticsearch, ES
---

分片不宜过大，在故障恢复的时候会更大的影响集群。

> The shard is the unit at which Elasticsearch distributes data around the cluster. The speed at which Elasticsearch can move shards around when rebalancing data, e.g. following a failure, will depend on the size and number of shards as well as network and disk performance.
>
> *TIP: Avoid having very large shards as this can negatively affect the cluster's ability to recover from failure. There is no fixed limit on how large shards can be, but a shard size of 50GB is often quoted as a limit that has been seen to work for a variety of use-cases.*

ES 只新增数据，不更新数据。更新也只是把旧的数据标记删除，再新增新的数据。被删除的数据在段合并前，是会一直占用资源的。有一种思路是按时间区间将数据分成不同的索引存储，比如 2017 年一份索引、2018 年一份索引。这样索引会更小，每次进入下一个年份都有机会调整分片数量和索引结果，冷数据也可以按年进入归档状态，不会影响在热数据上的业务服务。

<!-- more -->

太小的分片会导致问题：

> *Small shards result in small segments, which increases overhead. Aim to keep the average shard size between a few GB and a few tens of GB. For use-cases with time-based data, it is common to see shards between 20GB and 40GB in size.*

大量的小的段（segment）会影响搜索性能，定期的合并可以解决，但是合并是个高消耗操作，不能在业务高峰期合并。

> *As the overhead per shard depends on the segment count and size, forcing smaller segments to merge into larger ones through a forcemerge operation can reduce overhead and improve query performance. This should ideally be done once no more data is written to the index. Be aware that this is an expensive operation that should ideally be performed during off-peak hours.*

需要控制每个节点的分片数。

> *The number of shards you can hold on a node will be proportional to the amount of heap you have available, but there is no fixed limit enforced by Elasticsearch. A good rule-of-thumb is to ensure you keep the number of shards per node below 20 to 25 per GB heap it has configured. A node with a 30GB heap should therefore have a maximum of 600-750 shards, but the further below this limit you can keep it the better. This will generally help the cluster stay in good health.* 

ES 每个分片是单线程的处理一个查询的，多个分片的结果再合并到一起返回。因此一条查询的耗时也受分片数量和大小的影响。每个分片的大小和分片的总数需要权衡，不是分片越多（看起来并行执行）就越好，分片多意味着需要收集更多分片的结果并汇总排序，然后才能返回。

> In Elasticsearch, each query is executed in a single thread per shard. Multiple shards can however be processed in parallel, as can multiple queries and aggregations against the same shard.
>
> This means that the minimum query latency, when no caching is involved, will depend on the data, the type of query, as well as the size of the shard. Querying lots of small shards will make the processing per shard faster, but as many more tasks need to be queued up and processed in sequence, it is not necessarily going to be faster than querying a smaller number of larger shards. Having lots of small shards can also reduce the query throughput if there are multiple concurrent queries.

按照时间阔度来建索引（[Time-Based Data](https://www.elastic.co/guide/en/elasticsearch/guide/current/time-based.html)），比如作为日志平台的 ELK，可以每天新建一个索引，陈旧的索引只需要删除老的索引就行了，比 delete 会快。每天新建新的索引附带可以有机会调整新索引的分片数，如果索引量变大，增大分片数可以增加性能。每个时间跨度的索引可以设置别名，搜索的时候使用别名搜索所有的索引。[Rollover and Shrink APIs](https://www.elastic.co/blog/managing-time-based-indices-efficiently) 这两个 api 比较好用。

多索引（[multiple indices](https://www.elastic.co/guide/en/elasticsearch/guide/current/multiple-indices.html)）下的一次搜索和但索引多分片下的一次搜索本质是一致的，多索引下的索引热（on fly）扩容方法：[索引扩容方法](https://www.elastic.co/guide/en/elasticsearch/guide/current/multiple-indices.html#multiple-indices)。

>  Searching 1 index of 50 shards is exactly equivalent to searching 50 indices with 1 shard each: both search requests hit 50 shards.
>
> https://www.elastic.co/guide/en/elasticsearch/guide/current/multiple-indices.html#multiple-indices

可以先在单节点单分片上模拟测试真实的查询直到挂掉，这样可以评估出一台机器的能力。然后再评估自己大约要支撑最多多少数据量，然后再除以每个分片的量就能得出需要多少分片或者机器了。此处粗略描述，具体见 [Capacity Planning](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/capacity-planning.html)。

另外，也可以用基准测试去测量索引的性能。一款 ES 基准测试工具 [Announcing Rally: Our benchmarking tool for Elasticsearch](https://www.elastic.co/blog/announcing-rally-benchmarking-for-elasticsearch)

> **\*TIP:** The best way to determine the maximum shard size from a query performance perspective is to benchmark using realistic data and queries**. Always benchmark with a query and indexing load representative of what the node would need to handle in production, as optimizing for a single query might give misleading results.*

### 资料

[How many shards should I have in my Elasticsearch cluster?](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster)