# 简介
Zookeeper 的设计更专注于任务协作，并不提供任何锁的接口或通用存储数据接口。**Zookeeper 服务器集群管理着应用协作的关键数据，不适合作海量数据存储**，对于需要存储海量数据的可以选择数据库或分布式文件系统等。
本书中，**分布式系统的定义：分布式系统是同时跨越多个物理主机，独立运行的多个软件组件所组成的系统**。
分布式系统中进程的通信有两种：（1）直接通过网络进行消息交换；（2）读写某些共享存储，Zookeeper 采用的是共享存储模型来实现应用间的协作和同步原语。对于共享存储本身，又需要在进程和存储间进行网络通信，我们强调网络通信的重要性，因为它是分布式系统中并发设计的基础。
**问题：在实际情况中，我们很难判断一个进程是奔溃了还是某些因素导致了延迟。**
## 主从应用
![image](https://user-images.githubusercontent.com/12162133/58366172-01004300-7f01-11e9-8688-4b9895d7a708.png)
主从架构需要解决三类问题：
- 主节点崩溃：对于主节点的状态的可恢复性，我们不能依靠从已经崩溃的主节点来获取这些信息，而需要从其他地方获取，也就是通过 Zookeeper 获取。主节点奔溃可能会引起**脑裂**
- 从节点崩溃：从节点执行的任务执行失败，主节点需要重新指派任务
- 通信故障：通信故障可能会造成一个任务被执行多次，也就是仅一次和最多一次的语义。
  
通过上述主从架构，可以理解主从架构的需求：
- 主节点选举
- 崩溃检测
- 组成员关系管理
- 元数据管理

## 分布式协作的难点

拜占庭将军问题：

在分布式环境中，基于理想情况下，我们基于异步系统通信的假设来设计系统，即我们使用的主机可能发生时间偏移或通信故障，
我们来看看一个简单的情况，假设我们有一个分布式的配置信息发生了改变，一旦所有运行中的进程对配置信息的值达成一致，我们的应用进程就可以启动。

CAP:一致性（Consistency）、可用性（Availability）、分区容错性（Partition-tolerance）；分布式系统中没有系统能同时满足这三种属性。因此，Zookeeper 的设计尽可能满足一致性和可用性。

## Zookeeper 的成功
Zookeeper 在分布式计算领域进行了大量的工作，Paxos 算法和虚拟同步技术（virtual synchrony）给 Zookeeper 带来了很大影响。

# 了解 Zookeeper
## Zookeeper 基础
Zookeeper 操作和维护了一个小型的数据节点，这些节点被称为 znode，采用类似于文件系统的层级树状结构进行管理。
![image](https://user-images.githubusercontent.com/12162133/58369563-d0cf9900-7f2e-11e9-80f6-4f749593aaa6.png)
znode 节点可能含有数据，可能也没有数据。Zookeeper 的 API 包含了以下方法：
- create/path data
创建一个名为 /path 的 znode，并包含数据 data
- delete/path
删除名为 /path 的 znode
- exists/path
检查是否存在名为 /path 的 znode
- setData/path data
设置名为 /path 的 znode 的数据为 data
- getData/path
获取 /path 的 znode 的数据信息
- getChildren/path
返回所有 /path 节点的所有子节点列表

znode 节点的不同类型，持久节点和临时节点；持久节点只能通过调用 delete 来进行删除，临时节点与之相反，当创建该节点的客户端崩溃或关闭了与 Zookeeper 的连接时，这个节点就会被删除。
持久节点是一种非常有用的节点，可以通过持久类型节点为应用保存一些数据，即使 znode 的创建者不再属于应用系统时，数据也可以保存下来而不丢失。
临时 znode 传达了应用某些方面的信息，仅当创建者的会话有效时这些信息必须保存。一个临时 znode 在其创建者的会话期时被删除，所以我们现在不允许临时节点拥有子节点。
znode 一共有 4 种类型：持久的、临时的、持久有序的、临时有序的。

监视与通知
Zookeeper 通常以远程服务的方式被访问，如果每次访问 znode，客户端都需要获取节点中的内容，这样的代价就非常大。因为这样会导致更高的延迟，而且 Zookeeper 需要做更大的操作。
![image](https://user-images.githubusercontent.com/12162133/58370339-e09fab00-7f37-11e9-9c4a-0e5491683c91.png)
这是一个常见的轮询问题，为了替换客户端的轮询，我们选择了基于通知（notification）的机制，客户端向 Zookeeper 注册需要接收通知的 znode，通过对 znode 设置监视节点 watch 来接收通知。监视点是一个单次触发的操作，意即监视点会触发一个通知。为了接收多个通知，客户端必须在每次通知后设置一个新的监视点。
![image](https://user-images.githubusercontent.com/12162133/60765820-e78f1100-a0d2-11e9-94f3-2bdb27efda45.png)

（1）客户端 2 获取节点 /tasks 下的信息，并通过 watch 对该节点监听
（2）客户端 1 创建节点 /tasks 下的持久有序节点 /tasks/task-1，ZK 会给监听该节点的客户端发通知，ZK 发通知给客户端 2
（3）紧接着，客户端1 又创建节点 /tasks 下持久有序节点 /tasks/task-2，ZK 发现没有客户端设置监控，不发通知
（4）客户端 2 收到刚刚 （2）的通知，更新信息，获取 /tasks 节点下的信息，并设置当前节点的更新

这样，客户端在接收一个节点变更通知并设置新的监视节点时，及时在这过程中发生了更新，当前客户端也不会丢失信息。

ZooKeeper 中可以定义不同类型的通知，这依赖于设置监视点对应的通知类型。客户端可以设置多种监视节点，监控节点数据变化、监控子节点变化、监控节点创建与删除。

**版本号**
 每一个 zonde 都有一个版本号，随着每次数据变化而增加，两个 API 的操作 setData、delete 这两个调用以版本号作为传参，只有当转入参数的版本号与服务器上的版本号一致时才会调用成功。版本机制可以避免多个客户端对同一个节点进行操作时显得尤为重要。
![image](https://user-images.githubusercontent.com/12162133/60766273-01802200-a0da-11e9-92d0-628c997e3e13.png)

## ZooKeeper 架构
ZooKeeper 服务器运行两种模式：独立模式、仲裁模式。
![image](https://user-images.githubusercontent.com/12162133/60768405-431ec600-a0f6-11e9-942c-7c66dbdd16ca.png)

### 会话
客户端初始连接到集合中的某一个服务器上或一个独立的服务器，客户端通过 TCP 协议与服务器进行连接并通信。
会话提供了顺序保障，意味着同一个会话中的请求会以 FIFO 的顺序执行。通常，一个客户端只会打开一个会话，会话的状态如下：
![image](https://user-images.githubusercontent.com/12162133/60809526-711b0d80-a1bd-11e9-909e-871fb6e860e9.png)

![image](https://user-images.githubusercontent.com/12162133/60809686-d53dd180-a1bd-11e9-801e-0c3430ac9a2e.png)

![image](https://user-images.githubusercontent.com/12162133/60809829-34034b00-a1be-11e9-9afa-e27603843149.png)

### 通过 Zookeeper 实现一个分布式锁
利用 Zookeeper 的一些特性可以完成：
- 同一时刻多个客户端创建同一个节点，只有一个会争抢成功，利用这个特性可以做分布式锁
- 临时节点的生命周期与会话一致，会话关闭临时节点删除，这个特性经常会做心跳、动态监控、负载等动作
- 顺序节点会保证节点唯一，这个可以作为分布式锁全局环境下的自增 id
lock：
- 在 /lock_name 节点下创建临时有序节点
- 获取当前线程创建的节点及 /lock_name 下的所有子节点，确保当前节点序号是最小的，如果是最小的，则加锁成功。否则则监听序号较小的前一个节点。
unlock：
- 删除当前线程创建的临时节点

问题：
- 锁无法释放？
客户端会创建一个临时节点，一旦客户端获取锁之后挂掉（session 断开），那么这个临时节点就会删除，其他节点可以再次获取锁
- 非阻塞锁？
客户端通过 Zookeeper 创建临时有序节点，并且在节点上绑定监听器你，一旦节点有变化，就会通知客户端，客户端可以获取所有节点，比较当前节点是不是最小的节点，如果是的，那么就自动获取锁
- 不可重入
客户端在创建节点的时候，把当前客户端的主机信息和线程信息直接写入到节点中，下次想要获取锁的时候和当前最小的节点中的数据比对一下就可以了
- 单店问题
Zookeeper 可以通过集群部署的

### 安装与配置
```
version: '3.1'

services:
  zoo1:
    image: zookeeper
    restart: always
    hostname: zoo1
    ports:
      - 2181:2181
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=0.0.0.0:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo2:
    image: zookeeper
    restart: always
    hostname: zoo2
    ports:
      - 2182:2181
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=0.0.0.0:2888:3888;2181 server.3=zoo3:2888:3888;2181

  zoo3:
    image: zookeeper
    restart: always
    hostname: zoo3
    ports:
      - 2183:2181
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zoo1:2888:3888;2181 server.2=zoo2:2888:3888;2181 server.3=0.0.0.0:2888:3888;2181
```
```
docker-compose up
zkCli.sh -server localhost:2181,localhost:2182,localhost:2183
```
操作详情如下：
```
[zk: localhost:2181,localhost:2182,localhost:2183(CONNECTED) 0] create -e /master "master1.example.com:2223"
Created /master
[zk: localhost:2181,localhost:2182,localhost:2183(CONNECTED) 1] ls /
[zookeeper, master]
[zk: localhost:2181,localhost:2182,localhost:2183(CONNECTED) 2] get /master
master1.example.com:2223
cZxid = 0xb00000002
ctime = Wed Jul 10 22:41:13 CST 2019
mZxid = 0xb00000002
mtime = Wed Jul 10 22:41:13 CST 2019
pZxid = 0xb00000002
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x10027be03140000
dataLength = 24
numChildren = 0
```
创建 znode 节点，以便获得管理权，-e 标志表示为临时节点。
![image](https://user-images.githubusercontent.com/12162133/60982091-0fe46d00-a36a-11e9-9f04-349a142cb46f.png)

### 会话的状态和声明周期
会话的声明周期是指会话从创建到结束的时期，无论会话正常关闭还是因为超时而导致过期。一个会话的状态：
![image](https://user-images.githubusercontent.com/12162133/60982608-0c9db100-a36b-11e9-8308-9bb693505113.png)

创建一个会话，需要设置会话的超时时间。客户端断开连接然后进行重连时候：
![image](https://user-images.githubusercontent.com/12162133/61183786-0f084f80-a678-11e9-85e8-aa2883b6b111.png)

为了让服务器之间可以通信，服务器之间需要一些联系信息，每个 server.n 项都指定了编号为 n 的 zookeeper 服务器使用的地址和端口号，分别用于仲裁通信和群首选举。

## 使用 ZooKeeper 进行开发
ZooKeeper 提供了 Java 和 C 语言的 KPI 插件，这两个插件拥有相同的基础结构和特性，ZooKeeper 的 API  围绕 ZooKeeper 的句柄 handle 而构建，每个句柄代表与 ZooKeeper 之间的一个会话。ZooKeeper 的客户端会持续保持这个活跃连接，以保证与 ZooKeeper 服务器之间的会话存活。
![image](https://user-images.githubusercontent.com/12162133/61184233-e5eabd80-a67d-11e9-8b1d-7570b6c9827f.png)

- connectString：服务器连接地址和端口
- sessionTimeout：毫秒为单位
- watcher：用于接收会话事件的一个对象，这个对象需要我们自己创建，客户端使用 watch 接口来监控与 ZooKeeper 之间的会话健康情况。与 ZooKeeper 服务器之间建立和失去连接就会产生事件，它们同样还能用于监控 ZooKeeper 数据的变化。

ZooKeeper 有两种管理接口，JMX 和 四字母组成。

配置文件：
![image](https://user-images.githubusercontent.com/12162133/64033234-9e14ec80-cb7e-11e9-970b-168fe7adef3b.png)
clientPort: zk 服务器监听的端口，用于客户端的连接
dataDir:zk 用于保存内存数据库的快照的目录，除非设置了dataLogDir，否则这个目录也用来保存其他更新数据库的事务日志，在生产环境使用 zk 集群，强烈建议设置 dataLogDir，让 dataDir 只存放快照，因为写快照开销很低，这样 dataDir 就可以和其他日志目录挂载在一起了。
dataDirLog: zk 事务日志路径
tickTime:zk 时间单元
initLimit: 从 leader 同步数据的最大时间
syncLimit：与 leader 交互时的最大时间

API
watcher:用于接收会话事件的一个对象，这个对象需要我们自己创建，因为 Watcher 定义为接口，所以我们需要自己实现一个类，用于监控 zk 数据的变化，如果 zk 的会话过期，也会通过 Watcher 接口传递事件通知客户端的应用。

![image](https://user-images.githubusercontent.com/12162133/64076119-17cde700-ccf3-11e9-9582-71243b4ecc7f.png)

单次触发器，如果会话过期，等待中的监视点也会删除，不过监视点可以跨越不同服务端的连接而保持，当一个 zk 客户端与一个服务器连接断开后到另外一个服务器，客户端会发送未触发的监视点列表。

![image](https://user-images.githubusercontent.com/12162133/64076756-58c9f980-ccfb-11e9-9fb0-9a9e17cab2bf.png)

单次触发是否会丢失事件？
答案是肯定的，一个应用在接收通知后，注册另外一个监视点时，可能会丢失事件，当然丢失事件通常并不是问题，可以通过 getChildren 调用获取任务列表时会返回所有任务，同时调用 getChildren 时可以设置新的监视点，从而保证从节点不会丢失任务。

如何设置监视点？
zk 中所有的 API 读操作：getData、getChildren、exists 均可以选择读取的 znode 节点上设置监视点，使用监视点机制，我们需要实现 Watcher 接口类，实现其中的 process 方法。
![image](https://user-images.githubusercontent.com/12162133/64077243-e1976400-cd00-11e9-9ef4-1bfb8866ad0a.png)
监视点有两种类型：数据监视点和子节点监视点。对于监视点有一个问题，一旦设置监视点，就无法删除该监视点，只有两个方法，触发该监视点或者关闭会话。

管理权变化？
用于客户端在 zk 服务器创建主节点 /master（称为主节点竞选），如果 znode 节点已存在，应用客户端确认自己不是主节点并返回，如果主节点奔溃，备份主节点并不知道，因此我们需要在 /master 上设置监视点。
主节点等待从节点列表变化，我们在 zk 服务器上中的  /workers 中添加子节点来注册新的节点列表。分配任务也差不多如下：
![image](https://user-images.githubusercontent.com/12162133/64085511-d0cd0980-cd65-11e9-9d51-ae425a13970f.png)







## 脑裂
分布式系统最基本的问题是网络隔离（network-partitions，即 split-brain），当网络发生异常的情况，导致分布式系统中部分节点之间的网络延时不断增大，最终会导致只有部分节点能够正常通信，而另一些节点不能，我们将这个现象称之为网络分区。当网络分区出现时，分布式系统会出现局部小集群，这对分布式集群是一个非常严重的考验。
对于无状态服务的 HA，不在乎脑裂问题；但是对于有状态服务的 HA，脑裂问题需要解决。

zookeeper 脑裂，zookeeper 是 leader 和 follower 模式，是一个有状态服务的 HA，当集群网络信号不好，导致心跳检测失败。follower 节点认为 leader 节点由于某种原因挂掉了，可其实 leader 节点并未真正挂掉，假死后，zookeeper 集群通知所有 follower 节点进行选举，最终某个 follower 节点升级为 leader 节点，这样在集群中就会出现多个 leader 节点。那么 zookeeper 是如何解决脑裂问题的。

zookeeper 是通过 Quorums（法定人数），即只有集群中超过半数节点投票才能选举出 leader，这样可以确保 leader 的唯一性。那么选举过程中为什么一定要一个过半机制，因为这样不需要等待所有节点都投了一个节点就可以选举出一个 leader 节点了，那过半机制为什么是大于一半呢？当机房中间的网络断掉后，如果两个机房的节点分别是 50 个节点，如果是大于等于的化，这样就是每个机房都会选举出一个 leader 节点，整个集群就有两个 leader 节点啦（但是这个问题可以通过权重来解决），另外一种去情况是 leader 假死，当选举了一个新的 leader 节点后，zookeeper 维护了一个叫 epoch 的变量，每当新 leader 产生时，epoch 都会增加，followers 如果确认了新的 leader 存在，同时也会知道 epoch 的值，它们会拒绝 epoch 小于现在值的 leader 节点，所以这就避免了即使旧的 leader 节点复活了，也不会在 zookeeper 集群中同时包含两个 leader 节点。

## zookeeper 介绍
![image](https://user-images.githubusercontent.com/12162133/63732621-a5e73f00-c8a7-11e9-924b-6fff00f7a14b.png)
![image](https://user-images.githubusercontent.com/12162133/63732656-d6c77400-c8a7-11e9-80e7-009926bc1d8e.png)

- 强一致性：要求更新过的数据能被后续的访问都能看到
- 弱一致性：如果能容忍后续的部分或者全部访问不到
- 最终一致性：如果经过一段时间后要求能访问到更新后的数据

## Paxos 算法
Paxos 算法是 Lamport 1990 年提出的一种基于消息传递的一致性算法。后面 google 中的 chubby 锁服务使用 Paxos 作为一致性算法，从此 Paxos 一路狂飙。Paxos 算法解决的是分布式系统如何就某个值达成一致，分布式一致性协议。**Google 的 Chubby 的作者 Mike Burrows 说过世界上只有一种一致性算法，那就是 Paxos 算法，其他的算法都是残次品。**

### Paxos 解决什么问题？
Paxos 用来确定一个不可变变量的值，取值可以是任意一个二进制数据，一旦确定将不再更改，并且可以被获取到（不可变性、可读取性）

### Paxos 如何在分布式存储系统中应用？
可靠存储，多副本存储，延迟或系统故障可能导致多个副本的值是不一样的，这样很难保持一致；多个副本的更新操作序列【op1、op2、op3、op4、op5】是相同的、不变的。用 Paxos 依次来确定不可变量 opi 的取值。每确定完 opi 之后，让各个数据副本执行 opi，依次类推。为了保证各个节点执行相同的命令序列，是分布式中最重要的问题。
![image](https://user-images.githubusercontent.com/12162133/62476709-226f8c00-b7da-11e9-9588-c06c7d0d9c0d.png)
![image](https://user-images.githubusercontent.com/12162133/62476769-36b38900-b7da-11e9-80ea-7354520c75c4.png)


### Paxos 算法的核心思想是什么？

Paxos 算法有两个原则：
1. 安全原则
（1）只能有一个值被批准，不能出现第二个值把第一个覆盖的情况
（2）每个节点只能学习到已经被批准的值，不能学习没有被批准的值
2. 存活原则
（1）最终会批准某个被提议的值
（2）一个值被批准了，其他服务器最终会学习这个值

Paxos 有两个组件：
1. Proposer
提议发起者
2. Acceptor
提议批准者

问题：
1. 一个 Acceptor
如果只有一个 Acceptor，如果此 Acceptor  crash 了，这一条违反了存活原则
2. 多个 Acceptor
为了解决上述问题，就必须用到一种多数选择的方法，使用一个 Acceptor 集合，然后只有其中的多数批准了一个值，这个值才可以确实被批准了。

延伸出两阶段提交原则：
1. 提议排序
就需要一旦 Acceptor 批准了某个值，其他有冲突的值都应该被拒绝，为了做到这一点，就需要对 Proposer 进行排序，将排序在前的赋予最高优先级，Acceptor 批准优先级最高的值，拒绝排序最后的值，为了将提议进行排序，可以为每个提议赋予一个唯一的 ID，规定这个 ID 越大，优先级越高
1.1 提议 ID 生成算法

## CAP
![image](https://user-images.githubusercontent.com/12162133/62634825-1d8d1280-b969-11e9-901e-80170ee8e38c.png)
当你开发或设计一个分布式系统的时候，CAP Theorem 是你无论如何都绕不过去的， CAP Theorem 对于一个分布式系统，不能同时满足以下三点，最多只能满足其中的两项：
- 一致性
- 可用性
- 分区容错性
需要注意的是，CAP Theorem 中的 CA 和数据库事务中的 ACID 中的 CA 并不是一回事。
## Consistency 一致性
> all nodes see the same data at the same time
所有节点在同一时间的数据完全一致，所以一致性就是数据一致性
三种一致性策略：
- 对于关系型数据库，要求更新过的数据能被后续的访问都能看到，这是强一致性。
- 如果能容忍后续的部分或者全部访问不到，则是弱一致性。
- 如果经过一段时间后要求能访问到更新后的数据，则是最终一致性。
- CAP中说，不可能同时满足的这个一致性指的是强一致性。

## Availability 可用性
可用性是指 "Reads and writes always succeed"，即服务一直可用，而且是正常响应时间，我们一般在衡量一个系统可用性的时候，都是通过停机时间来计算的

可用性分类 | 可用水平（%） | 年可容忍停机时间
-- | -- | --
容错可用性 | 99.9999 | <1 min
极高可用性 | 99.999 | <5 min
具有故障自动恢复能力的可用性 | 99.99 | <53 min
高可用性 | 99.9 | <8.8 h
商品可用性 | 99 | <87.6 h

## Partition Tolerance 分区容错性
分区容错性指 "the system continues to operate despite arbitrary message loss or failure of part of the system" ，即分布式系统在遇到某节点或网络分区故障的时候，仍然能够对外提供满足一致性和可用性的服务。

## CAP 的证明
假设一个分布式系统中有三个节点，分别是 N1、N2、N3，它们之间网络可以连通。在满足一致性的时候，三个节点的 V 的值是一样的；在满足可用性的时候，不管请求哪个节点，都会立即得到响应；在满足分区容错性的情况下，三个节点有任何一个节点宕机，或者网络不通的时候，都不会影响各自节点的正常运作。
![image](https://user-images.githubusercontent.com/12162133/63860462-c0bad000-c9db-11e9-936c-36694800060c.png)


### 强一致性算法
一致性模型：
- 弱一致性：最终一致性，DNS Gossip
- 强一致性：同步、Paxos、Raft、ZAB

数据不能存在单点上，分布式系统对分区容错性的一般解决方案是 state machine replication，Paxos 其实是共识算范，系统的最终一致性，不仅需要达成共识，还会取决于 client 的共识。
1. 主从同步复制
Master 接受写请求
Master 复制日志值 Slave
Master 等待，直到所有从库返回
问题：一个节点失败，Master 阻塞，导致整个集群不可用，保证了一致性，可用性却大大降低

2. 多数派
每次写入都保证大于 N/2 个节点，每次读保证大于 N/2 个节点中读
在并发环境中，无法保证系统正确性，顺序非常重要，要不然结果也不一致
![image](https://user-images.githubusercontent.com/12162133/63226335-1688be80-c20b-11e9-99bc-6b003731274c.png)

3. Paxos 算法
![image](https://user-images.githubusercontent.com/12162133/63226348-36b87d80-c20b-11e9-9a1a-eac6aba3c68f.png)

角色介绍：
- Client：客户端角色
- Propser：提议角色，像议员
- Acceptor：提议投票或接收者，只有在法定人数（Quorum）接收，该提议才会最终被接收，像国会
- Learner：提议接收者，记录员

![image](https://user-images.githubusercontent.com/12162133/63274058-b7da4800-c2d1-11e9-846c-4e08100c5f01.png)
![image](https://user-images.githubusercontent.com/12162133/63275122-8ebab700-c2d3-11e9-9277-6741f6fd7b2c.png)
![image](https://user-images.githubusercontent.com/12162133/63275435-1bfe0b80-c2d4-11e9-87e6-3551a1249b35.png)
![image](https://user-images.githubusercontent.com/12162133/63275471-2a4c2780-c2d4-11e9-90f8-ee0f57e543b0.png)
![image](https://user-images.githubusercontent.com/12162133/63275505-32a46280-c2d4-11e9-89e3-4c7fa465c430.png)

Basic Paxos 存在活锁问题，通过 random timeout 来解决。
Basic Paxos 的问题难实现、效率低（2轮RPC）、活锁

Multi Paxos 算法
因为有多个 Proposer，可能存在活锁问题，Multi Paxos 只有一个 Proposer，可以保证提案的顺序，唯一一个提案，也可以说是唯一一个 leader，不需要两轮 RPC。有点类似 Master-Slave 架构。
![image](https://user-images.githubusercontent.com/12162133/63650526-ff534f00-c77d-11e9-9e66-e6d08685562c.png)

Raft 算法（简单的 Multi Paxos）
划分成三个子问题
- Leader Election：timeout 时钟
- Log Replication：心跳包 写入日志 commit
- Safety：失败 分区
重定义角色
- Leader
- Follower
- Candidate
https://raft.github.io/ raft 动画
http://thesecretlivesofdata.com/raft/ 演示

一致性和共识是否相同，一致性还需要 client 的行为。因为共识是没问题的，但是一致性还需要client来确定的。
![image](https://user-images.githubusercontent.com/12162133/63652009-cae78f00-c78d-11e9-9159-0635e678137c.png)
![image](https://user-images.githubusercontent.com/12162133/63701452-8d4f3880-c857-11e9-9229-f618ea4210ec.png)
![image](https://user-images.githubusercontent.com/12162133/63701565-c5567b80-c857-11e9-8430-ce53b1c9e711.png)
一致性是需要客户端和共识共同达到的
ZAB 协议，作为 zookeeper 的一致性协议，基本与 Raft 相同，如 ZAB 将一个 leader 的周期称为 epoch，而 Raft 称为term。
![image](https://user-images.githubusercontent.com/12162133/63652019-01bda500-c78e-11e9-83dd-73d892575b28.png)

Zookeeper 实例
![image](https://user-images.githubusercontent.com/12162133/63702162-000ce380-c859-11e9-84da-8e2c60fdb897.png)

ETCD 实例

ZAB
ZAB（Zookeeper Atomic Broadcast）原子广播协议为分布式协调服务专门设计的一种支持崩溃恢复的原子性广播协议，当初在设计的时候，并不是一种通用的分布式一致性算法，它是一种特别为 zookeeper 设计的崩溃可恢复的原子消息广播算法。具体的，zookeeper 使用一个单一的主进程来接收并处理客户端所有事务请求，并采用 ZAB 的原子广播协议，将服务数据的状态变更以事务 Proposal 的形式广播到所有的副本进程上去。此外当 leader 节点挂掉后，必须可以恢复并且能够继续工作。
ZAB 协议的核心是定义了对于那些会改变 zookeeper 服务器数据状态的事务请求的处理方式：所有事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被称为 leader 服务器，而余下的服务器被称为 Follower 服务器。

ZAB 包括两种基本模式：崩溃恢复和消息广播模式。
- 消息广播
![image](https://user-images.githubusercontent.com/12162133/64116820-ef62ed00-cdc5-11e9-957d-417c5c1d6c5f.png)
ZAB 协议中涉及的二阶段提交过程则与策略有所不同，在 ZAB 协议中，移除了中断操作。ZAB 协议中的二阶段提交中的中断逻辑移除意味着我们可以在过半的 Follower 服务器已经反馈 ACK 之后就开始提交事务 Proposal，而不需要等待集群中所有的 Follower 服务器都反馈响应，整个消息广播协议是基于具有 FIFO 特性的 TCP 协议来进行网络通信的，因此很容易保证消息广播过程中消息接收和发送的顺序性。

在整个消息广播过程中，Leader 服务器会为每个事务请求生成对应的 Proposal 来进行广播，并分配一个全局自增ID(ZXID)。在消息广播过程中，Leader 服务器会为每一个 Follower 服务器都各自分配一个单独的队列，然后将需要广播的事务 Proposal 依次放入队列中，并且根据 FIFO 策略进行消息发送。每个 Follower 服务器在接收到这个事务 Proposal 之后，都会首先将以事务日志的形式写入到本地磁盘中，并且在成功写入后反馈给 Leader 服务器一个 ACK 响应，当 Leader 服务器接收到过半 Follower 服务器的响应后，就会广播一个 Commit 消息给所有的 Follower 服务器以通知其进行事务提交，同时 Leader 自身也会完成事务的提交。

- 崩溃恢复
![image](https://user-images.githubusercontent.com/12162133/64119841-589a2e80-cdcd-11e9-8995-5c5caf7c5ace.png)
假设一个事务在 Leader 服务器上被提交了，并且已经得到过半 Follower 服务器的 Ack 反馈，但是它将Commit 消息发送给所有 Follower 机器之前，Leader 服务器挂了。
![image](https://user-images.githubusercontent.com/12162133/64119851-5f28a600-cdcd-11e9-90a4-33ef54d9ec31.png)
![image](https://user-images.githubusercontent.com/12162133/64120054-d3634980-cdcd-11e9-9139-492b5f74dbb1.png)

综合上面两种情况，ZAB 协议必须设计这样一个 Leader 选举算法，能够确保提交已经被 Leader 提交的事务 Proposal，同时丢弃已经被跳过的事务 Proposal。选举出来最高编号（即ZXID最高）的事务 Proposal。如果让具有最高编号事务的 Proposal 的机器成为 Leader，就可以省去服务器检查 Proposal 的提交和丢弃工作这一步操作了。

- 数据同步
Leader 服务器会首先确认事务日志中的所有 Proposal 是否已经被集群中过半机器提交了，即是否完成数据同步。具体 Leader 服务器会为 Follower 准备一个队列，并在每一个 Proposal 消息过后再发一个 Commit 消息，以表示该事务已经被提交。
在 ZAB 协议当中，ZXID 是一个 64位的数字，其中低 32 位表示简单的递增计数器，高 32 位代表了Leader 的周期 epoch 的编号，每当选举产生一个最大事务 Proposal 的 ZXID，就解析在原有的 epoch 上加1，然后自增为0开始。基于这样的策略，当一个包含上一个 Leader 周期中尚未提交过的事务 Proposal 的服务器启动时，其肯定无法成为 Leader，当这台机器加入到集群中，以 Follower 方式加入到集群中，Leader 服务器会根据自己服务器最后被提交的 Proposal 来和 Follower 服务器作对比，回退脏数据以及更新新的数据。

### 深入 ZAB 协议
进一步可以分为 发现、同步、广播阶段。
- 发现




## BASE
BASE 理论是对 CAP 理论的延伸，核心思想是即使无法满足一致性（强一致性），但应用可以采用适合的方式达到最终一致性。ASE— Basically Available、Soft State、Eventual Consistency
- 基本可用（Basically Available）：基本可用是指分布式系统在出现故障的时候，允许损失部分可用性，即保证核心可用。
- 软状态：软状态是指允许系统存在中间状态，而该中间状态不会影响系统整体可用性。分布式存储中一般一份数据至少会有三个副本，允许不同节点间副本同步的延时就是软状态的体现。mysql replication的异步复制也是一种体现。
- 最终一致性：最终一致性是指系统中的所有数据副本经过一定时间后，最终能够达到一致的状态。弱一致性和强一致性相反，最终一致性是弱一致性的一种特殊情况。
总的来说，BASE理论面向的是大型高可用可扩展的分布式系统，和传统的事物ACID特性是相反的，它完全不同于ACID的强一致性模型，而是通过牺牲强一致性来获得可用性，并允许数据在一段时间内是不一致的，但最终达到一致状态。但同时，在实际的分布式场景中，不同业务单元和组件对数据一致性的要求是不同的，因此在具体的分布式系统架构设计过程中，ACID特性和BASE理论往往又会结合在一起。



























































































