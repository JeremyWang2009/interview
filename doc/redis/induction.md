# 概述
NoSQL数据库，非关系型数据库，它可以存储键（Key）与5种不同类型的值（Value）之间的映射，可以将存储在内存的键值对数据持久化磁盘，可以使用复制特性来扩展读性能，还可以使用客户端分片来扩展写性能。
高性能键值缓存服务器memcached也经常被拿来与Redis进行比较，彼此的性能也相差无几，但是Redis能够自动以两种不同的方式写入硬盘，并且Redis除了能存储普通的字符串以外，还可以存储其他4种数据结构，而Memcached只能存储普通的字符串键。

_分片是一种将数据划分为多个部分的方法，对数据的划分可以基于键包含的ID、基于键的散列值，或者基于以上两者的某种组合。通过对数据进行分片，用户可以将数据存储在多台机器里面，也可以从多台机器里面获取数据，这些方法在解决某些问题时可以获得线性级别的性能提升。_

# 各个数据库对比
![image](https://user-images.githubusercontent.com/12162133/42142391-ae6bb582-7de1-11e8-8e8d-4850a6cc6732.png)
# 存储数据
STRING、LIST、SET、HASH、ZSET（有序集合）
![image](https://user-images.githubusercontent.com/12162133/42192924-628b132a-7e9e-11e8-9826-23f8253bc378.png)
完整的Redis列表可以在 http://redis.io/commands 上面查看。Java代码操作Redis可以在网站：https://github.com/josiahcarlson/redis-in-action
1. STRING
GET  SET DEL
2. LIST
RPUSH LRANGE  LINDEX LPOP
>lrange list_name 0 -1       --使用0作为起始，-1作为结束，将获取列表包含的所有元素
3. SET
列表可以存储多个相同的字符串，而集合则通过使用散列表来保证自己存储的每个字符串都是各不相同的。
SADD SMEMBERS SISMEMBER SREM
SMEMBERS 返回集合的所有元素，如果集合元素很多，那么命令执行的速度很慢
SINTER SUNION SDIFF 常见的交集、并集、差集计算
4. HASH
![image](https://user-images.githubusercontent.com/12162133/42286290-f458080c-7fe4-11e8-9313-a4d993244876.png)
散列既可以存储字符串又可以使数字值，并且用户可以同样对散列存储的数字值执行自增操作或自减操作。
HSET HGET HGETALL HDEL
可以将Redis的散列看作是关系数据库里面的行
5. ZSET
![image](https://user-images.githubusercontent.com/12162133/42286490-ad41b4e4-7fe5-11e8-9224-0ef731174705.png)
有序集合和散列一样，都用于存储键值对：有序集合的键被称为成员（各不相同），有序集合的值被称为分支（score），分值必须是浮点数。
>zadd zset_name 1 member1
>zadd zset_name 2 member2
>zrange zset_name 0 -1 withscores    --获取有序集合的所有元素，多个元素会按照分值大小进行排序
1)"member1"
2)"1"
1)"member2"
2)"2"
>zrangebyscore zset_name 0 2 withscore   --可以根据分值范围来获取有序集合中的一部分元素
>zrem zset_name  member1

