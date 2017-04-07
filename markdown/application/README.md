# Redis

## LRU 缓存

64位系统默认使用的内存大小无限制，但是32位系统默认使用3GB的内存

当内存使用满了之后，就需要释放掉一部分缓存数据，有许多策略可以选择

* noeviction: 内存满了之后再插入就报错
* allkeys-lru: lru
* volatile-lru: lru 只针对设置了过期时间的key
* allkeys-random: 随机
* volatile-random: 随机 只针对设置了过期时间的key
* volatile-ttl: 设置了过期时间的key集合中中就快要过期的key

当没有过期key的时候，volatile-xxx的作用和noeviction一样


Redis采用的是近似的LRU算法
