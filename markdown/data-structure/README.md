# Redis

## 数据类型

传统的KV存储模型，K和V一般都是字符串，在redis里面K值仍然对应字符串，但是V值不在局限于字符串，可以是更为复杂的数据结构，它们可以是

* 二进制安全([binary-safe](https://en.wikipedia.org/wiki/Binary-safe))的字符串
* 列表，通常是linked list
* 去重的字符串集合，但没有刻意排序
* 排序了的去重的字符串集合，排序是依靠给每个字符串元素讲了一个浮点数实现的，称之为score
* 哈希结构，类似Python中的dict
* bit数组(位图)，可以把字符串的值看作是bit的数据来操作
* HyperLogLogs，需要研究一下

## Redis key

Redis的key使用二进制安全([binary-safe](https://en.wikipedia.org/wiki/Binary-safe))的字符串，而且key值可以为空

除此之外，Redis还支持针对key值设置过期时间，过期之后，相对于对该key-value对执行了删除操作，而且过期信息是会被持久化到磁盘上并分发到子节点上

key值最大是512MB

在创建或者删除key的时候，redis遵守三条规则

1. 给一个不存在的key添加一些聚合类型的值的时候，会先创建一个空的聚合值，然后再把元素添加进去

```
> del mylist
(integer) 1
> lpush mylist 1 2 3
(integer) 3
```

2. 当把一个聚合类型的值里面所有的元素都能移除之后，key值自动配删除

```
> lpush mylist 1 2 3
(integer) 3
> exists mylist
(integer) 1
> lpop mylist
"3"
> lpop mylist
"2"
> lpop mylist
"1"
> exists mylist
(integer) 0
```

3. 针对一个不存在的K-V对，如果用只读命令或者写命令移除元素，操作的结果和对一个存在K值，但是V值为空的操作相同

```
> del mylist
(integer) 0
> llen mylist
(integer) 0
> lpop mylist
(nil)
```

## Redis String

最大空间占用是512MB

## Redis Lists

Redis使用Linked List，具体实现是C语言中双向链表结构

```C
typedef struct listNode {
    struct listNode *prev;
    struct listNode *next;
    void *value;
} listNode;
```

优点是访问并修改头和尾的元素值会非常高效，尤其是在列表非常长的时候，缺点也很明显，随机访问的效果不好。Redis这么做有他自己的想法

> Redis Lists are implemented with linked lists because for a database system it is crucial to be able to add elements to a very long list in a very fast way. Another strong advantage, as you'll see in a moment, is that Redis Lists can be taken at constant length in constant time.

Redis为自己这么做找到了具有代表性的使用场景

> 在社交网络中查看用户最新的状态更新(Twitter的社交网络就采用了Redis的list结构)
> 两个进程之间的通信，生产者和消费者模型

如果非要对一长串数据的中间某些元素的访问，可以用集合(Set)

特别的，针对生产者和消费者模型，可以把Redis当做是一个消息队列来使用，但是如果list的元素为空，那么消费这就不得不写个循环来不停的请求数据，请求的频率过快，会产生大量的无效操作；请求的太慢，实时性又不够。因此Redis支持阻塞式的请求，当没有数据的时候消费者阻塞，直到有数据或者达到超时时间

## Redis 哈希

Redis的哈希数据类型和Python中的dict类似，{key1:value1; key2:value2}，下面的示例中创建了一个K值为user:1000，V值为hash的记录，hash里面一共有三组K-V对，{username:antirez;birthyear:1977; verified:1}

```
> hmset user:1000 username antirez birthyear 1977 verified 1
OK
> hget user:1000 username
"antirez"
> hget user:1000 birthyear
"1977"
> hgetall user:1000
1) "username"
2) "antirez"
3) "birthyear"
4) "1977"
5) "verified"
6) "1"
```

















## 参考

1. 官方说明文档. https://redis.io/topics/data-types-intro
2. 二进制安全. [binary-safe](https://en.wikipedia.org/wiki/Binary-safe)
