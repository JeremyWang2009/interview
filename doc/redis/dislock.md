# 概述
当业务发展到一定程度的时候，集群环境搭建服务就在所难免，集群环境不同于单机环境，单纯的 JVM 锁不再有意义，分布式锁自然应用而生，分布式锁可以有效解决程序的并发执行。
实现分布式锁的方式大致有三种，分别是基于 DB、缓存、Zookeeper 这三种。下面将讲解这三种分布式锁的具体实现方式。
# Base Redis
## 锁的要求
实现一个分布式锁最基本的需要满足三点：
- 安全属性：独占（相互排斥），在任意一个时刻，最多只有一个客户端持有该锁
- 活性 A：无死锁，即便持有锁的客户端崩溃，锁仍然可以释放
- 活性 B：容错性，只要大部分 Redis 节点都活着，客户端仍然可以获取和释放锁

## 锁的实现
1. 获取锁  

分布式锁解决的场景就是有点像我们平常玩的“抢椅子”游戏，当有人占用时，其他人只能放弃或者稍后重试。一般获取锁可以用 SETNX 指令，只允许被一个客户端占用，想离开时，只需要调用 DEL 指令删除就行。但是有个问题，如果程序执行到中间系统出现异常或者奔溃了，可能会导致 DEL 指令没被调用，这样就会出现**死锁**问题。于是我们想到 SETNX 的时候给命令加一个 EXPIRE 过期时间，但是这种也容易出现问题，SETNX 和 EXPIRE 是两个指令，在SETNX 和 EXPIRE 之间系统出现异常或者奔溃了，也会导致**死锁**。或许会有人想到 Redis 的事务，但是 Redis 事务不确保原子性（参考： [Redis Transcation](https://redis.io/topics/transactions) ）。但是从 Redis 2.8 版本开始支持 SET 命令同时设置 NX、EXPIRE 等参数，具体代码如下：
```
127.0.0.1:6379> SET lock_key unique_value NX PX 30000
OK
```
这个命令行 Redis 服务器保证 SETNX 和 EXPIRE 是一条指令， unique_value  这个值在所有获取客户端必须是唯一的，唯一性是为了后面释放锁的时候避免其他客户端释放该锁，unique_value 可以理解为 request_id；NX 的意思是 SET IF NOT EXIST,即 lock_key 不存在时，我们进行 SET 操作，若 lock_key 已经存在，不做任何操作；PX 给 lock_key 添加过期时间，具体时间就是后面的 30000。

2. 释放锁

释放锁大家首先会想到 DEL 命令直接操作，这样就会导致任意客户端都可以进行解锁。为了安全起见，应该只有加锁的拥有者能解锁，所以参考 1 里面说的添加 unique_value，在解锁之前先判断 lock_key 的值是否相等，如果相等，才能解锁，于是就有了下面的命令行解锁：
```
127.0.0.1:6379> GET lock_key
"7663e69f-dfee-4c77-9153-2e9a2f16d478"
// 程序里面判断 value 是否和客户端添加锁的 unique_value 是否相等，相等则执行如下 DEL 操作
127.0.0.1:6379> DEL lock_key
(integer) 1
```
上述操作其实忽略了一个问题， GET 和 DEL 是两个指令，也不是原子性的，所以可能会出现调用 DEL 指令的时候这个锁已经自动失效了（接口超时情况下可能会出现），不是该客户端锁持有的锁了，所以为了安全起见，Redis 的 Lua 脚本该应用了，代码如下：
```
127.0.0.1:6379> eval 'if redis.call("get",KEYS[1]) == ARGV[1] then return redis.call("del",KEYS[1]) else return 0 end' 1 lock_key unique_value
(integer) 1
```
上述代码是通过 Lua 脚本实现的，先获取 lock_key 对应的值，如果 lock_key 的值和获取锁设置的  unique_value 值一样，则删除锁，这样能避免其他客户端删除该锁。那为什么用 Lua 脚本实现？Redis 服务器会单线程原子性的执行 Lua 脚本，保证 Lua 脚本在处理的过程中不受外界打扰而中断。
## 锁的问题
即便按照上述要求实现了分布式锁，分布式锁也不是 100% 靠谱的，Redis 大多数公司仅仅只作为缓存，不作为持久化工具。对于分布式锁存在的问题总结如下：
1. 接口超时问题
如果获取锁和释放锁之间，接口出现超时，以至于超出了锁的过期时间，会导致并发执行临界区程序。
2. 不支持重入性
可重入性，是指线程在持有锁的情况下再次请求该锁，如果一个锁支持同一个线程多次添加锁，那么这个锁就是支持重入性的。单纯的 Redis 命令不支持，需要自己进行封装
3. Sentinel 集群主从发生 failover
客户端从 Master 节点获取锁，Master 将锁同步到 Slave 之前，Master 宕机了，Slave 节点被晋级为 Master 节点，此时也会出现并发执行临界区程序。

上述罗列的三个问题，生产环境都是小概率场景，但是为了避免，问题（1）建议分布式锁不要应用于时间较长的接口或任务；问题（2）一般业务使用到分布式锁都不会用到重入性，如需用到，可自行封装，也可以引用 [Redssion](https://github.com/redisson/redisson)；问题（3）的解决方案可以参考 [Redlock](https://redis.io/topics/distlock)。对于使用分布式的目的可以参考这篇文章 [Redis RedLock 完美的分布式锁么？](https://www.xilidou.com/2017/10/29/Redis-RedLock-%E5%AE%8C%E7%BE%8E%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E4%B9%88%EF%BC%9F/)，对于严格要求的场景（比如订单、金额），即便使用分布式锁，也不能保证锁的正确性，其他非严格场景，可以参考自己的业务是否能容忍这种小概率事件了。

# 参考阅读
1. [redis dislock](https://redis.io/topics/distlock)
2. [redis transactions](https://redis.io/topics/transactions)
3. [RedLock 是完美的分布式锁吗？](https://www.xilidou.com/2017/10/29/Redis-RedLock-%E5%AE%8C%E7%BE%8E%E7%9A%84%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81%E4%B9%88%EF%BC%9F/)
4. [分布式锁](http://zhangtielei.com/posts/blog-redlock-reasoning.html)
