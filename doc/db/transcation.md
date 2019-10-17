# InnoDB各版本对比
# InnoDB体系架构
![image](https://user-images.githubusercontent.com/12162133/41810105-52507038-772b-11e8-9c9a-8fcba1fc759d.png)
InnoDB存储引擎是多线程的模型，因此，后台有多个不同的后台线程，负责处理不同的任务。
1. Master Thread
Master Thread是一个非常核心的后台线程，主要负责将缓冲池中的数据异步刷新到磁盘上，保证数据的一致性，包括脏页的刷新、合并插入缓冲（INSERT BUFFER）、UNDO页的回收等。
2. IO Thread
在InnoDB存储引擎中大量使用AIO（Async IO）来处理写IO请求，这样可以极大提高数据库的性能。而IO Thread的工作主要是负责这些IO请求的回调（call back）处理。InnoDB工有4个IO Thread，分别是write、read、insert buffer和log IO thread。
```
mysql>SHOW VARIABLES LIKE 'innodb_version';
mysql>SHOW VARIABLES LIKE 'innodb_%_io_threads';
mysql>SHOW ENGINE INNODB STATUS;
```
3. Purge Thread
事务被提交后，其所使用的undolog可能不再使用，因此需要Purge Thread来回收已经使用并分配的undo页。用户可以在MySQL数据库的配置文件中添加以下命令：
```
[mysqld]
innodb_purge_threads=1
```
4. Page Cleaner Thread
Page Cleaner Thread 是在InnoDB1.2.x版本中引入的，其作用是将之前版本中的脏页的刷新操作都放入到单独的线程中完成。而其目的是为了减轻原Master Thread的工作。
# 内存
1. 缓冲池
InnoDB存储引擎是基于磁盘存储的，并将其中的记录按照页的方式进行管理，在数据库系统中，由于CPU速度和磁盘速度之间的鸿沟，基于磁盘的数据库系统通常使用缓冲池技术提高数据库的整体性能。
缓冲池简单来说是**一块内存区域**，在数据库读取页的操作中，首先将从磁盘读到的页存放在缓冲池中，这个过程将页"FIX"在缓冲池中，下一次读取相同的页时，可以直接读取缓冲池中的页。对于数据库中页的修改操作，则首先修改在缓冲池中的页，然后以一定的频率刷新到磁盘中，页从缓冲池刷新回磁盘的操作并不是在每次页发生更新时触发，而是通过一种称为**Checkpoint**的机制刷新回磁盘。对于InnoDB存储引擎来说，其缓冲池配置通过参数innodb_buffer_pool_size来设置。
从InnoDB 1.0.x版本开始，允许有多个缓冲池实例，每个页根据**哈希值**平均分配到不同的缓冲池实例中，可以通过参数innodb_buffer_pool_instances来进行配置。然后可以SHOW ENGINE INNODB STATUS来查看状态。
2. LRU list、Free List、Flush List
缓冲池是一块很大的内存区域，其中存放各种类型的页。数据库中的缓冲池是通过LRU（Latest Recent Used，最近最少使用）算法来进行管理的，即最频繁使用的页在LRU列表的前端，而最少使用的页在LRU列表的尾端。当缓冲池不能存放新读到的页面时，首先释放LRU列表中的尾端的页。
在InnoDB存储引擎中，缓冲池中的页的大小默认16KB，同样使用LRU算法，稍有不同的是对传统的LRU算法做了一些优化，LRU列表中加入midpoint位置，新读取到的页，并不是直接放入到LRU列表的首部，而是放入到LRU列表midpoint位置。这个算法在InnoDB存储引擎中称为**midpoint insertion strategy**，在默认配置下该位置在LRU列表长度的5/8处。
那为什么采用此种LRU算法呢？这是因为这些新查询的页通常不是活跃的热点数据，如果页被放入到LRU列表的首部，那么非常可能将所需要的热点数据从LRU列表中移除。

# 锁
锁是数据库区别于文件系统的一个关键特性。锁机制用于管理对共享资源的并发访问，提供数据的完整性和一致性。
对于 MyISAM 引擎来说，其锁是表锁，并发情况下读没问题，并发情况下插入性能较差。
## InnoDB 锁
- 共享锁（S Lock）：允许事务读一行数据
- 排他锁（X Lock）：允许事务删除或更新一行数据

排它锁和共享锁的兼容性：

||X|S|
|----|-----|------|
|X|冲突|冲突|
|S|冲突|兼容|

InnoDB 存储引擎支持多粒度锁定，这种锁定允许在行级上的锁和表级上的锁同时存在，为了支持在不同粒度上进行加锁操作，InnoDB 存储引擎支持一种额外的方式，我们称为**意向锁**。意向锁是表级别的锁，其设计目的主要是在一个事务中揭示下一行将被请求锁的类型。InnoDB 存储引擎支持两种意向锁：
- 意向共享锁（IS Lock）：事务想要获得一个表中某几行的共享锁
- 意向排他锁（IX Lock）：事务想要获得一个表中某几行的排他锁
因为 InnoDB 支持的是行级别的锁，所以意向锁其实不会阻塞除全表扫描以外的任何请求。在 INFORMATION_SHCEMA 架构下添加 INNODB_TRX、INNODB_LOCKS、INNODB_LOCK_WAITS，通过这三张表，可以简单地监控当前的事务并分析可能存在锁的问题。
```
select * from INFORMATION_SCHEMA.INNODB_TRX;
select * from INFORMATION_SCHEMA.INNODB_LOCKS;
select * from INFORMATION_SCHEMA.INNODB_LOCK_WAITS;
```
当事务量非常大，锁和等待也时常发生，这个时候不容易判断，但是通过 INNODB_LOCK_WAITS 可以很直观地反映出当前的等待。
```
`requesting_trx_id` varchar(18) NOT NULL DEFAULT '',    申请锁资源的事务 ID
`requested_lock_id` varchar(81) NOT NULL DEFAULT '',   申请的锁 ID
`blocking_trx_id` varchar(18) NOT NULL DEFAULT '',        阻塞的事务 ID
`blocking_lock_id` varchar(81) NOT NULL DEFAULT ''      阻塞的事务 ID
```
### 一致性非锁定读操作
一致性非锁定行读操作（consistent nonlocking read），是指 InnoDB 存储引擎通过多版本控制（multi versioning）的方式读取当前执行时间数据库中行的数据，如果读取的行正在执行 DELETE、UPDATE 操作，这时读取操作不会因此而会等待行上锁的释放，相反，InnoDB 存储引擎会去读取行的一个快照数据。
之所以称为非锁定读，因为不需要等待访问的行上 X 锁的释放。快照数据是指该行之前的版本数据，**该实现是通过 Undo 段来实现**，而 Undo 用来在事务中回滚数据，因此快照数据本身是没有额外的开销，此外，读取快照数据是不需要上锁的，因为没有必要对历史数据进行修改。
可以看到，非锁定读的机制大大提高了数据读取的并发性，这是默认的读取方式。但是在不同的事务隔离级别下，读取的方式不同，并不是每个事务隔离级别下读取都是一致性的。
快照数据其实就是当前行数据之前的历史版本，可能有多个版本，我们称这种技术为行多版本技术，由此，带来的并发控制，称之为多版本并发控制（Multi Version Concurrency Control，MVCC）。
在 RC 和 RR 级别下，InnoDB 存储引擎使用非锁定的一致性读，然而对于快照数据的定义却不相同，在 RC 隔离级别下，对于快照数据，非锁定的一致性读总是读取被锁定行的最新一份快照数据。在 RR 事务隔离级别下，对于快照数据，非锁定的一致性读总是读取事务开始时行数据版本。下面以 RR 事务隔离级别为例：
```
# Session A
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select * from user where id = 1;
+----+------+-----+---------------------+---------------------+
| id | name | sex | created_at          | updated_at          |
+----+------+-----+---------------------+---------------------+
|  1 | www  |   0 | 2019-03-26 00:39:05 | 2019-03-26 00:39:05 |
+----+------+-----+---------------------+---------------------+
1 row in set (0.01 sec)
```
会话 A 中的事务已经开始，读取 id = 1 的数据，但是没有结束事务。这时我们再开启另外一个事务：
```
# Session B
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> update user set id = 3 where id = 1;
Query OK, 1 row affected (0.03 sec)
Rows matched: 1  Changed: 1  Warnings: 0
```
会话 B 中将 id=1 的行记录修改为 id=3，但是事务同样没有提交，这样 id=1 的行其实加了一个 X 锁。这时如果再在会话 A 中读取 id=1 的数据，根据 InnoDB 引擎的特性，在 RC 和 RR 事务隔离级别下，会使用非锁定的一致性读。我们回到会话 A，接着上次未提交的事务，再执行查询操作，这个时候不管是 RC 或者 RR 事务隔离级别下，查询的数据都如下：
```
# Session A
mysql> select * from user where id = 1;
+----+------+-----+---------------------+---------------------+
| id | name | sex | created_at          | updated_at          |
+----+------+-----+---------------------+---------------------+
|  1 | www  |   0 | 2019-03-26 00:39:05 | 2019-03-26 00:39:05 |
+----+------+-----+---------------------+---------------------+
1 row in set (0.00 sec)
```
因为当前 id=1 的数据被修改了一次，因此只有一个版本的数据，接着，我们提交会话 B：
```
# Session B
mysql> commit;
Query OK, 0 rows affected (0.01 sec)
```
会话 B 提交后，这时在会话 A 中运行查询，在 RC 和 RR 事务隔离级别下，查询的数据就不一样了，对于 RC 事务隔离级别下，总是读取最新的数据，在 RR 事务隔离级别下读取的是事务开始时行的数据，RR 如下：
```
# Session A
mysql> select * from user where id = 1;
+----+------+-----+---------------------+---------------------+
| id | name | sex | created_at          | updated_at          |
+----+------+-----+---------------------+---------------------+
|  1 | www  |   0 | 2019-03-26 00:39:05 | 2019-03-26 00:39:05 |
+----+------+-----+---------------------+---------------------+
1 row in set (0.00 sec)
```
对于 RC 事务隔离级别下，其实违反了事务 ACID 中 I 的隔离性。
### SELECT …… FOR UPDATE & SELECT …… LOCK IN SHARE MODE
默认情况下，InnoDB 引擎的 SELECT 操作使用一致性非锁定读，但是在某些情况下，我们需要对读取操作进行加锁。InnoDB 存储引擎对于 SELECT 语句支持两种加锁操作：
- SELECT …… FOR UPDATE：对读取的行记录加 X 锁，其他事务想在这些行上加任何锁都会阻塞
- SELECT …… LOCK IN SHARE MODE：对读取的行记录加 S 锁，其他事务可以向被锁定的行记录加 S 锁，但是对于加 X 锁，则会被阻塞。

对于一致性非锁定读，即使被读取的行记录已经使用 SELECT …… FOR UPDATE，也是可以进行读取的。在使用上述两个语句时。

## 锁的算法





# 性能优化
- 在使用类型字段时，一般原则是最小满足原则
- 存储引擎
```
mysql> show storage engines;
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| Engine             | Support | Comment                                                        | Transactions | XA   | Savepoints |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
| PERFORMANCE_SCHEMA | YES     | Performance Schema                                             | NO           | NO   | NO         |
| MRG_MYISAM         | YES     | Collection of identical MyISAM tables                          | NO           | NO   | NO         |
| MyISAM             | YES     | MyISAM storage engine                                          | NO           | NO   | NO         |
| BLACKHOLE          | YES     | /dev/null storage engine (anything you write to it disappears) | NO           | NO   | NO         |
| InnoDB             | DEFAULT | Supports transactions, row-level locking, and foreign keys     | YES          | YES  | YES        |
| CSV                | YES     | CSV storage engine                                             | NO           | NO   | NO         |
| ARCHIVE            | YES     | Archive storage engine                                         | NO           | NO   | NO         |
| MEMORY             | YES     | Hash based, stored in memory, useful for temporary tables      | NO           | NO   | NO         |
| FEDERATED          | NO      | Federated MySQL storage engine                                 | NULL         | NULL | NULL       |
+--------------------+---------+----------------------------------------------------------------+--------------+------+------------+
9 rows in set (0.01 sec)
```
- 执行计划
```
id：表示查询中select操作表的顺序，按顺序从大到小依次执行，比如上面的sql中，in的子查询会先执行
select_type：表示选择类型，常见可选择有：SIMPLE(简单的), PRIMARY(最外层) ，SUBQUERY(子查询)
type：表示访问类型，常见有：ALL(全表扫描)，index(索引扫描)，range(范围扫描)，ref(非唯一索引扫描)
table，数据所在的表
possible_keys，可能使用的索引
key：实际使用的索引
key_len：索引列所用的字节数
ref：连接匹配条件,如果走主键索引的话，该值为: const, 全表扫描的话，为null值
row，扫描的行数，行数越少，查询效率就会越高，我们的优化大部分都是为了降低该值
extra：这个属性非常重要，该属性中包括执行SQL时的真实情况信息，常用的有"Using temporary"，使用临时表；"using filesort": 使用文件排序
```
- 统计某个列的数据离散程度
```
select count(distinct(id))/count(1) from t;
```
- 10 个常见的 sql 命令
```
show databases
use database_name
show tables
show full columns from table_name
select version()
select current_user()
show table status like "table_name"
show processlist
show index from table_name    # 查看表的索引，有个很重要的概念，叫做：Cardinality，这个值越大，表示索引效果越好
```

# 数据库事务
## 事务的特性
* 原子性：数据库事务是不可分割的单位，即要么都做，要么都不做
* 一致性：一致性指事务将数据库从一种状态转变为下一种一致的状态
* 隔离性：即该事务提交前对其他事务不可见，通常使用锁来实现
* 持久性：事务一旦提交，其结果就是永久性的。即使发生宕机等故障，数据库也能将数据恢复

事务的一致性基本可以理解为事务对数据完整性约束的遵循。这些约束包括主键约束、外键约束或一些用户自定义约束。事务执行前后都是合法的数据状态，不会违背任何数据的完整性。
## 事务的隔离级别
SQL标准定义了四个隔离级别，分为别：
1.READ UNCOMMITTED
2.READ COMMITTED
3.REPEATABLE READ
4.SERIALIAZBLE
InnoDB引擎支持的隔离级别是REPEATALBE READ，SQL和SQL2标准的默认事务隔离级别是SERILIAZBLE，但与标准不同的是InnoDB存储引擎在REPETABLE READ事务隔离级别下，使用Next-Key Lock算法，因此避免幻读的产生。所以说，InnoDB存储引擎在默认的REPETABLE READ的事务隔离级别已经能完全保证事务隔离性的要求了。在InnoDB引擎中，可以设置当前事务隔离级别：
SET [GLOBAL|SESSION] TRANSCATION ISOLATION LEVEL
{RU | RC |RR |S}
如果想在MySQL数据库启动时候就设置事务的隔离级别，那就需要修改MySQL的配置文件，在[mysqld]中添加如下行：
[mysqld]
transcation-isolation = READ-COMMITTED
## 数据库隔离级别解决了哪些问题
1.脏读
有AB两个事务，B事务能读取A事务未提交的结果，B事务读取的可能不是A事务修改的最终结果，所以存在脏读。
RU隔离级别下会出现这种问题，RC隔离级别解决了这种问题
2.重复读
有AB两个事务，A事务两次查询同一个记录，在两次查询间隙，B事务修改这一条记录并且已经提交，造成了A事务两次查询的记录不一致。
RC隔离级别及以下都会出现这种问题，RR隔离级别解决了这种问题，RR解决的是修改问题，即UPDATE操作。
3.幻读
有AB两个事务，A事务两次查询某一统计结果，在两次查询间隙，B事务插入了一条新的记录，造成了两次查询结果不一致。造成了幻读问题。Serializable隔离级别解决了这个问题，在该级别下，事务串行化的执行，可以避免脏读、不可重复读、幻读，但是这种事务隔离级别效率低下，比较消耗数据库性能。
**不可重复读的重点是修改，幻读的重点在于新增或者删除。**
## 锁的算法
InnoDB存储引擎存在3种行锁的算法，其分别是：
* Record Lock：**行锁**，单条索引记录上加锁，锁住的永远是索引，而非记录本身（如果InnoDB存储引擎表在建立的时候没有设置任何一个索引，那么这时InnoDB存储引擎会使用隐式的主键进行锁定）
* Gap Lock：**间隙锁**，在索引记录之间的间隙中加锁，锁定一个范围，但不包含记录本身
* Next-Key Lock：**后码锁**，Gap Lock+Record Lock的结合，锁定一个范围，并且包括记录本身。
Next-Key Lock是结合了Gap Lock和Record Lock的一种锁定算法，在Next-Key Lock算法下，InnoDB对于行的查询都是采用这种锁定算法。例如一个索引有10 11 13 20这四个值，那个该索引可能被锁定的区间为：
```
(-oo,10]
(10,11]
(11,13]
(13,20]
(20,+oo)
```
采用Next-Key Lock的锁定技术称为Next-Key Locking。其设计的目的是为了解决Phantom Problem。除了Next-Key Locking，还有Previous-Key Locking，那么上述索引锁定的区间为：
```
(-oo,10)
[10,11)
[11,13)
[13,20)
[20,+oo)
```
然而，当查询的索引含有唯一属性时，InnoDB存储引擎会对Next-Key Lock进行优化，降级为Record Lock，即仅锁住索引本身，而不是范围。正如前面所介绍的情况，Next-Key Lock降级为Record Lock仅在查询的列是唯一索引（**聚集索引和唯一索引**）的情况下。
## MySQL解决Phantom Problem
- 在快照读读情况下，mysql通过mvcc来避免幻读。
- 在当前读读情况下，mysql通过next-key来避免幻读

快照读和当前读的分类：
- 快照读：简单的 select 操作，属于快照读，不加锁。
select * from table where ?;
- 当前读：特殊的读操作，插入/更新/删除操作，属于当前读，需要加锁。
select * from table where ? lock in share mode;
select * from table where ? for update;
insert into table values (…);
update table set ? where ?;
delete from table where ?;

MySQL数据库默认的事务隔离级别是RR，存在幻读问题。快照读情况下，不需要用到锁，是通过 MVCC 解决幻读的：
```
# Transcation 1
mysql> begin;
Query OK, 0 rows affected (0.01 sec)

mysql> select count(*) from user where id >=4;
+----------+
| count(*) |
+----------+
|        2 |
+----------+
1 row in set (0.01 sec)
```
```
# Transcation 2
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into user(id,name) values(7,'jjj');
Query OK, 1 row affected (0.02 sec)

mysql> commit;
Query OK, 0 rows affected (0.01 sec)
```
```
# Transcation 1
mysql> select count(*) from user where id >=4;
+----------+
| count(*) |
+----------+
|        2 |
+----------+
1 row in set (0.00 sec)
```
如上按照顺序执行，在 Transcation 1 中属于快照读，避免了当前读的发生。

当前读的情况下，InnoDB 存储引擎采用了Next-Key Locking机制来避免Phantom Problem。Phantom Problem问题演示：
```
# Transcation 1
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> select count(*) from user where id >=4 for update;
+----------+
| count(*) |
+----------+
|        3 |
+----------+
1 row in set (0.01 sec)
```
```
# Transcation 2
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into user(id,name) values(8,'kkk');
ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction
```
InnoDB存储引擎采用了Next-Key Locking算法避免了Phantom Problem，对于上述的SQL语句SELECT * FROM t WHERE a>2 FOR UPDATE，其锁住了不单是5这个值，而是对（2，+oo）这个范围加了X锁。因此任何对于这个范围的插入都是不被允许的，从而避免了Phantom Problem。

## MVCC
即多版本控制并发协议，它使得大部分支持行锁的事务引擎，不再单纯的使用行锁来进行数据库的并发控制，取而代之的是把数据库的行锁与行的多个版本结合起来只需要很小的开销,就可以实现非锁定读，引入多版本控制协议后，只有写写之间会相互阻塞，其他三种操作可以并行，从而大大提高数据库的并发性能。其他数据库也有 MVCC 这一概念，只是各个数据库的实现机制并不相同。**MVCC 只有在 RC 和 RR 两个隔离级别下工作**，其他两个级别和 MVCC 不兼容。
InnoDB 的存储引擎，默认隔离级别是 RR、行级锁、实现了 MVCC、Consistent  nonlocking read。InnoDB 通过 undo log 来实现  MVCC 协议，undo log 也用来实现事务的回滚。

mysql 除了binlog、错误日志、查询日志外还有另外三种日志：redo log、bin log、undo log。
binlog 是用来实现数据恢复，数据库复制，常见的 mysql 主从架构，就是采用 slave 同步 master 的 binlog 来实现的，另外通过 binlog 也可以复制到其他数据源（例如：es）。
redo log 重做日志文件，当开始事务时，会记录一个事务编号（LSN），当事务执行时，会往InnoDB 存储引擎的日志缓冲里插入事务日志，当事务提交时，必须先将 InnoDB 存储引擎的日志缓冲写入磁盘，也就是写数据之前，先写日志，这种方式称为预写日志的方式，通过预写日志来保证事务的完整性。


undo log 撤销日志，对于数据库进行修改时，会产生一定了的 undo log，即使你执行的事务失败了，会 rollback，就可以利用这些 undo 信息将数据回滚到修改之前的样子，undo 会存放在数据库内部一个特殊段（segment），undo 段位于共享表空间内。通过 undo 日志只是逻辑地恢复到原来的样子，因为在多并发用户系统中，可能会有数十、数百、数千个并发事务，如一个事务在修改当前页的几条记录，因此不能将一个页回滚到事务开始的样子，因为这样会影响其他事务执行。因此当 InnoDB 引擎回滚时，实际上做的是与之前相反的操作。

主键所在的索引为聚簇索引，cluster index 叶子节点保存了对应的数据内容。undo log 分为 insert 和 update，insert undo 为 insert 操作产生的记录，update undo 为 delete 和 update 产生的记录。undo log 记录了数据之前的信息。可以看到 InnoDB 存储引擎在数据库每行数据后面增加三个字段。这三个字段分为 DB_TRX_ID、DB_ROLL_PTR、DB_ROW_ID。
对于当前行的最后提交事务为 trx_id_current，分为三种情况：
1.trx_id_current < up_limit_id, 这种情况比较好理解, 表示, 新事务在读取该行记录时, 该行记录的稳定事务ID是小于, 系统当前所有活跃的事务, 所以当前行稳定数据对新事务可见, 跳到步骤5.
2.trx_id_current >= trx_id_last, 这种情况也比较好理解, 表示, 该行记录的稳定事务id是在本次新事务创建之后才开启的, 但是却在本次新事务执行第二个select前就commit了，所以该行记录的当前值不可见, 跳到步骤4。
3.trx_id_current <= trx_id_current <= trx_id_last, 表示: 该行记录所在事务在本次新事务创建的时候处于活动状态，从up_limit_id到low_limit_id进行遍历，如果trx_id_current等于他们之中的某个事务id的话，那么不可见, 调到步骤4,否则表示可见。
4.从该行记录的 DB_ROLL_PTR 指针所指向的回滚段中取出最新的undo-log的版本号, 将它赋值该 trx_id_current，然后跳到步骤1重新开始判断。
5.将该可见行的值返回。

## 基础知识
```
mysql -hlocalhost -uroot -p
Enter password:
```
我们最好不要在一行里面输入密码，防止别人看到

一般的数据库和客户端运行在不同的机器上，它们之间采用的连接是TCP，mysql 服务器默认是 3306端口。
![image](https://user-images.githubusercontent.com/12162133/65834380-d68a2080-e30c-11e9-8e69-ba0bd1abcb78.png)
不过既然是缓存，那就有它缓存失效的时候。MySQL的缓存系统会监测涉及到的每张表，只要该表的结构或者数据被修改，如对该表使用了INSERT、 UPDATE、DELETE、TRUNCATE TABLE、ALTER TABLE、DROP TABLE或 DROP DATABASE语句，那使用该表的所有高速缓存查询都将变为无效并从高速缓存中删除！从MySQL 5.7.20开始，不推荐使用查询缓存，并在MySQL 8.0中删除。
```
SHOW ENGINES;
```
通过以上命令可以查看存储引擎。




























