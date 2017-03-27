# Redis

## 持久化

Redis提供两种数据持久化的方法，当然也可以不持久化数据

* RDB, 在指定间隔内数据的快照，备份的文件是序列化过的二进制文件dump.rdb
* AOF, 所有写操作的日志记录，这样可以用来重建数据集，当日志文件太大的时候还可以进行压缩

## RDB的优劣

RDB相当于对全数据集的一个定时备份，而备份的过程中不会对查询操作阻塞

定时备份的缺点是有一部分数据会丢失，在发生故障的时候，文件可能会损坏

## AOF的优劣

AOF相当于写前日志，但是也是可以配置的，每次查询、每秒记录。由于追加写的特性，在故障之后，AOF文件是可恢复的。文件太大的时候会进行压缩。AOF文件具有良好的可读性，易于人工恢复

AOF文件通常比RDB文件要大

## RDB 备份流程

* fork()出一个子线程。子线程拥有和父线程相同的执行代码，它们之间通过pid来区别

```C
if ((childpid = fork()) == 0) {
    /* Child */
    // ... ignore some codes
} else {
    /* Parent */
    // ... ignore some codes
}
```


* 子线程开始将数据写到临时RDB文件中
* 子线程完成上述工作之后，将原来的文件替换

上面的方法用到了Copy-on-write的技术

> Copy-on-write (COW), sometimes referred to as implicit sharing or shadowing, is a resource-management technique used in computer programming to efficiently implement a "duplicate" or "copy" operation on modifiable resources. If a resource is duplicated but not modified, it is not necessary to create a new resource; the resource can be shared between the copy and the original. Modifications must still create a copy, hence the technique: the copy operation is deferred to the first write. By sharing resources in this way, it is possible to significantly reduce the resource consumption of unmodified copies, while adding a small overhead to resource-modifying operations.


## AOF备份流程

AOF作为一个可靠性更强的备份机制，可以将每次操作的信息记录，也可以每秒记录，或者不记录。主要流程如下

* fork()出一个子线程。子线程拥有和父线程相同的执行代码，它们之间通过pid来区别
* 子线程开始将数据写到临时AOF文件中(要注意这里是直接访问内存中的数据，构成新的AOF文件，不会去读原来的磁盘上的文件)
* 父线程将所有新的修改缓存在内存中，同时把这些修改写到旧的AOF文件中
* 当子线程完成了重写操作之后，父线程得到一个信号，将缓存在内存中的操作写到新的AOF文件中
* Redis将原来旧的AOF文件更名为新的AOF文件

> The rewrite is NOT performed by reading the old one, but directly accessing data in memory, so that Redis can create the shortest AOF that is possible to generate, and will not require read disk access while writing the new one.
> 
> see more from http://oldblog.antirez.com/post/redis-persistence-demystified.html


## AOF和RDB协作

两个备份机制不会同时进行，避免大量磁盘IO

## 参考

1. [copy-on-write](https://en.wikipedia.org/wiki/Copy-on-write)(COW)
2. http://oldblog.antirez.com/post/redis-persistence-demystified.html
