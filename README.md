# RedisNotes

Redis学习笔记

**借由讨论班的机会，一窥K-V数据库**

本系列将结合源码分析redis在架构、数据模型、事务、并发、持久化等方面的内容

## 简介

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。 它支持多种类型的[数据结构](https://github.com/MyXOF/RedisNotes/tree/master/markdown/data-structure)，如字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）， 有序集合（sorted sets） 与范围查询， bitmaps， hyperloglogs 和地理空间（geospatial） 索引半径查询。 Redis内置了复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动分区（Cluster）提供高可用性（high availability）。

## 持久化

作为内存数据库，redis自然把数据保存在内存中，但是这样发生故障之后就不可恢复了。自从2.4版本之后，redis支持可以配置使用[虚拟内存](https://en.wikipedia.org/wiki/Virtual_memory),不过这个不推荐用。

现在的方式有两种。一个是快照技术，定时异步的将内存的数据写到磁盘，使用redis 定义的RDB dump格式。1.1版本之后，更加安全的策略是AOF(append-only file)，相当于将修改数据库的操作记录下来，同时redis自身会对这个文件进行重写，避免不可预见的文件增长

## 副本
redis采用主从模式，数据从server端复制给slave端，而salve端又可以成为别的slave端的server，这样，从开始的server出发，可以构建一棵树

redis的数据复制支持发布-订阅模式



## 参考资料

1. redis官方网站 https://redis.io/
2. 维基百科关于redis的介绍 https://en.wikipedia.org/wiki/Redis
3. redis中文网站 http://www.redis.cn/
4. redis源码分析 http://bbs.redis.cn/forum.php?mod=viewthread&tid=544
