# RedisNotes

Redis学习笔记

**借由讨论班的机会，一窥K-V数据库**

本系列将结合源码分析redis在架构、数据模型、事务、并发、持久化等方面的内容

## 简介

Redis 是一个开源（BSD许可）的，内存中的数据结构存储系统，它可以用作数据库、缓存和消息中间件。 它支持多种类型的数据结构，如 字符串（strings）， 散列（hashes）， 列表（lists）， 集合（sets）， 有序集合（sorted sets） 与范围查询， bitmaps， hyperloglogs 和 地理空间（geospatial） 索引半径查询。 Redis 内置了复制（replication），LUA脚本（Lua scripting）， LRU驱动事件（LRU eviction），事务（transactions） 和不同级别的磁盘持久化（persistence）， 并通过 Redis哨兵（Sentinel）和自动 分区（Cluster）提供高可用性（high availability）。

## 参考资料

1. redis官方网站 https://redis.io/
2. 维基百科关于redis的介绍 https://en.wikipedia.org/wiki/Redis
3. redis中文网站 http://www.redis.cn/
4. redis源码分析 http://bbs.redis.cn/forum.php?mod=viewthread&tid=544
