---
layout: post
title: Redis分布式锁
date: 2017-12-03
tags: Reids, 分布式锁
---

公司有个项目要用到分布式锁，之前有模仿别人用 memcache 实现过一个，但是现在的公司用 redis ，基本思路还是差不多，网上搜点资料就开始撸。查了不少用 redis 实现分布式锁的文章，发现以前想的思路并不能保证能锁的住，而且不出幺蛾子。本文记录一些基本思路，还有一些资料记录。

<!-- more -->

#### 最基本的实现思路

加锁用下面这个 redis 命令，给锁的 key 设置一个随机的 value，并加上过期时间。过期时间是为了当出现客户端因为各种意外而无法释放锁的时候，让锁能够最终自己释放掉自己。这个超时时间是要权衡的。 random_value 是一个唯一串，存在的目的是让锁的是否只能被锁的拥有者执行。

```
set key random_value NX PX expireTime 
```

解锁：

```
if redis.call('get', KEYS[1]) == ARGV[1] 
    then return redis.call('del', KEYS[1]) 
else 
    return 0 
end
```

这是一段 lua 脚本，redis 保证原子性执行。

更多的实现细节可以参考[Redis 分布式锁的正确实现方式（Java版）][lock-java]

#### 这个实现的问题

1. 如果持有了锁以后，业务逻辑占用了太多时间最终导致锁超时而自己自动释放了（GC、网络延时等都会造成锁超时），那么其他的客户端也能获得锁，这个时候锁就重入了。
2. 如果客户端在持有了锁以后宕机，没有去释放锁。那在锁自己超时释放以前，同样的加锁请求是会被阻塞的。
3. redis 单机或者主从结构下，redis 本身宕机会造成锁服务不可用。单机不用说，主从情况，因为复制是异步的。主在宕机以后，切换到从，可能会丢失一部分 key，这部分的锁也会失效。

#### Redlock

redlock 解决了上面的第三个问题，具体实现思路可以看这篇文章[基于Redis的分布式锁到底安全吗（上）？][lock1]

他的思路是用多独立的 redis 去作为锁服务，在这多台 redis 上成功获取锁超过一半时，才算获取成功，只要这些 redis 不全挂，那锁服务本身是可用。但是他还是有一些问题：

1. 上面提到的问题 1 和 2 其实同样解决不了
2. 依赖系统时间，当系统时间发生跳跃的时候，锁会失效。

#### ZooKeeper的实现方式

zookeeper 实现的思路类似，可以看这篇文章[基于Redis的分布式锁到底安全吗（下）？][lock2]，相对于 redis 有优势的一点是，他不需要设置锁的超时时间，zookeeper 有一种机制可以自动检测客户端是否存活，如果不存活就去自主释放掉锁。这能解决上面提到的 redis 基本实现中的第二个问题。单机可用性问题需要依赖 zookeeper 本身的能力，zookeeper 本身不太熟悉，先埋个坑。

还有一个优势是获取锁的时候可以用上 watch 机制，当锁已经存在时加锁请求会被阻塞。具体实现依旧看[基于Redis的分布式锁到底安全吗（下）？][lock2]这篇文章。

关于这个实现可以看这篇[sc_recipes_Locks][sc_recipes_Locks]。

#### Chubby

google 的分布式锁，未深入了解，先埋下个坑，更多内容待补充。

#### 另一篇写分布式锁，redisson 相关

http://www.jsondream.com/2016/12/20/lock-redis-lock.html

这篇写了很多之前没考虑到的东西，值得研究一下

另一篇文章：

https://zhuanlan.zhihu.com/p/33038831?iam=4aaee5f51c32c3ee2d20727430d50386?utm_medium=social&utm_source=wechat_session

#### 一些资料

> http://ifeve.com/redis-lock/
>
> https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
>
> https://xiaozhuanlan.com/topic/4672859130   这篇可以参考下，正确程度待考量
>
> http://zhangtielei.com/posts/blog-redlock-reasoning.html 这篇写得非常好
>
> http://zhangtielei.com/posts/blog-redlock-reasoning-part2.html  这篇是上一篇的续，非常好
>
> 《redis 实战》6.2节 分布式锁
>
> <http://pengshuang.space/2016/10/26/Google-Chubby-学习-基本概念>
>
> http://zookeeper.apache.org/doc/r3.4.9/recipes.html#sc_recipes_Locks

[lock-java]: https://xiaozhuanlan.com/topic/4672859130
[Lock1]: http://zhangtielei.com/posts/blog-redlock-reasoning.html
[lock2]: http://zhangtielei.com/posts/blog-redlock-reasoning-part2.html
[sc_recipes_Locks]: http://zookeeper.apache.org/doc/r3.4.9/recipes.html#sc_recipes_Locks