![image](https://user-images.githubusercontent.com/12162133/42297331-535c8354-8031-11e8-9d36-8f21e30676d8.png)

如何通过Redis来实现简单的限流策略？
- 简单限流
系统限定用户的某个行为在指定的时间内只能允许发生N次，如何使用Redis的数据结构来实现这个功能。这个限流是一个**滑动时间窗口**。而且我们可以保留这个时间窗口，窗口之外的数据都可以去掉，zset即可，zset的score值非常重要，value值只需要保证唯一即可（时间戳或者uuid），java代码如下：
```
public boolean isActionAllowed(String userId, String actionKey, int period, int maxCount) {
    String key = String.format("hist:%s:%s", userId, actionKey);
    long nowTs = System.currentTimeMillis();
    Pipeline pipe = jedis.pipelined();
    pipe.multi();
    pipe.zadd(key, nowTs, "" + nowTs);
    pipe.zremrangeByScore(key, 0, nowTs - period * 1000);
    Response<Long> count = pipe.zcard(key);
    pipe.expire(key, period + 1);
    pipe.exec();
    pipe.close();
    return count.get() <= maxCount;
}
```
每一个用户请求时，都维护一次时间窗口，将时间窗口外的记录全部清理掉，只保留窗口内的记录。zset集合只有score值重要，value值没有特别意义。上述代码所示，几个命令都是对同一个key操作，使用pipline可以显著提升Redis的存取效率。但是这种方案也有缺点，如果是一个时间窗口内请求量特别大（例：100万），上述代码不适合这样的场景，因为会消耗大量的存储空间。
- 漏斗限流
漏斗限流是常用的限流方法之一，这个算法来源于漏斗的结构，漏斗的容量是有限的，如果将漏斗堵住，往里面灌水则会满，如果将漏斗放开，往里面灌水速率大于漏水速率则会满，反之，则永远不会满。

- 令牌桶限流
两个核心组件：令牌桶、令牌桶生成器
![image](https://user-images.githubusercontent.com/12162133/64484198-67f00080-d241-11e9-9c77-d7fe89794e89.png)
令牌桶的数量：limit-burst
令牌生成器：limit 
```
package com.zfzjic.lock.dislock;

import java.util.concurrent.ScheduledThreadPoolExecutor;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Created by shanguang.wang on 2019-09-08
 */
public class TokenBucket {

    /**
     * 令牌桶容量
     */
    private int capacity;

    /**
     * 速率（count/s）
     */
    private int rateCountPerSeconds;

    /**
     * 当前令牌个数
     */
    volatile AtomicInteger currentToken = new AtomicInteger(0);

    public void tokenGenerator(int capacity, int rateCountPerSeconds) {
        this.capacity = capacity;
        this.rateCountPerSeconds = rateCountPerSeconds;

        ScheduledThreadPoolExecutor scheduledThreadPoolExecutor = new ScheduledThreadPoolExecutor(1);
        scheduledThreadPoolExecutor.scheduleAtFixedRate(() -> {
            put();
        }, 0, 1, TimeUnit.SECONDS);
    }

    private void put() {
        if (currentToken.get() < capacity) {
            currentToken.addAndGet(rateCountPerSeconds);
            System.out.println("令牌桶生成器生成：" + currentToken.get());
            return;
        }
        System.out.println("令牌桶已经满了");
    }

    private boolean get() {
        if (currentToken.get() >= 1) {
            currentToken.decrementAndGet();
            System.out.println("令牌桶取走令牌:"+currentToken.get());
            return true;
        }
        System.out.println("令牌桶已经空了");
        return false;
    }

    public static void main(String[] args) throws Exception{
        TokenBucket tokenBucket = new TokenBucket();
        tokenBucket.tokenGenerator(10, 2);

        while (true){
            Thread.sleep(800);
            tokenBucket.get();
        }
    }
}
```
上述代码只是简单的一个令牌通的限流算法，而且仅仅适用于单机。如果在分布式环境中想要限流怎么办，首先大家会想到的是 Redis，那么 Redis 是如何解决分布式限流问题呢？解决方案可以是 Redis + lua script 来实现。为什么需要用到 lua script 呢，获取令牌的操作要尽可能保持原子性，否则无法保证限流器是否能正常工作，在 RateLimiter 实现中使用了mutex 作为互斥锁来保证操作的原子性。这里采用 redis + lua script 的方式来保证原子性。

6. 位图
在我们平时开发过程中，会有一些bool类型数据需要存储，为了解决这个问题，Redis提供了位图结构，称为bitmap，是一连串的2进制数字，每一位所在的位置称为offset，bitmap就是通过最小的单位bit进行0或1的设置。设置的时候时间复杂度是O(1)，读取的时间复杂度是O(N)，二进制数据的存储，操作速度非常快，而且方便扩容。但是redis的key最大限制在512M之内，所以最大是2^32位。
场景：比如需要记录用户一年的签到记录，要记录365天，如果要使用普通的key/value，每个用户要记录365个，当用户上亿的时候，所需要的存储空间是很大的。
解决上述问题，bitmap派上了用场，这样，每天的签到记录只占一位，365天就是365位，大概46个字节。这大大的节省了空间。位图不是特殊的数据结构，它的内容就是普通的字符串，也就是byte数组。我们可以通过普通的get/set获取整个位图的内容，也可以使用位图命令getbit/setbit等将byte数组看成位数组来处理。
统计和查找，Redis提供了位图统计bitcount和位图查找命令bitpos，bitcount用于统计指定范围内1的个数，bitppos指令查找第一个出现1的位数。参数start end是字节索引，也就是是指定位范围是必须是8的倍数。
```
127.0.0.1:6379> SETBIT hello_key 1 0
(integer) 0
127.0.0.1:6379> SETBIT hello_key 2 0
(integer) 0
127.0.0.1:6379> SETBIT hello_key 3 1
(integer) 0
127.0.0.1:6379> SETBIT hello_key 4 0
(integer) 0
127.0.0.1:6379> SETBIT hello_key 5 1
(integer) 0
127.0.0.1:6379> BITCOUNT hello_key 0 0
(integer) 2
127.0.0.1:6379> BITPOS hello_key 1 0 0
(integer) 3
```

7. HyperLogLog
Redis的HyperLogLog是用来做基数统计的算法，HyperLogLog的优点是，在输入元素的数量或者体积非常大的时，计算基数所需空间总是固定的，并且很小的。在Redis里面，每个HyperLogLog键只需12KB的内存，就可以计算接近2^64个不同元素的基数。概率算法不直接存储数据集合本身，通过一定概率方法统计预估基数值，这种方法可以大大节省内存，同时保证误差控制在一定范围内。
```
127.0.0.1:6379> PFADD web_uv 'wsg'
(integer) 1
127.0.0.1:6379> PFADD web_uv 'gsw'
(integer) 1
127.0.0.1:6379> PFADD web_uv 'www'
(integer) 1
127.0.0.1:6379> PFADD web_uv 'sss'
(integer) 1
127.0.0.1:6379> PFCOUNT web_uv
(integer) 4
```
8.Scan
有时候需要从Redis从百上万个key中找出特定前缀的key列表来手动处理数据，修改它的值，Redis提供了一个简单暴力的指令keys用来列出所有满足特定正则字符串规则的key。
```
127.0.0.1:6379> KEYS *
 1) "wsg_uv"
 2) "f"
 3) "str-key"
 4) "bitarray"
 5) "counter"
 6) "total"
 7) "web_uv"
 8) "hello_key"
 9) "hello_bit"
10) "lock:apple"
11) "incr:key"
12) "a"
13) "b"
127.0.0.1:6379> KEYS h*
1) "hello_key"
2) "hello_bit"
```
这个指令非常简单，提供一个简单的正则字符串即可。但是这个命令很**”危险“**，因为没有offset、limit限制，万一实例中有几百万个key，则会全部显示出来；keys算法是遍历算法，复杂度是o(n)，如果实例中有千万级以上的key，这个指令就会导致Redis卡顿，所有读写Redis的其他的指令都会被延后甚至超时处理，因为Redis是单线程程序，顺序执行所有指令，其他指令必须等到当前的keys指令执行完才可以继续。

Redis为了解决这个问题，它在2.8版本中加入了指令scan，时间复杂度是o(n)，但是是通过游标分步执行的，不会阻塞线程；提供limit参数，可以控制每次返回的结果的最大条数；同keys一样，它也提供模式匹配功能；scan游标状态返回给客户端；返回的结果可能出现重复，需要客户端去重；遍历过程中如果有数据更改，则更改后的数据能不能遍历不能确定；单次返回的结果为空，并不意味着遍历结束，而要看返回的游标是否为0。
```
127.0.0.1:6379> SCAN 0 match str* count 10
1) "10"
2) 1) "str7"
   2) "str8"
   3) "str9"
   4) "str-key"
127.0.0.1:6379> SCAN 10 match str* count 10
1) "19"
2) 1) "str4"
   2) "str6"
   3) "str1"
   4) "str2"
   5) "str3"
127.0.0.1:6379> SCAN 19 match str* count 10
1) "0"
2) 1) "str5"
```
虽然上面是返回数量为10，但是返回结果不到10个，因为这个limit不是限定返回结果的数量，而是限定服务器单词遍历的字典槽位数量，即游标值不为0，意味着遍历还未结束。**在Redis中所有的key都存储在一个很大的字典中，这个字典的结构和Java中的HashMap一样，一位数组+二维链表**。scan指令返回的游标就是第一位数组的位置索引，我们称这个位置为槽(slot)，limit参数就是所谓的表示需要遍历的槽位数，之所以返回的结果有多有少，是因为不是所有的槽位数都会挂链表，有些槽位可能是空的，还有些槽位上挂的数据有可能是空的。所以每一次遍历的返回的数据都不一样。
scan指令是一系列指令，除了可以遍历所有key，还可以对指定的容器集合进行遍历，zscan遍历zset集合元素，hscan遍历hash字典的元素，sscan遍历set集合的元素。

9. geohash
Redis 3.2 版本之后增加了地理位置 GEO 模块，地图元素的位置使用二维的经纬度表示，GeoHash 算法将二维的经纬度映射到一维数组。
![image](https://user-images.githubusercontent.com/12162133/64537390-54ca5700-d34d-11e9-86c8-90e7dcfef00a.png)

（1）经纬度通过二分法转换成 0、1 序列
（2）将经度序列和维度序列顺序串在一起，奇数位放经度，偶数位放维度，组成一个 0、1 序列
（3）将序列 5 位分成一组，转换为 10 进制，然后通过 base32 编码组成字符串
https://www.cnblogs.com/LBSer/p/3310455.html

geohash 运用了 peano 空间填充曲线算法，将空间分成 4 块，编码的顺序分别为 00、01、10、11，也就类似 Z 曲线。
![image](https://user-images.githubusercontent.com/12162133/64541092-e341d700-d353-11e9-822f-80cfcc124456.png)
编码相似的距离很近，但是会出现最大的缺点就是突变性。

10. pipeline
这个技术本质上是客户端提供的，Redis客户端与服务器之间使用TCP协议进行通信，并且很早就支持管道（pipelining）技术了。在某些高并发的场景下，网络开销成了Redis速度的瓶颈，所以需要使用管道技术来实现突破。
- 客户端把命令发送到服务器，然后阻塞客户端，等待着从socket读取服务器的返回结果
- 服务器处理命令并将结果返回给客户端
为了解决这个问题，Redis 很早就支持了管道技术，也就是说客户端一次可以发送多条命令。
![image](https://user-images.githubusercontent.com/12162133/64905940-c588bf00-d711-11e9-93a9-b503f5b4f6d4.png)
命令的执行时间 = 客户端调用write并写网卡时间+一次网络开销的时间+服务读网卡并调用read时间++服务器处理数据时间+服务端调用write并写网卡时间+客户端读网卡并调用read时间。这其中除了网络开销，花费时间最长的就是 read 和 write 调用了。这一操作需要操作系统用户态切换到内核态。使用管道时候，多个命令只会一次 read 和 write 操作。因此使用管道会提升 Redis 服务器处理命令的速度，最后会趋近于 10 倍的速度。Redis 2.6 版本以后，脚本的命令会优于管道。
可以通过这个工具 redis-benchmark 来测试，而且管道不保证原子性，如果其中一个指令失败了，后面命令继续运行，类似于批处理模式。

11. lua script
redis 会单线程原子性的执行 lua 脚本，不受干扰。比如分布式锁中，def_if_equals 命令。而且 redis server 可以保存脚本，以免频繁上传脚本。

11. redis pipeline lua script 区别？
当多个 redis 命令没有依赖、顺序关系，建议使用 pipline；当多个命令有依赖和顺序关系，用 lua  script。
redis.pcall() 会保护机制，如果出现异常，不会再继续执行，会返回客户端错误信息。





例子：实现文章的自增ID？
INCR命令来完成
注意：在业务使用当中，在Redis实例中会形成很大的对象，比如一个大的hash、zset、list等，这样的对象会Redis的集群数据迁移带来了大的问题，因为在迁移环境下，如果某个key过大，会导致数据迁移困难。另外在内存分批上，如果一个key过大，那么当它需要扩容时，会一次性申请更大的一块内存，也会导致卡顿，如果这个大key会被删除，内存会一次性回收。
所以在平时的业务当中，尽量避免使用大的key，如果线上有，可以通过scan指令，扫描出每一个key，会用type获取类型，然后使用相应的数据结构获取其长度。上面的过程需要编写脚本，但是redis-cli官方提供下面命令，可以直接用：
```
redis-cli -h 127.0.0.1 -p 7001 –-bigkeys
# Scanning the entire keyspace to find biggest keys as well as
# average sizes per key type.  You can use -i 0.1 to sleep 0.1 sec
# per 100 SCAN commands (not usually needed).

[00.00%] Biggest string found so far 'wsg_uv' with 57 bytes
[00.00%] Biggest string found so far 'a' with 15433 bytes
[00.00%] Biggest string found so far 'b' with 29321 bytes
[45.45%] Biggest zset   found so far 'total' with 4 members
[45.45%] Biggest hash   found so far 'counter' with 4 fields

-------- summary -------

Sampled 22 keys in the keyspace!
Total key length in bytes is 114 (avg len 5.18)

Biggest string found 'b' has 29321 bytes
Biggest   hash found 'counter' has 4 fields
Biggest   zset found 'total' has 4 members

20 strings with 44911 bytes (90.91% of keys, avg size 2245.55)
0 lists with 0 items (00.00% of keys, avg size 0.00)
0 sets with 0 members (00.00% of keys, avg size 0.00)
1 hashs with 4 fields (04.55% of keys, avg size 4.00)
1 zsets with 4 members (04.55% of keys, avg size 4.00)
```
上述这个指令会大幅提升Redis的ops导致线上报警，可以增加一个休眠参数：
redis-cli -h 127.0.0.1 -p 7001 –-bigkeys -i 0.1

_提示：ops是指每秒操作数_


# 使用Redis构建Web应用
## 登录和cookie缓存
对于用来登录的cookie，有两种常见的方法可以将登录信息存储在cookie里面：一种是签名signed cookie，另一种是令牌token cookie。
* 签名cookie通常会存储用户名、用户ID以及用户相关信息。除了用户相关信息外，还有一个cookie的签名（可以使用这个签名来验证浏览器发送的消息是否未经改动）
* 令牌cookie会在cookie里面存储一串随机字节作为令牌，服务器可以根据令牌在数据库中查找令牌的拥有者
![image](https://user-images.githubusercontent.com/12162133/42333419-450f9cba-80ad-11e8-8d8e-85838e7da212.png)

普通关系型数据库在每台数据库服务器上面每秒也只能插入、更新或者删除200~2000个数据行。高并发的情况下，使用Redis往往比关系型数据库性能更好。
用HASH存储cookie令牌和已登录用户之间的一一映射关系，使用redis存储用户信息，有效的缓解数据库的压力。
## 实现购物车
使用cookie实现购物车———也就是将整个购物车的信息存储在cookie里面很常见。这一种做法的一大优点就是无须对数据库进行写入就可以使用购物车功能，而缺点就是程序需要重新解析和验证cookie，确保cookie的格式正确。cookie购物车还有一个缺点，因为浏览器每次发送请求都会连cookie一起发生，如果cookie购物车体积比较大，那么请求发送和处理的速度可能会有所降低。同理，购物车的信息可以存储在redis里面。
## 网页缓存
在动态生成网页的时候，通常会使用模板语言来简化网页的生成操作，现在的Web网页通常由首部、尾部、侧栏菜单、工具条、内容域的模板生成。而这些内容通常不会改变。对于可以被缓存的请求，函数首先会尝试从缓存里面取出并返回被缓存的页面，如果页面不存在，那么函数会生成页面并将其缓存在Redis里面5分钟。
## 数据行缓存
即使是那些无法被整个缓存起来的页面-比如账号页面、记录用户以往购买商品的页面等等，程序也可以通过缓存页面载入时所需的数据行来减少载入页面所需的时间。
使用JSON格式而不是其他格式：因为JSON简明易懂，并且Redis客户端语言都带有能够高效地编码和解码JSON格式的函数库。
# Redis命令
## 字符串
字符串可以存储以下3种类型，字节串、整数、浮点数。
INCR DECR INCRBY DECRBY  只能操作整数
INCRBYFLOAT  既能操作整数，又能操作浮点数
Redis会将这个值解释成十进制整数或者浮点数，如果用户对一个不存在的键或者一个保存为空串的键执行自增或自减操作，那么Redis在执行操作时会将这个键值当作是0来处理。
APPEND key-name  value 将值value追加到key-name后面
GETRANGE key-name start end 获取一个由偏移量start至end范围内所有的子串，超出字符串末尾的数据被视为空串
GETBIT  key-name offset 将字节串看作是一个二进制位串，并返回串中偏移量为offset的二进制位的值
BITCOUNT key-name [start end] 统计一个字符串中二进制位串里面值为1的二进制位的数量 
**Redis字符串是动态的字符串，是可以修改的字符串，内部结构类似于Java的ArrayList，采用了预分配冗余空间的方式来减少内存，当字符串长度小于1M时，扩容都是扩到当前长度一倍，如果超过1M，扩容的时候最多增加1M的空间。**
## 列表
Redis列表允许用户从序列的两端推入或者弹出元素。
LTRIM key-name start end 对列表中的元素进行修剪，只保留从start偏移量到end偏移量范围内的元素，其中偏移量start和end也包含在内。
有几个列表命令可以将元素从一个列表移动到另外一个列表，或者阻塞（block）执行命令的客户端直到有其他客户端给列表添加元素为止，命令如下：
BLPOP key-name [key-name ……] timeout  从第一个非空列表中弹出位于最左端的元素，或者在timeout秒内阻塞并等待可弹出的元素出现。同理BRPOP从最右端阻塞弹出元素。
RPOPLPUSH source-key dest-key 从source-key列表中弹出位于最右端的元素然后将这个元素推入dest-key列表的最左端，并向用户返回这个元素。
BRPOPLPUSH source-key dest-key timeout 从source-key列表中弹出位于最右端的元素，然后将这个元素推入到dest-key列表的最左端，并向用户返回这个元素；如果source-key为空，那么在timeout秒之内阻塞并等待可弹出的元素出现。
**对于阻塞弹出命令和弹出并推入命令，主要用于就是消息传递和任务队列**
Redis里面的列表相当于Java里面的链表，这意味着列表的插入和删除操作非常快，时间复杂度为O(1)，但是索引定位很慢，时间复杂度为O(n)。
ltrim命令和字面上不一样，ltrim保留一个区间内的值。lrange 0 -1获取所有元素。
但是你会发现Redis实现列表的底层不是一个简单的LinkedList，而是称为快速链表quicklist的一个结构。首先在元素很少的情况下，是一块压缩列表，分配的一块连续的内存。当数据量比较多的时候才会变成quicklist。因为普通的链表需要的附加指针空间太大，会比较浪费空间，而且会增加内存的碎片，所以Redis将链表和ziplist结合起来组成了quicklist，这样既能满足快速插入删除性能，又不会出现太多空间的冗余。
## 集合
SCARD key-name 返回集合中包含元素的数量
SRANDERMEMBER key-name count 从集合里面随机的返回一个或多个元素。当count为正数时，命令返回的随机数不会重复，为负数，可能会出现重复。
SDIFF key-name [key-name ……] 返回哪些存在第一个集合，但不存在于其他集合的元素。
SINTER key-name [key-name……] 交集
SUNION 并集
Redis里面集合就相当于Java语言里面的HashSet，它内部键值对是无序的且唯一的，它的内部实现相当于一个特殊的字典，字典中的值都是一个NULL值。
## 散列
HMGET HMSET 可以一次获取或设置多个键值对
HLENG 返回键值对的长度
HEXISTS key-name key 检查散列中是否存在该键
HKEYS key-name 获取散列包含的所有键
HVALS key-name 获取散列包含的所有值
HGETALL key-name 获取散列包含的所有键值对
HINCRBY key-name key increment 
HINCRBYFLOAT
内部实现结构上和Java的HashMap一样的，同样的数组+链表结构，不同的是Redis的字典只能是字符串，并且它们的rehash方式也不一样，因为Java的HashMap在字典很大的时候，rehash是非常耗时的操作，Redis为了提高性能，不能堵塞服务，所以采用了渐进式rehash。渐进式rehash会在rehash同时，保留旧新两个hash的结构，查询时会同时查询两个hash结构，然后在后续的定时任务以及hash操作指令中，循序渐进地将旧的hash内容迁移到新的hash结构中，当搬迁完成了，就会用新的hash结构取而代之。当hash移除了最后一个元素后，该数据结构就会被自动删除。
## 有序集合
ZCARD key-name  返回有序集合包含的成员数量
ZCOUNT key-name min max 返回分支介于min和max之间的成员数量
![image](https://user-images.githubusercontent.com/12162133/42412467-5fb095fe-823f-11e8-9479-92d26ce55151.png)
![image](https://user-images.githubusercontent.com/12162133/42412486-c8b5bb56-823f-11e8-90fe-5499ad2c5655.png)
有序集合的过程，这次交集运算使用的是默认的聚合函数sum，所以输出有序集合成员的分值都是通过加法计算得出的。
![image](https://user-images.githubusercontent.com/12162133/42412492-0bd6ba98-8240-11e8-93c6-ba9729bc6f8b.png)
使用聚合函数min执行并集运算的过程。
zset的内部是采用跳跃列表数据结构来实现的，它的结构非常特殊，也很复杂。
# 发布与订阅
pub/sub 发布与订阅的特点是订阅者（listener）负责订阅频道（channel），发送者（publisher）负责向频道发生二进制字符串消息（binary string message）。
SUBSCRIBE channel [channel……] 订阅给一个或多个频道
UNSUBSCRIBE [channel [channel……]] 退订一个或多个频道，如果执行时没有给定任何频道，那么退订所有频道。
PUBLISH channel message 向给定频道发送消息
PSUBSCRIBE pattern [pattern ……] 订阅给定模式相匹配的所有频道
PUNSUBSCRIBE
虽然Redis的发布/订阅模式非常有用，但是不好的地方有也两点：
1. 读取消息速度不够快的话，就会不断地累计消息使得Redis体积越来越大，这可能导致Redis速度变慢或者导致Redis操作系统被强制杀死（后续Redis会修复这个问题）
2. 依靠频道来接收消息的用户可能会对Redis提供的PUBLISH和SUBSCRIBE感到失望，因为由可能客户端断网会导致丢失消息
# 其他命令
1. 排序
SORT命令类似于数据库的order by语句
SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC | DESC] [ALPHA] [STORE destination]
SORT key-name 默认按照数字排序
SORT key-name ALPHA 按照字符排序
ASC|DESC 升降排序
LIMIT offset count 限制返回数据行
BY 可以根据某个字段进行排序
STORE 可以存储排序结果
2. 基本的Redis事务
Redis有5个命令可以在用户在不被打断的情况下对多个键执行操作，WATCH MULTI EXEC UNWATCH DISCARD。
Redis的基本事务需要用到MULTI命令和EXEC命令，这种事务可以让一个客户端在不被其他客户端打断的情况下执行多个命令，和数据库不同的是，在Redis里面，被MULTI命令和EXEC命令包围的所有命令会一个一个的执行，直到所有命令都执行完毕。当一个事务执行完毕后，Redis才会处理其他客户端的请求。
当Redis收到一个客户端那里接收到的MULTI命令时，Redis会将这个客户端后面发送的所有命令都放入一个队里里面，直到这个客户端发生EXEC为止，然后Redis就会在不被打断的情况下，一个一个地执行存储在队列里面的命令。
3. 键的过期时间
DEL删除数据或者设置Redis的过期时间，当我们说一个键过期时，指的是Redis会在这个键的过期时间到达时自动删除该键。
键过期时间只能对整个键设置过期时间，而没办法对键里面的单个元素设置过期时间（为了解决这个问题，使用存储时间戳的有序集合来实现单个元素的过期操作）
# 数据安全与性能保障
## 持久化选项
快照，它可以将存在于某一时刻的所有数据都写入硬盘里面。 AOF（只追加文件）：它会在执行写命令时，将被执行的写命令复制到硬盘。这两种持久化方式可以同时使用，又可以单独使用。甚至某些情况下可以两种都不用。
## 快照
Redis可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本，创建快照后，用户可以对快照进行**备份**，可以将快照复制到其他服务器上从而创建相同的数据的服务器副本。还可以将快照留在本地作为重启服务器使用。
根据配置，快照被写入dbfilename选项指定的文件里面，并存储在dir选项指定的路径上面（如果在新的依次快照文件创建过程中，Redis、系统或者硬件三者之中任意一个发生故障，那么Redis将会丢失最近一次快照存储的文件）
创建快照的方法如下：
1. 客户端通过向Redis发送BGSAVE命令来创建一个快照。WINDOWS平台不支持，Redis会调用一个fork来创建一个子进程，然后子进程负责将快照写入硬盘，而父进程则会继续处理命令请求。
2. 客户端通过Redis发送SAVE命令，接收到SAVE命令的Redis服务器会在创建快照完毕之前将不再响应任何的其他命令。SAVE命令不常用，只会在没有足够内存的情况下才会执行此命令
3. 用户设置了save选项，比如save 60 10000，当60s之内有10000次写入，Redis就会自动触发BGSAVE命令
4. 当Redis通过SHUTDOWN命令接收到关闭服务器的请求时候，或者接收到标准的TERM信号时，会执行一个SAVE命令，在SAVE命令执行完毕后关闭服务器
5. 当一个Redis服务器连接另一个Redis服务器，并向对方发了一次SYNC命令来开始一次复制的操作

在客户端执行 BGSAVE 命令的同时，客户端检验后台是否执行成功，可以用 LASTSAVE 命令，返回最后一次保存成功的时间戳。原理是利用 Linux 写时拷贝技术（copy on write）。
![image](https://user-images.githubusercontent.com/12162133/64873986-c6234600-d67c-11e9-8cdd-1948e6089d8a.png)
Linux的fork()使用写时拷贝（copy-on-write）页实现。写时拷贝是一种可以推迟甚至免除拷贝数据的技术。内核此时并不复制整个进程地址空间，而是让父进程和子进程共享同一个拷贝。只有在需要写入的时候，数据才会被复制，从而使各个进程拥有各自的拷贝。也就是说，资源的复制只有在需要写入的时候才进行，在此之前，只是以只读方式共享。这种技术使地址空间上的页的拷贝被推迟到实际发生写入的时候。
Redis 可以配置自动生成快照,在配置文件中配置即可,当执行多少次写操作,触发一次 BGSAVE 命令，Redis 在使用 BGSAVE 命令生成快照时，仍然会对外提供服务，因为使用了 COW 技术，Redis 使用快照技术是精确的。但是 save 命令会阻塞当前 Redis 服务器，直到持久化完成。
子进程做数据持久化，它不会修改现有的内存数据结构，它只是对数据结构进行遍历读取，然后序列化写到磁盘中。但是父进程不一样，它必须持续服务客户端请求，然后对内存数据结构进行不间断的修改。
这个时候就会使用操作系统的 COW 机制来进行数据段页面的分离。数据段是由很多操作系统的页面组合而成，当父进程对其中一个页面的数据进行修改时，会将被共享的页面复制一份分离出来，然后对这个复制的页面进行修改。这时子进程相应的页面是没有变化的，还是进程产生时那一瞬间的数据。
随着父进程修改操作的持续进行，越来越多的共享页面被分离出来，内存就会持续增长。但是也不会超过原有数据内存的 2 倍大小。另外一个 Redis 实例里冷数据占的比例往往是比较高的，所以很少会出现所有的页面都会被分离，被分离的往往只有其中一部分页面。每个页面的大小只有 4K，一个 Redis 实例里面一般都会有成千上万的页面。
子进程因为数据没有变化，它能看到的内存里的数据在进程产生的一瞬间就凝固了，再也不会改变，这也是为什么 Redis 的持久化叫「快照」的原因。接下来子进程就可以非常安心的遍历数据了进行序列化写磁盘了。


**注意：在使用Redis快照的时候，一定要记住，在系统发生奔溃的时候，用户将丢失最近一次生成快照之后更改的所有数据。**
## AOF持久化
AOF持久化将被执行的写命令写到AOF文件的末尾，依次来记录数据的变化。因此，Redis只要从头到尾执行一次AOF的包含的所有写命令，就可以恢复AOF文件所记录的数据集。
<table>
<tr>
<td>选项</td>
<td>同步频率</td>
</tr>
<tr>
<td>always</td>
<td>每个Redis命令都要同步写入磁盘，这样做会严重影响Redis性能</td>
</tr>
<tr>
<td>everysec</td>
<td>每秒执行一次同步，显示地将多个写命令同步到磁盘</td>
</tr>
<tr>
<td>no</td>
<td>让操作系统来决定何时同步</td>
</tr>
</table>

文件同步的时候，当调用file.write()命令方法对文件进行写入的时候，写入的内容首先会被存储到缓冲区，然后操作系统会在将来的某个时候将缓冲区的内容写入硬盘，而数据只有在被写入硬盘后，才算真正的持久化硬盘，用户可以通过调用file.flush()方法请求操作系统将缓冲区的数据刷入硬盘，但是何时写入仍然由操作系统决定。用户也可以使用sync同步命令写入磁盘。
1. 转盘式硬盘：每秒处理200个写命令
2. 固态硬盘：每秒处理大概几万个命令

虽然AOF持久化灵活地提供了多种不同的选项来满足不同应用程序对数据安全的不同要求，但AOF持久化也有缺陷—那就是AOF文件的体积大小。
为了解决AOF文件体积的大小问题，用户可以向Redis发送BGREWRITAOF命令，这个命令通过移除AOF文件中冗余命令来重写AOF文件。其原理就是开辟一个子进程对内存进行遍历，序列化到一个新的AOF日志文件。
AOF 整个流程可以分为两步，一步命令实时写入，第二步是对 AOF 文件重写。第一步主要流程是：命令写入->追加到 AOF_BUF -> 同步到 AOF 磁盘。如果是实时写入磁盘会带来很高的磁盘 IO，影响整体性能。
手动触发：通过命令 bgrewriteaof；自动触发：通过规则配置触发。
通过使用AOF持久化或者快照持久化，用户可以在系统重启或奔溃的情况下仍然保留数据，但是随着负载量的上升，或者数据的完整性变得越来越重要，用户可能通过使用复制特性。

RDB 只是一个文件，非常适合用于备份，适合用于灾难恢复，RDB 会在服务器故障时丢数据，每次保存 RDB 的同时，都需要 fork 一个子进程，fork 可能会非常耗时，如果数据集非常大，且 CPU 资源紧张，则会导致服务器停顿，虽然 AOF 重写也需要进行 fork，但是停顿时间会少一点。
AOF 可以设置每秒执行一次 fsync 操作，在这种配置下 Redis 会保持很好的性能，并且就算发生故障，也最多会丢失一秒的数据。而且还自带 redis-check-aof   工具可以修复一些写入不完整的命令，遇到 AOF 文件重写时的问题。当然你也可以同时使用这两种方式。
Redis 4.0 带来一个非常好的措施，混合持久化方式，将 rdb 文件和 aof 混合使用。这里 aof 文件不再是全量的日志，而是由持久化开始到持久化结束的这段时间发生的增量 aof 日志。
![image](https://user-images.githubusercontent.com/12162133/64905242-e9470780-d707-11e9-8585-b1c0dc1c6fd6.png)
于是在 Redis 重启的时候，可以先加载 rdb 的内容，然后再重放增量 aof 日志。那么 redis 适合用来做数据库吗？
1.成本-首先成本原因，因为Redis是内存数据库，而内存相对于磁盘来说虽然性能很高，但是成本更高，在做大量数据存储时成本问题尤为突出
2.持久性-Redis数据丢失率，上面看到不管是RDB还是AOF都会进行IO操作，那么在持久化过程为了提高性能就需要降低保存频率，那么必然会有数据丢失的情况，无法保证数据的完整性
3.查询特性-Redis的数据特性导致查询功能特性较少，而数据库一般需要支持多样查询，redis无法适用这样的场景
4.事务特性：redis不具备事务特性，尤其是在分布式环境下尤为特出
# 复制
可以让其他服务器拥有一个不断地更新的数据副本，从而使得拥有数据副本的服务器可以用于处理客户端发送的读请求。关系型数据库通常是一个主服务器向其他从服务器发送更新，并使用从服务器来处理所有读请求，Redis也是采用同样的方法来实现自己的复制特性。客户端每次向主服务器进行写入时，从服务器都会实时地更新。
## Redis主从结构搭建
Redis主从结构不需要安装多个，只需要复制多个配置文件即可。然后分别运行即可，下面展示的是单机伪主从结构配置如下：
1. master
```
# 监听端口
port 6379
# 以守护进程在后台执行
daemonize yes
# 以守护进程启动，pid信息放到此文件里面
pidfile /var/run/redis_6379.pid
# 日志文件
logfile /Users/a1/local/logs/redis/master.log
# 数据库指定的文件
dbfilename master-dump.rdb
# 文件目录
dir /usr/local/var/db/redis/
# 设置Redis连接的密码
requirepass 123456
```
2. slave1
```
# 监听端口
port 6380
# 以守护进程在后台执行
daemonize yes
# 以守护进程启动，pid信息放到此文件里面
pidfile /var/run/redis_6380.pid
# 日志文件
logfile /Users/a1/local/logs/redis/slave.log
# 数据库指定的文件
dbfilename slave-dump.rdb
# 文件目录
dir /usr/local/var/db/redis/
# 当master服务器设置了密码，slave服务连接密码
masterauth 123456
# 设置绑定主服务器IP和端口
slaveof 127.0.0.1 6379
```
>redis-cli -p 6379 -a 123456        #连接主服务器
## 复制配置
当从服务器连接主服务器的时候，主服务器会执行BGSAVE操作，因此为了正确的使用复制特性，主服务器需设置dir和dbfilename选项，并且这两个选项所指示的路径和文件对于Redis进程来说是可写的。
**对于一个正在运行的Redis服务器来说，用户可以发生slaveof no one命令来让服务器终止复制操作。也可以通过发生slaveof host port命令来让服务器开始复制一个新的主服务器。**
## 复制启动过程
![image](https://user-images.githubusercontent.com/12162133/42637039-c0c74afa-861c-11e8-8ed2-2f904016b5f4.png)
在实际中，最好让主服务器只使用50%~65%的内存，留下30%~45%的内存用于执行BGSAVE操作和创建记录写命令的缓冲区。
注意：从服务器在进行同步时，会清空自己所有的数据，并被替换成主服务器发来的数据。Redis是不支持主主服务器复制的。
### 旧版主从复制过程
Redis2.8之前，当客户端向服务端发送SLAVEOF命令时，要求从服务器复制主服务器，也即是，将从服务器的数据库状态更新至主服务器当前所处的数据状态。
1. 同步
（1）从服务器向主服务器发送**SYNC**命令
（2）收到**SYNC**命令后，主服务器执行**BGSAVE**命令，在后台生成一个**RDB**文件，并使用一个缓冲区记录从现在开始执行的所有写命令
（3）当主服务器的**BGSAVE**执行完毕后，主服务器会将生成的**RDB**文件发送给从服务器，从服务器接收并载入**RDB**文件，将自己的数据库状态更新至主服务器执行**BGSAVE**命令时的数据库状态。
（4）主服务器将记录在缓冲区里面所有命令发送给从服务器，从服务器执行这些命令，将自己的数据库更新至主服务器数据库当前的状态
2. 命令传播
在执行完同步操作后，主从服务器之间的数据库状态已经相同了。但这个状态会随着主服务器执行了写操作，会变化，导致主从服务器状态不再一致，所以，为了让主从服务器再次回到一致的状态，主服务器对从服务器执行了命令传播机制，主服务器将自己写的命令，发送给从服务器执行。
3. 缺陷
初次复制，上述步骤能很好的完成任务。但对于那种断网后重连的复制来说，上述步骤虽然也能让主从服务器回到一致状态，但效率非常低。从服务器重新连接上主服务器，所以从服务器向主服务器发送**SYNC**命令。

### 新版主从复制过程
为了解决旧版复制功能在处于断线重复复制情况的低效问题，Redis从2.8版本开始，使用PSYNC命令代替SYNC命令来复制的时候，PSYNC命令具有完整重同步（full resynchronization）和部分重同步（partial 
 resynchronization）两种模式：
- 完整重同步用于处理初次复制情况：完整重同步和SYNC一样
- 而部分重同步用于处理断线后重复制情况：当从服务器在断线后连接主服务器时，如果条件允许，主服务器可以将主从服务器连接断开期间执行的写命令发送给从服务器，从服务器只要接收并执行这些命令，就可以将数据库更新至主服务器当时所处的状态。
## 主从链
创建多个从服务器可能会造成网络不可用—当复制需要互联网进行或者在不同的数据中心之间进行时，尤为如此。因为Redis主服务器和从服务器没有不同的地方，所以从服务器也可以拥有自己的从服务器，并由此形成主从链。
当读请求的重要性明显高于写请求的重要性，并且读请求的数量远远超出一台Redis服务器可以处理的范围时。随着负载的不断上升，主服务器可能会无法快速地更新所有从服务器，或者因为重新连接和重新同步而导致系统超载。为了解决这个问题，用户可以创建一个Redis主从节点组成的中间层来分担主服务器的复制工作。

## 建议硬盘写入
为了验证主服务器是否已经将写数据发生至从服务器，用户需要在向主服务器写入真正的数据之后，再向主服务器写入一个唯一的虚拟值（unique dummy value）。然后检查虚拟之是否存在于从服务器就可以了。
另一方面，判断数据是否落到硬盘，检查INFO命令的输出结果aof_pending_bio_fsync属性的值是否为0。
INFO命令功能如下:
- server: General information about the Redis server
- clients: Client connections section
- memory: Memory consumption related information
- persistence: RDB and AOF related information
- stats: General statistics
- replication: Master/slave replication information
- cpu: CPU consumption statistics
- commandstats: Redis command statistics
- cluster: Redis Cluster section
- keyspace: Database related statistics
详情：https://redis.io/commands/info

# 处理系统故障
## 验证快照或AOF文件
快照持久化或AOF持久化，都提供了遇到故障时的进行数据修复的工具。Redis提供了两个命令：redis-check-aof和redis-check dump。
```
redis-check-aof --fix          
```
程序会对AOF文件进行修复，寻找不正确或者不完整的命令，发现第一个出错的命令的时候，程序会删除出错的命令以及位于出错命令之后的所有命令。在大多数情况下，程序都会删除AOF文件末尾的不完整的命令。
因为快照文件经过压缩，所以快照文件没有办法修复，通过计算快照文件的SHA1和SHA256散列值对内容进行验证。
## 更换故障服务器
```
// B服务器
127.0.0.1:6379> SAVE
OK
127.0.0.1:6379> QUIT
$ scp dump.rdb rhost            --上传文件到远程服务器C上
127.0.0.1:6379>SLAVEOF machine-c.vpn 6379          --告知机器B的Redis，让它将机器C作为新的主服务器
```
提示：Redis Sentinel 可以监视指定的Redis主服务器以及其属下的从服务器，并主从服务器下线时自动进行故障转移（fail over）。
#Redis事务
## 事务命令
Redis在执行事务过程中，会延迟执行已入队的队列的命令直到客户端发送EXEC命令为止。这种一次性发送多个命令，然后等待所有回复出现的做法称为“流水线”，它可以减少客户端与Redis服务器之间的网络通信次数来提升Redis执行多个命令的性能。
MULTI、EXEC、DISCARD、WATCH命令是Redis的事务功能的基础。Redis允许一次执行一组命令。Redis会将一个事务中的命令序列化，然后按顺序执行，Redis不会在执行一个事务的过程中再执行另外一个客户端的请求，在一个Redis事务中，要么执行其中所有的命令，要么都不执行。因此，Redis事务能保证其原子性，EXEC命令会触发执行事务中所有的命令。
当客户端执行一个事务的时候，如果它在调用MULTI之前就和服务器断开连接，那么就不会执行事务中的所有操作；如果在执行EXEC命令之后和服务器断开连接，则会执行事务中的所有操作；当Redis使用AOF方式时，Redis能够确保使用一个单独的write系统调用，这样便能够将事务写入磁盘；如果Redis服务器宕机，或者Redis服务器停止运行，那么Redis很有可能只执行事务中的一部分操作，Redis将会在重启的时候检查上述状态，然后退出运行，并且输出报错信息，使用redis-check-aof工具可以修复上述的只增文件，这个工具将会从上述文件中删除执行不完全的事务，这样Redis服务器才能再次启动。
- MULTI
标记事务开始，这个命令返回值是一个简单的字符串，OK
- EXEC
在一个事务中执行先前放入的所有命令，然后恢复正常连接；当使用WATCH命令的时候，只有当受监控的键没有被修改的时，EXEC才会执行事务中的命令，这种利用CAS的机制；EXEC命令的返回值是一个数组，其中每个元素分别是原子化事务中每个命令的返回值。当使用WATCH命令时，如果事务执行中止，那么EXEC命令就会返回一个NULL值。
- DISCARD
清除在一个事务中放入队列的所有命令，然后恢复正常的连接状态，如果使用WATCH命令，那么DISCARD命令就会将当前连接监控的所有键取消监控。
- WATCH
当某个事务需要执行条件时，就要使用命令将给定的键设置为受监控的，对于每个键来说，时间复杂度为o(1)。
## 事务内部错误
一个命令可能会存在放入队列时失败，事务可能在调用EXEC命令之前发生错误，可能会有语法问题，或者可能是某些临界条件，或者可能是对某个键执行了错误类型的操作。
- <Redis 2.6.5，服务器会记住事务累计命令期间的错误，然后Redis会拒绝执行这一事务
- >=Redis 2.6.5，如果发生上述错误，那么客户端在调用EXEC命令之后，Redis还会执行这个出错的事务，执行已经成功放入事务的命令，忽略错误的事务命令。
## 不支持回滚
实际上，只有程序错误才会导致Redis命令执行失败，这种错误可能会在程序开发期间发现，一般很少在生产环境发现，Redis已经在事务内部进行功能简化，这样可以确保更快的运行速度，因此Redis不需要事务回滚的能力。
## demo
- 普通的事务执行
```
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> incr f
QUEUED
127.0.0.1:6379> incr f
QUEUED
127.0.0.1:6379> get f
QUEUED
127.0.0.1:6379> exec
1) (integer) 1
2) (integer) 2
3) "2"
127.0.0.1:6379>
```
- WATCH的应用
Redis已经提供了基于incr命令来实现一个整型数值的原子递增操作，但是如果没有这样的命令我们该如何实现这样的操作呢？
```
127.0.0.1:6379> WATCH incr:key
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set incr:key 2
QUEUED
127.0.0.1:6379> EXEC
1) OK
127.0.0.1:6379> get incr:key
"2"
```
```
127.0.0.1:6379> WATCH incr:key
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> set incr:key 3
QUEUED
127.0.0.1:6379> EXEC
(nil)
```
上述两个例子分举例运用WATCH命令后成功的事务和失败的事务。
# 非事务型流水线
使用事务的一个好处就是底层的客户端会通过使用流水线来提高事务执行时的性能，我们要如何不使用事务，通过使用流水线来提升Redis的性能的:
```
pipe = coon..pipeline()
```
如果用户在执行pipeline的时候传入TRUE参数，那么客户端将使用MULTI和EXEC包裹用户的要执行的命令；如果传入FALSE参数，客户端同样会像执行事务那样包裹用户将要执行的所有命令，只是不再使用MULTI和EXEC包裹这些命令。通过将标准的Redis连接替换成流水线连接，程序可以将通信往返次数减少至原来的1/2~1/5。

Redis cluster 如何支持 pipeline 命令呢？

# 性能方面
通常Redis带来了上百倍的提升相对于关系型数据库，但是Redis的性能还是可以进一步的提升，性能优化前需要知道Redis命令到底能跑多快，而这一点可以通过Redis附带的性能测试工具redis-benchmark来了解Redis在自己服务器上的各种性能特征。
```
$ redis-benchmark -c 1 -q
PING_INLINE: 19316.21 requests per second
PING_BULK: 19168.10 requests per second
SET: 19611.69 requests per second
GET: 20785.70 requests per second
INCR: 19719.98 requests per second
LPUSH: 20593.08 requests per second
RPUSH: 20673.97 requests per second
LPOP: 21217.91 requests per second
RPOP: 20678.25 requests per second
SADD: 19600.16 requests per second
HSET: 19361.08 requests per second
SPOP: 22573.37 requests per second
LPUSH (needed to benchmark LRANGE): 24777.01 requests per second
LRANGE_100 (first 100 elements): 23889.15 requests per second
LRANGE_300 (first 300 elements): 24497.80 requests per second
LRANGE_500 (first 450 elements): 19755.04 requests per second
LRANGE_600 (first 600 elements): 18625.44 requests per second
MSET (10 keys): 18782.87 requests per second
```
在一般情况下，一个不是用流水线的客户端性能大概只有redis-benchmark所示性能的50%~60%。
![image](https://user-images.githubusercontent.com/12162133/45157243-233b1f00-b213-11e8-82a3-8f04da575c54.png)
# Redis构建支持程序
## Redis构建日志
LPUSH记录日志到队列中，LRANGE命令拉取列表中的消息
## 计数器
为了对计数器进行更新，我们需要存储实际的的计数信息，我们将用一个散列来存储网站在每个5s的时间片之内获得点击量
```
127.0.0.1:6379> HGETALL counter
1) "123123123123"
2) "5"
3) "234234234234"
4) "6"
5) "234322342343"
6) "6"
7) "2342323423434"
8) "10"
```
随着时间的迁移，为了能够清理计数器的值，我们需要使用计数器的同时，对被使用的计数器进行记录。为了做到这一点，我们需要一个有序序列。
## 统计器
对于给定一个上下文，程序将使用一个有序集合来记录这个上下文以及这个类型的最小值、最大值、平均值、和等，用有序集合记录每个时间段的数据，时间作为score，统计值作为key，在高并发的情况下性能是满足的。
```
127.0.0.1:6379> ZADD total 1536283150000 1
(integer) 1
127.0.0.1:6379> ZADD total 1536283160000 5
(integer) 1
127.0.0.1:6379> ZADD total 1536283170000 3
(integer) 1
127.0.0.1:6379> ZADD total 1536283180000 4
(integer) 1
127.0.0.1:6379> ZRANGE total 0 10 WITHSCORES
1) "1"
2) "1536283150000"
3) "5"
4) "1536283160000"
5) "3"
6) "1536283170000"
7) "4"
8) "1536283180000"
```
## 服务发现与配置
服务发现与配置，它的原理其实很简单。服务提供者，简单来说就是一个HTTP服务，提供了API服务，包含一个IP和端口；服务消费者，需要请求服务提供者提供的服务。现实情况下，一个HTTP服务既是服务提供者，又是服务消费者。
服务发现有三个角色，服务提供者、服务消费者、服务中间件。服务中间件是服务提供者和服务消费者的桥梁，服务提供者将自己的服务地址注册到服务中间件上，服务消费者从服务中间件上查找自己需要的服务地址，然后使用这个服务。
服务中间件，简单来说，就是一个字典，字典里面有许多key/value，key是服务名称，value是服务提供者地址列表，服务注册就是PUT地址，服务查找就是GET地址。当服务提供者有节点挂掉后，要求服务能够及时取消，当服务提供者有节点注册后，要求服务中间件能够及时添加，以便让服务消费者及时使用服务。
Redis作为服务的中间件，因为Redis具有丰富的结构，用set作为结构存储服务地址列表，如果服务提供者节点注册，则用SADD命令添加，如果服务提供者节点挂掉，则用SREM命令移除服务地址。
但是，做好上面几点，并不能完全的完成服务注册，第一个问题就是服务提供者节点进程挂掉后，不能主动调用SREM命令，这个时候服务中间件中就有一个黑地址，这个时候我们需要引入保活和检查机制，服务提供者需要每隔5s向服务中间件汇报存活，服务中间件会将服务地址和汇报时间记录在zset数据结构的value和score中。服务中间件需要每10s左右检查zset的数据结构，需要踢掉汇报时间落后的服务地址列表。这样可以准确实时地保证服务列表地址的有效性。第二个问题就是服务列表变动是如何通知服务消费者，有两种解决方案，第一种是轮询，消费者需要每隔几秒查询服务列表是否有改变，当然如果服务很多，消费者很多，对Redis有一定的压力，所以这个时候需要引入服务列表的版本号，给每个服务设置一个服务版本号，只有在服务发生变动的时候，递增这个版本号，消费者只需要轮询这个版本号就知道这个服务有没有变更。第二种是采用pubsub，这种方式的及时性要优于轮询，缺点就是会占用消费者一个线程和一个额为的Redis连接，为了减少对线程和连接的浪费，我们使用单个pubsub广播全局版本号的的变动。所谓全局版本号就是任意服务列表发生了变动，这个版本号就会递增。接收到版本变动的消费者再去检查各自的依赖服务列表版本号是否发生了变动。这种全局版本号也可用于第一种轮询方案。第三个问题就是redis是单点的，如果挂掉了怎么办，流行的服务发现都使用分布式数据库zookeeper/etcd/consul等来作为服务中间件，如果挂掉会怎样呢？其实服务消费者在本地内存里面都存放一份，那么Redis作为服务就不靠谱嘛，其实还有个redis-sentinel可以消除Redis的单点问题。

注意：其实服务提供者不仅仅是HTTP服务，还可以是数据库、RPC、UDP服务。现代的服务发现和配置还会集成统一配置管理功能。
# Redis构建应用程序组件
## 自动补全
在Web领域，自动补全是一种能让用户在不进行搜索的情况下，快速的找到所需东西的技术。因为服务器上数百万用户都要有自己的一个属于自己的联系人列表来存储最近联系过的100个人，所以我们需要能够快速向列表里面添加用户或者删除用户的前提下，尽量减少存储这些联系人列表带来的内存消耗，Redis列表可以存储最近联系人，列表占用的内存是最少的，但是Redis列表不提供过滤工作，所以过滤工作就交给客户端程序完成。
## 分布式锁
实现分布式锁通常有三种方式：数据库乐观锁、基于Redis的分布式锁、基于zookeeper的分布式锁。为了确保分布式锁的可靠性，需要满足4个条件：
- 互斥性：同一个时间点，只能有一个客户端持有锁
- 避免死锁：即使有一个客户端在持有锁期间发生奔溃而没有主动去解锁，也能保证后续其他客户端能加锁
- 容错性：只要大部分节点能正常运行，客户端就可以加锁和解锁
- 加锁解锁一致性：加锁和解锁必须是同一个客户端，客户端不能把其他客户端加的锁解除了
下面以redis的命令来展示分布式锁：
```
127.0.0.1:6379> SETNX lock:toilet 1
(integer) 1
………………do something………………
127.0.0.1:6379> del lock:toilet
(integer) 1
```
上面命令展示了，如果只有一个厕所，多个人在抢，抢的命令就是用setnx(set if not exists)，只允许被一个人抢到，谁抢到了，用完了，再调用del命令释放这个厕所，让其他人继续去抢。但是上面有一个问题，如果do something出现异常没有执行del命令，那么这个厕所就一直被占着。
于是我们再改进，拿到锁过后，就在锁上设置一个过期时间
```
127.0.0.1:6379> SETNX lock:toilet 1
(integer) 1
127.0.0.1:6379> EXPIRE lock:toilet 5
(integer) 1
………………do something………………
127.0.0.1:6379> del lock:toilet
(integer) 1
```
上面命令虽然解决了客户端没有执行del命令导致死锁问题，但是还是会出现另外一个问题，setnx和expire不是一个原子性命令，这时候会有人想到事务解决，但是在这里面不行，因为expire是依赖于setnx的结果，如果setnx没抢到锁，expire不应该执行的，Redis里面没有if-else分支的。为了解决这个问题，从Redis2.8开始作者加入了set指令的扩展参数，使得setnx和expire指令可以一起执行，彻底解决了分布式锁的乱象
```
127.0.0.1:6379> set lock:toilet 1 ex 5 nx
OK
………………do something………………
127.0.0.1:6379> del lock:toilet
```
超时问题：Redis分布式锁不能解决超时问题，如果在加锁和解锁期间逻辑执行的时间太长，以至于超出了锁的超时限制，就会出现问题，因为这个时候锁过期了，第二个线程重新持有了该锁，但是紧接着第一个线程执行了业务逻辑，就把锁给放了。
解决这个问题需要，有一个方案就是set的时候设置一个随机数字，释放锁的时候先匹配随机数，发现是否一致，然后再决定是否删除，这是为了确保当前线程的锁不会被其他线程释放。但是这个只是相对安全一点，因为如果真的超时了，其他线程的逻辑没有执行完，其他线程也会趁虚而入的。


# Redis淘汰机制
在redis里面，允许用户设置最大内存maxMemory值。如果不设置maxMemory，则表示不限制内存，如果物理内存不足，可能会使用SWAP，内存的数据会频繁的和磁盘发生交换，交换会让Redis的性能急剧下降，会让Redis基本上不可用；如果设置maxMemory，当达到maxMemory时，redis会实行数据淘汰策略，一般从redis性能来说，maxMemory一般设置物理内存的3/4最好。redis的淘汰策略如下：
- noeviction：不会继续让服务写请求（除DEL之外），读请求可以继续使用，这样可以保证不会丢失数据，但是会让线上业务不可用，会直接报 OOM 错误，默认淘汰策略
- volatile-lru：从已设置过期时间的数据集上，挑选最近最少使用的数据淘汰
- volatile-ttl：从已设置过期时间的数据集上，挑选将要过期的数据淘汰
- volatile-random：从已设置过期时间的数据集上，随机挑选数据淘汰
- allkeys-lru：从数据集中，挑选最近最少使用的数据淘汰
- allkeys-random：从数据集中，随机挑选数据淘汰

在Reidis里面，实例中的内存除了保存原始的键值对信息外，还有一些运行时候占用的内存，如下：
- 垃圾数据和过期的key占用的空间
- 字典渐进式rehash导致未删除所占用的空间
- Redis管理数据，包括底层数据结构开销，客户端信息，读写缓冲区
- 主从复制，BgSave时额外开销
- 其他
## LRU淘汰机制
在服务器中配置保存了lru计数器，
```
struct redisServer {
    ……
    unsigned int lruclock;      /* Clock for LRU eviction */
    ……
}
```
另外在每个redisObject中都有对应的一个属性lru，即最近访问的时间。每一次访问都会更新这个属性lru，LRU数据淘汰机制是这样的，从数据集中随机挑选几个键值对，取出其中lru最大的键值对淘汰，所以Redis并不是保证取得数据集中最近最少使用的键值对，而只是随机挑选的键值对作比较的。
## LRU 算法
**LRU （Least recently used，最近最少使用） 算法的设计原则：如果一个数据在最近一段时间内没有被访问，那么它在将来访问的可能性也很小。**
例：假定缓存容量为 4，顺序访问数据如下：4 3 4 1 5 2 4 1 2
4 访问内存：4
3 访问内存：3 4
4 访问内存：4 3
1 访问内存：1 4 3
5 访问内存：5 1 4 3
2 访问内存：2 5 1 4
4 访问内存：4 2 5 1
1 访问内存：1 4 2 5
2 访问内存：2 1 4 5
Java 里面实现 LRU 算法通常有两种选择，一种实现 LinkedHashMap，一种是设计数据结构，使用链表和 HashMap。 

Redis 使用的是一种近似 LRU算法，它跟 LRU 算法还不太一样。

## TTL淘汰机制
Redis数据集中保存了键值对过期时间的表，即redisDB.expires，和LRU淘汰数据机制类似，TTL淘汰数据机制是这样的，从过期时间redisDB.expires表中随机挑选几个键值对，取出其中ttl最大的键值对淘汰。同样Redis并不是保证取得所有过期时间最快过期的键值对，而只是随机挑选的键值对作比较的。
## 键的过期时间
Redis所有数据结构都可以设置过期时间，时间到了，Redis会自动删除相应的对象，需要注意的是，过期是以对象为单位，比如一个hash结构，而不是其中的某个key，还有一个需要注意的地方是如果一个字符串已经设置了过期时间，然后你调用了set方法，过期时间就会失效。

# Redis线程IO模型
Redis是一个单线程模型，除了Redis，Node.js、Nginx也是单线程，但是它们都支持高并发，性能高。Redis是单线程为什么这么快，因为所有的数据都在内存里，所有的运算都是内存级别的运算，以及Redis的内部数据结构都有优化，**但是Redis是单线程，对于那些时间复杂度为O(n)级别的指令，一定要谨慎使用，容易造成Redis卡顿**。Redis是单线程而且可以处理并发客户端操作，因为多路复用。

- 完全基于内存
- 数据结构简单，数据结构是专门设计的
- 采用单线程，避免不必要的多线程切换和竞争条件
- 使用多路复用模型，非阻塞IO

 那么 Redis 为什么是单线程的，因为 Redis 是基于内存操作的，CPU 不是内存的瓶颈，Redis 的瓶颈最有可能是机器内存的大小或者网络带宽，

## 多路复用
多路复用 IO 模型是利用 select、poll、epoll 机制的，可以同时监察多个流的 IO 事件的能力，当一个或多个流有 IO 事件时，就从阻塞状态中唤醒，于是程序就会轮询所有的流（epoll 中只轮询真正触发 IO  事件的流），并且只依次顺序的处理就绪的流。

# 缓存常见问题
## 缓存穿透
访问一个不存在的 key，流量大时，DB 会挂掉。如果有人恶意用这种一定不存在的数据来频繁请求系统（攻击系统），请求都会达到数据库层导致 DB 瘫痪。目前成熟的解决方案有两种：
- bloom filter:类似于哈希表的一种算法，用所有可能的查询条件生成一个 bitmap，在进行数据查询的时候会使用这个 bitmap，如果不在其中则直接过滤，从而解决数据库层面的压力
- 空值缓存：一种比较简单的方法，在第一次查询完不存在数据后，将该 key与对应的空值放入缓存中，只不过设定很短的失效时间，这样可以应对短时间的大量 key 攻击，设置为短时间是因为该数据与业务无关，故可以早点失效。
## 缓存击穿
一个存在的 key，在缓存过期的一刻，同时有大量请求这个 key，这些请求会击穿到 DB，造成瞬间 DB 请求量大、压力、压力骤增。
缓存击穿是缓存雪崩的一个特例，参考微博某个热门话题的功能，用户对于热门话题的搜索量通常大于其他话题，这种我们称为“热点”，在热点的缓存达到失效的时间内，此时，可能依然有很大的请求过来，这样可能会将 DB 打爆。目前成熟的方案有如下三种：
- 二级缓存：对于热点数据，进行二级缓存
- LRU：最近最少访问算法，存在一个缺点：例如，到达队列的尾部数据可能一次请求会将数据重写放到队列的头部
- LRU-K 算法：LRU-K 算法维护两个个队列或者更多队列，用于记录缓存数据被访问的历史，只有当数据访问的次数达到 K 的时候，才放入队列。步骤：（1）第一步添加数据到队列1的头部（2）如果数据再队列1访问没有达到 K 次，则会继续达到链表底部直至淘汰，如果该数据在队中达到了 K 次，则会被加入到队列2中（3）接下来2级链表如上面的算法相同，链表中的数据如果再被访问则移到头部，链表满时，则底部数据淘汰。
## 缓存雪崩
大量的 key 设置了相同的过期时间，导致缓存在同一个时刻全部失效，造成瞬间请求 DB 的压力增大，引起雪崩。目前成熟的解决方案也有两种：
- 线程互斥：互斥锁或者分布式锁，只让其中一个线程构建缓存，其他线程等待该线程构建完再访问，这样做减轻了 DB 的压力，但缺点是降低了系统的 qps。
- 失效时间交错，可以在一定的范围时间内设置随机失效时间。
https://cloud.tencent.com/developer/article/1154769


### 跳跃表
skip lists 跳跃表，跳跃表是一种数据结构，可以用来代替平衡树。跳跃表使用概率平衡而不是严格强制的平衡，因此在跳跃表中插入和删除的算法要比平衡树的等价算法简单得多，而且要快得多。

一般查找的方法是分为两个：数查找、哈希查找
我们先来看一个有序链表：
![image](https://user-images.githubusercontent.com/12162133/65815491-d44b9780-e222-11e9-9a2e-8347e427f2f6.png)
在这样一个链表中，查找一个元素，需要遍历整个链表，时间复杂度为 O(n)，同样，插入的时候也需要遍历，从而确定位置。

假如我们在相邻的两个节点之间，增加一个指针，如下：
![image](https://user-images.githubusercontent.com/12162133/65815625-820b7600-e224-11e9-9d26-d50c55924aa4.png)
这样新增的一个指针造成了一个新的链表，但它包含的节点只是原来的一半，如果数据量很大的化，再回到原来的链表进行查找。
![image](https://user-images.githubusercontent.com/12162133/65815707-a7e54a80-e225-11e9-845f-fbe6c6bae0c5.png)
在一个新的三层链表中，如果查找23，可以很快的进行查找。

skiplist 正是受到这种多层链表的想法的启发而设计出来的，按照上面生成链表的方式，上面的每一层链表节点的个数是下面一层链表节点个数的一半，这样的查找类似于一个二分查找，使得查找的时间复杂度可以降低为 O(logN)，但是这种方法在插入数据的时候有很大的问题，新插入一个节点，会打破上下相邻层链表节点的个数严格2:1的关系。如果需要维持这种关系，就必须把后面所有的节点都要进行调整，这样时间复杂度会蜕变为O(n)。

skiplist 为了避免这个问题，它不要求上下相邻两层链表之间的节点个数有严格的关系，而是为每个节点随机选择一个层数。插入的过程，每个节点都是随机的，而且新插入每个节点不会影响其他节点的层数，这就降低了插入操作的复杂度。这让它插入性能上明显优于平衡树的方案。

skiplist 的算法性能如下：




























































































































































































































