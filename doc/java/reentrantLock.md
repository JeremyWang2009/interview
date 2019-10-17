# 引言
一般的应用系统中，存在着大量的计算和大量的 I/O 处理，通过多线程可以让系统运行得更快。但在 Java 多线程编程中，会面临很多的难题，比如线程安全、上下文切换、死锁等问题。
## 线程安全
引用 《Java Concurrency in Practice》 的作者 Brian Goetz 对线程安全的定义：
>线程安全，当多个线程访问一个对象时，如果不用考虑这些线程在运行时环境下的调度和交替执行，也不需要进行额外的同步，或者在调用方进行任何其他的协调操作，调用这个对象的行为都可以获得正确的结果，那这个对象就是线程安全的

那么如何实现线程安全呢？互斥同步是最常见的一种线程安全保障手段，同步是指在多个线程并发访问共享数据时，保证共享数据在同一时刻只被一条线程使用。而互斥是实现同步的一种手段，临界区、互斥量和信号量都是主要的互斥实现方式。

在 Java 中，最基本的互斥同步手段就是 synchronized 关键字，synchronized 有两种使用形式，分别为同步方法和同步代码块：
```
/**
 * 同步方法在执行前先获取一个监视器，如果是一个静态方法，监视器关联这个类，如果是一个实例方法， 
 * 则关联这个调用方法的对象。同步方法是隐式的，同步方法常量池中会有一个 ACC_SYNCHRONIZED 标 
 * 志，当某个线程访问某个方法的时候，会检查是否有 ACC_SYNCHRONIZED，如果有，则需要先获得监 
 * 视器锁，然后开始执行方法，方法执行之后再释放监视器锁，这时候如果有其他线程来请求执行该方法， 
 * 会因为无法获得监视器锁而被阻塞住。
 */
class Test {
    int count;
    synchronized void bump（）{
        count++;
    }
    static int classCount;
    static synchronized void classBump（）{
        classCount ++;
    }
}
```
```
/** 
 * 同步块在执行前先获取一个监视器，监视器关联括号里面 expression 引用的对象。 同步代码块则是使用 
 * monitorenter 和 monitorexit 两个指令实现的，monitorenter 可以理解为加锁，monitorexit 可以理解为 
 * 释放锁，每个对象自身维护着一个被加锁次数的计数器，当计数器为 1 时，只有获得锁的线程才能再次获 
 * 得锁，即可重入锁，当计数器为0时表示任意线程可以获得该锁。
 * synchronized 的 expression 只能是引用类型，否则会发生编译错误，如果为空，则会抛出空指针异常，
 * 不管同步块中的代码是否正常执行完，监视器都会解锁。
 */
synchronized (expression){
   // block code
}
```
那么对象如何与监视器关联呢，在 Java 中，对象包含三块：对象头、实例数据、填充数据，其中对象头中就包含 Mark Word，Mark Word 一般存储对象的 hashCode、GC分代年龄以及锁信息，锁信息就包含指向互斥量（重量级锁）的指针，指向了一个监视器；监视器是通过 ObjectMonitor 来实现的，代码如下：
```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
}
```
从上面代码可以看到有 ObjectMonitor 两个队列,分别是 _WaitSet 和 _EntryList，_owner 指向持有 ObjectMonitor 对象的线程，当多个线程获取到对象 monitor 后进入 _owner 区域，并把 _owner 设置为指向当前线程，并把 _count 数量加1；当调用 wait() 方法后，将释放当前持有的 monitor，_owner 置为空，_count 减 1 操作，同时，将该线程进入 _WaitSet 集合中等待唤醒，总结如下图：
![image](https://user-images.githubusercontent.com/12162133/59607828-a4322a00-9146-11e9-8ca2-52d420da5fbf.png)

## 上下文切换
现代操作系统中，运行一个程序，系统会为它创建一个进程。现代操作系统调度的最小单元是线程，也叫轻量级进程（Light Weight Process），在一个进程里可以创建多个线程，这些线程都拥有各自的计数器、堆栈和局部变量等属性，并且能够访问共享内存变量。

主要有三种方式实现线程：内核线程实现、用户线程实现、用户线程加轻量级进程混合实现。Java 里面的 Thread 类，它的所有关键方法都是声明 Native 的，和平台有关的方法。从 JDK 1.2 起，对于 Sun JDK 来说，它的 Windows 版与 Linux 版都使用一对一的线程模型实现，一条 Java 线程就映射到一条轻量级进程中，程序一般不会直接去使用内核线程，而是去使用内核线程的一种高级接口——轻量级进程，轻量级进程就是我们通常意义上所说的线程，由于每个轻量级进程都由一个内核线程支持，因此，只有先支持内核级线程，才能有轻量级进程。这种轻量级进程与内核线程之间 1:1 的关系称为一对一的线程模型。
![image](https://user-images.githubusercontent.com/12162133/59609877-d47bc780-914a-11e9-8f59-7f487d5adb6a.png)

由于有内核线程的支持，每个轻量级进程都成为一个独立的调度单元，即使有一个轻量级进程阻塞了，也不会影响整个进程的工作，但是轻量级进程有它的局限性，每个轻量级进程都需要一个内核线程的支持，因此轻量级进程要消耗一定的内核资源，因此一个系统支持轻量级进程的数量是有限的，其次，系统调用的代价相对较高，需要在用户态（User Mode）和内核态（Kernal Mode）中来回切换，线程上下文切换直接的损耗CPU寄存器需要保存和加载, 系统调度器的代码需要执行, TLB实例需要重新加载, CPU 的pipeline需要刷掉。对于抢占式操作系统来说：
- 当前执行任务的时间片用完之后，系统CPU正常调度下一个任务
- 当前执行任务碰到IO阻塞，调度器将此任务挂起，继续下一任务
- 多个任务抢占锁资源，当前任务没有抢到锁资源，被调度器挂起，继续下一任务
- 用户代码挂起当前任务，让出CPU时间
- 硬件中断

综上所述，在 JDK 1.5 之前，通过 synchronized 关键字是保证线程安全的一种重要手段，但是 synchronized 是一个重量级锁，为什么说 synchronized 是一个重量级锁呢？因为 synchronized 依赖于操作系统的 MutexLock（互斥锁）来实现的，且等待获取锁的线程将会阻塞，被阻塞的线程不会消耗 CPU，但是阻塞或唤醒一个线程都需要涉及到上下文切换，涉及到用户态和内核态的切换，所以是比较耗时的。

那除了 synchronized 之外，我们还有其他的方案吗？答案是肯定的，我们可以使用 java.util.concurrent 的 ReentrantLock，但是在了解 ReentrantLock 之前，我们先了解下同步器框架。

# 同步器框架
## 概要
在 JDK1.5 的 java.util.concurrent 包中，大部分的并发类都是基于 AbstractQueuedSynchronizer（简称 AQS） 这个简单的同步器框架构建的。这个框架为原子性管理同步状态、阻塞和唤醒线程、排队等提供了一种通用的机制。

提供了一个框架，用于实现依赖先进先出（FIFO）等待队列的阻塞锁和相关同步器（semaphores、events 等），这些同步器依赖于单个原子 int 值来表示同步状态，提供了原子方法更新状态 getState()、 setState(int)、compareAndSetState(int, int)。

AQS 是一个抽象类，并没有对并发类提供了一个统一的接口定义，而是由子类根据自身的情况实现相应的方法，AQS 中一般包含两个方法 acquire(int)、release(int)，获取同步状态和释放同步状态，AQS 根据其状态是否独占分为**独占模式**和**共享模式**。
- 独占模式：同一时刻最多只有一个线程获取同步状态，处于该模式下，其他线程试图获取该锁将无法获取成功。
- 共享模式：同一时刻会有多个线程获取共享同步状态，处于该模式下，其他线程试图获取该锁可能会获取成功。

这个类还定义了 AbstractQueuedSynchronizer.ConditionObject，实现了 Condition 接口，用于支持管程形式的 await、signal 操作，这些操作与独占模式的 Lock 类有关，且 Condition 的实现天生就和与其关联的Lock 类紧密相关。
## 设计与实现
同步器的背后的基本思想非常简单，acquire 操作如下：
```
while (synchronization state does not allow acquire) {
    enqueue current thread if not already queued;
    possibly block current thread;
}
dequeue current thread if it was queued;
```
release 操作如下：
```
update synchronization state;
if (state may permit a blocked thread to acquire)
    unblock one or more queued threads;
```
为了实现上述 acquire、release 操作，需要完成以下三个功能：
- 同步状态的原子性管理；
- 线程的阻塞与唤醒；
- 排队机制；
### 同步状态
AQS 类通过使用单个 int 类型来保存同步状态，并提供了 getState()、 setState(int)、compareAndSetState(int, int) 三个方法来读取和更新状态，并且此同步状态通过关键字 volatile 修饰，保证了多线程环境下的可见性，compareAndSetState(int, int) 是通过 CAS(Compare and swap，比较并交换) 实现的，当多个线程同时对某个资源进行 CAS 操作的时候，只能有一个线程操作成功，但并不会阻塞其他线程，其他线程会收到操作失败的信号，CAS 是一个轻量级的乐观锁。CAS 的底层通过 Unsafe 类实现的，利用处理器提供的 **CMPXCHG** 指令实现其原子性，使得仅当同步器状态为一个期望值的时候，才会被原子的更新成目标值，相比 synchronized 不会导致过多的上下文切换切换和挂起线程。在 java.util.concurrent 包中，大量地使用 CAS 来实现原子性。
### 线程的阻塞和唤醒
LockSupport 类是一个非常方便的线程阻塞工具类，它可以在线程任意位置让线程阻塞。和Thread.suspend() 相比，它弥补了由于 resume() 在前发生，导致线程无法继续执行的情况。和Object.wait() 相比，它不需要先获得某个对象的锁，也不会抛出 InterruptedException 异常。LockSupport.park() 方法阻塞当前线程，这是因为 LockSupport 类使用类似信号量的机制，它为每一个线程准备了一个许可，如果许可可用，那么 park() 会立即返回，并且消费这个许可（设置许可不可用），就会阻塞，而 unpark() 则使得一个许可变为可用（但是和信号量不同的是，许可不可累加可用，你不可能拥有超过一个许可，它永远只有一个）。
### 排队机制
**同步队列**
AQS 整个框架的关键都是如何管理被阻塞线程的队列，在 AQS 中，运用到了 CLH 锁的思想，CLH 锁被用于自旋锁，可以确保没有饥饿感，提供先到先得的公平服务，CLH锁是基于列表的可伸缩，高性能，公平和自旋锁，应用程序线程仅在局部变量上旋转，它不断轮询前驱状态，如果发现预释放锁定结束旋转。 

AQS 同步队列是一个 FIFO 队列，在此同步队列中，一个节点表示一个线程，它保存着线程的引用、状态、前驱节点、后继节点。同步队列通过两个节点 tail 和 head 来存取，初始化时，tail、head 初始化为一个空节点，线程要加入到同步队列中，通过 CAS 原子地拼接为新的 tail 节点，线程要退出队列，只需设置 head 节点指向当前线程节点。
![image](https://user-images.githubusercontent.com/12162133/59964570-05863e80-9535-11e9-950c-a73c5c6a78c2.png)

同步队列的优点在于其出队和入队的操作都是无锁的、快速的。为了将 CLH 锁队列用于阻塞同步器，该同步队列需要做些额外的修改以提供一种高效的方式定位某个节点的后继节点，在自旋锁中，一个节点只需改变其状态，下一次自旋中其后继节点就能注意到这个改变。但是在阻塞式同步器中，一个节点需要显示地唤醒其后继节点。同步队列包含一个 next 链接到它的后继节点。第二个对 CLH 锁队列主要的修改是将每个节点都有的状态字段用于控制阻塞而非自旋。

基本的 acquire 操作的最终实现一般形式（互斥、非中断、无超时）：
```
if(!tryAcquire(arg)) {
	node = create and enqueue new node;
    pred = node's effective predecessor;
    while (pred is not head node || !tryAcquire(arg)) {
        if (pred's signal bit is set)
            park();
        else
            compareAndSet pred's signal bit to true;
        pred = node's effective predecessor;
    }
    head = node;
}
```
release 操作如下：
```
if(tryRelease(arg) && head node's signal bit is set) {
    compareAndSet head's bit to false;
    unpark head's successor, if one exist
}
```
acquire 的操作的主循环次数依赖于具体实现类 tryAcquire 的实现方式。支持取消或超时的操作主要是在 acquire 循环里的 park() 返回时检查中断或超时，由超时或中断而被取消等待的线程会设置其节点状态，然后 unpark 其后继节点。由于“取消”操作，该线程再也不会被阻塞，节点的链接和状态字段可以被快速重建。

尽管同步队列是基于 FIFO 的，但它们并不一定是公平的，可以注意到基础的 acquire 算法中，tryAcquire 是在入队前被执行的，因此一个新的 acquire 线程能够”窃取“本该属于队列头部的第一个线程获取通过同步器的机会，可闯入FIFO 策略通常会提供比其他技术更高的吞吐率。当然，需要严格的公平性需求时，可以改写 tryAcquire 方法定义，可以通过框架提供的 getFirstQueuedThread() 方法检查是否是头节点，如果是则获取同步状态，如果不是则返回失败。

**等待队列**
AQS 框架提供了一个 ConditionObject 类，给维护独占同步的类及实现 Lock 接口的类使用。一个锁对象可以关联任意数目的条件对象，可以提高类似管程风格的 await、signal 和 signalAll 操作，包括带有超时的以及一些检测、监控的方法。

等待队列是一个 FIFO 队列，在队列中的每个节点都包含一个线程引用，该线程就是在 Condition 对象上等待的线程，如果一个线程调用了 Condition.await() 方法，那么线程将会释放锁，构造成节点将加入等待队列进入等待状态。事实上，节点的定义复用了同步器中节点的定义，也就是说同步队列和等待队列中节点类型都是同步器的静态内部类 java.util.concurrent.locks.AbstractQueuedSynchronizer.Node。
```
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```
如图所示，Condition 拥有首尾节点的引用，而新增节点只要将原有的尾节点 nextWaiter 指向它，并且更新尾节点即可。上述节点的更新并不需要使用 CAS 保证，原因在于调用 await() 方法的线程必定获取了锁的线程，也就是该过程是由锁来保证线程安全的。调用 sigal() 方法将会唤醒在等待队列中等待时间最长的节点（首节点），主要将 CONDITION 状态节点从等待队列中移到同步队列中。 AQS 的图结构如下：
![image](https://user-images.githubusercontent.com/12162133/60019058-82ddba80-96bf-11e9-82da-99d735a27dd7.png)

## 用法
AQS 是一个抽象类，提供了一些重要的方法，如下：
**同步状态管理**

方法 | 说明
-- | --
getState() | 返回同步状态的当前值
setState(int newState) | 设置同步状态的值
compareAndSetState(int expect, int update) | 如果当前状态值等于期望值，则自动将同步状态设置为给定的更新值

**同步方法重写**

方法 | 说明
-- | --
tryAcquire(int arg) | 试图以独占模式获取
tryRelease(int arg) | 试图以独占模式释放
tryAcquireShared(int arg) | 试图在共享模式下获取
tryReleaseShared(int arg) | 试图在共享模式下释放
isHeldExclusively() | 检查当前线程是否在独占模式下占用同步状态

**同步模板方法**

方法 | 说明
-- | --
acquire(int arg) | 以独占模式获取，忽略中断
acquireInterruptibly(int arg) | 以独占模式获取，响应中断
tryAcquireNanos(int arg, long nanosTimeout) | 尝试以独占模式获取，响应中断，如果超时超时将失败
acquireShared(int arg) | 以共享模式获取，忽略中断
acquireSharedInterruptibly(int arg) | 以共享模式获取，响应中断
tryAcquireSharedNanos(int arg, long nanosTimeout) | 尝试以共享模式获取，响应中断，如果超时超时将失败
release(int arg) | 以独占模式释放
releaseShared(int arg) | 以共享模式释放
# 同步器
同步器根据同步状态分为独占模式和共享模式，独占模式包括类：ReentrantLock、ReentrantReadWriteLock.WriteLock，共享模式包括：Semaphore、CountDownLatch、ReentrantReadWriteLock.ReadLock，这里面着重介绍 ReentrantLock 的具体实现，其他有兴趣的同学可以自己研究下。

## ReentrantLock
### 概述
一个可重入互斥 Lock 具有与使用 synchronized 方法和语句访问的隐式监视锁相同的基本行为和语义，但具有扩展功能。ReentrantLock 支持可重入性，并提供了公平锁和非公平锁两种实现方式（默认非公平锁）。因为是独占模式，重写了 tryAcquire、tryRelease、isHeldExclusively 三个方法，支持响应中断、超时机制，下面详细解读下 JDK 1.8 中的 ReentrantLock 的源代码实现。
### 构造方法
提供两种构造方法：（1）无参构造方法，构造非公平锁（2）有参构造方法，根据参数值来构造公平、非公平锁。
```
public ReentrantLock() {
    sync = new NonfairSync();
}
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```
从上面代码看出，公平锁和非公平锁基于类 FairSync、NonfairSync 实现的，而 FairSync、NonfairSync 继承了 Sync，Sync 又继承了 AbstractQueuedSynchronizer，可以看出，Sync 定义了一个抽象方法 lock()，FairSync、NonfairSync 均实现了这个抽象方法，类图关系如下：
![image](https://user-images.githubusercontent.com/12162133/59238990-9f471500-8c32-11e9-9f49-989c04ae0d04.png)
### lock()
从上面通过构造方法创建了一个可重入锁对象，接下来我们开始看下 ReentrantLock.lock() 方法的源代码实现：
```
/**
 * 获得锁的方法，如果锁没有被另一个线程占用并且立即返回，则将锁定计数设置为1。
 * 如果当前线程已经保持锁定，则保持计数增加1，该方法立即返回。
 * 如果锁被另一个线程保持，则当前线程将被禁用以进行线程调度，并且在锁定已被获取之前处于休眠状态，此时锁定保持计数被设置为1。
 **/
public void lock() {
    sync.lock();
}
```
默认实现非公平锁，那我们看下 NonfairSync.lock() 方法实现：
```
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
调用 compareAndSetState(int,int) 方法，如果同步状态期望值是 0，则原子的设置同步状态值为 1，表示当前线程获取同步状态成功，并设置当前锁的占用线程为当前线程。这就体现了非公平锁的性质，不管同步队列是否有线程在排队，直接调用同步状态，这样的设计增强了锁的吞吐率，然而对后面的线程而言不能体现出公平的按照 FIFO 原则获取锁。

如果调用 compareAndSetState(int,int) 失败，表示当前同步状态已经有线程占用了，可能是其他线程也可能是当前线程重入占用了，接着调用 AbstractQueuedSynchronizer.acquire(int) 方法。
```
/**
 * 以独占模式获取（忽略中断），通过调用至少一次 tryAcquire(int) 实现，成功返回。
 * 否则线程排队，可能会重复阻塞和解除阻塞，调用 tryAcquire(int) 直到成功
 **/
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
tryAcquire(int) 方法会调用 Sync.nonfairTryAcquire(int) 方法，代码如下：
```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
从上面代码看出，如果当前同步状态为 0，表示未被线程占用，则调用 compareAndSetState(int,int) 原子设置同步状态为 1，如果设置成功，并设置同步状态占用线程为当前线程，并返回为 true。

如果同步状态的占用线程为当前线程，则为重入性，将当前 state 加 1，否则返回为 false。紧接着会调用 acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) 方法，addWaiter(Node.EXCLUSIVE) 方法的源代码如下：
```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```
上述方法在获取同步状态失败的基础上，创建独占模式的节点，如果尾节点不为空，则将新节点的 prev 指向当前最后一个节点，当前的最后一个节点的 next 指向当前节点，然后 tail 指向当前节点。如果调用失败，则调用 enq() 方法设置尾节点。

在队列中加入节点成功后，接下来会调用 acquireQueued(Node,int) 方法，自旋的获取同步状态，当然前提是当前节点的 prev 节点是 head 节点才能获取同步状态，调用 tryAcquire(int) 方法获取同步状态，如果成功获取到，并将当前节点设置为 head 节点，并将当前节点 prev 设置为 null（因为按照 FIFO 的同步队列原则，因为是非公平锁的模式，所以即使是第一个节点也有可能被新来的线程抢占到）。

如果获取同步状态失败，则调用 shouldParkAfterFailedAcquire(node,node) ，则根据条件判断是否应该阻塞，防止 CPU 处于忙碌状态，会浪费 CPU 资源，如果 prev 节点状态是 SIGNAL 则阻塞，如果是 CANCELLED 状态则向前遍历，移除前面所有为该状态的节点 ，此方法里面阻塞当前节点用的方法是 LockSupport.park()。 
```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```
综上所述 acquireQueued(Node,int) 方法，如果当前线程被中断，则返回为 true，然后调用 selfInterrupt() 方法中断当前线程。

如果是实现公平锁，与非公平锁的不同的是 tryAcquire(int) 方法中，如果同步状态未被占用，在调用 compareAndSetState(int,int) 方法之前，会调用 AQS 中的 hasQueuedPredecessors() 方法，源代码如下：
```
if (c == 0) {
    if (!hasQueuedPredecessors() &&
        compareAndSetState(0, acquires)) {
        setExclusiveOwnerThread(current);
        return true;
    }
}
public final boolean hasQueuedPredecessors() {
    // The correctness of this depends on head being initialized
    // before tail and on head.next being accurate if the current
    // thread is first in queue.
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```
此方法被设计用作是公平锁来用，查询任何线程是否等待获取比当前线程更长的时间，如果当前线程之前有一个排队的线程，则返回为 true；如果当前线程在队列的头部或队列为空，则返回为 false。同样下面 lockInterruptibly()、tryLock(long,long) 方法中公平锁的实现模式也是 tryAcquire(int) 方法与非公平锁不同，关键之处也是调用了 hasQueuedPredecessors() 方法。
### lockInterruptibly()
从方法名字来看，与 lock() 方法不同的是，此方法获取同步锁状态并响应中断，并抛出 InterruptedException 异常，下面我们看下 lockInterruptibly() 方法与 lock() 方法的差别，源代码如下：
```
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```
首先判断当前线程是否已经中断，如果已中断，抛出InterruptedException异常；紧接着和 lock() 方法一样，调用 tryAcquire(int) 方法，非阻塞获取同步状态，如果获取到，则返回，反之获取不到调用 doAcquireInterruptibly(int) 方法，与 lock(int) 方法不同的是，当前方法如果线程中断则直接抛出 InterruptedException 异常，源码如下：
```
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
### tryLock(long,long)
获取同步状态，可响应超时、中断，如果在超时之前获取到同步状态，则返回为 true，如果在超时后还未获取同步状态则返回为 false，或者被其他线程中断了，则抛出 InterruptedException 异常。
```
public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```
与 lockInterruptibly() 方法大同小异，不同的是调用 doAcquireNanos(int,long) 方法，源代码如下：
```
    private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        // 计算超时截止时间（单位：纳秒）
        final long deadline = System.nanoTime() + nanosTimeout;
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                // 计算到截止时间的剩余时间（单位：纳秒）
                nanosTimeout = deadline - System.nanoTime();
                // 超时了，返回 false
                if (nanosTimeout <= 0L)
                    return false;
                // 如果需要阻塞，并且时间大于 1000 纳秒才会阻塞（小于 1000 纳秒认为超时）一定的时间，如果线程中断，则抛出 InterruptedException 异常
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```
### unlock()
尝试释放该锁，如果当前线程是该锁的持有者，则保持计数递减。 如果保持计数现在为零，则锁定被释放。 如果当前线程不是该锁的持有者，则抛出IllegalMonitorStateException，源代码如下：
```
public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
tryRelease(int) 属于 AQS 中的抽象方法，源代码如下：
```
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```
先将当前同步状态减 1 并记录为 c，并判断当前持有锁的线程是不是当前线程，如果不是，则抛出 IllegalMonitorStateException 异常。如果 c 等于 0，并设置当前线程为 null，并设置同步状态为 c，仅当 c 等于 0 才返回 true。

再看 release() 方法，成功释放锁后，唤醒同步队列中的下一个节点，会调用 LockSupport.unpark() 方法释放下一个节点。

综上所述，介绍了 ReentrantLock 了主要方法实现，在 JDK 1.5 及之前，ReentrantLock 提供了比 synchronized 更多的功能，并且在性能上也高。但在 JDK 1.6 及之后，synchronized 的性能和 ReentrantLock 大概持平，JDK 1.6 中加入了很多针对锁的优化，感兴趣的同学可以了解下锁的优化，包括偏向锁、轻量级锁等。

# 总结
j.u.c 大多数同步器都是基于 AQS 同步框架实现的，大量运用乐观锁 CAS、排队机制、阻塞与唤醒工具实现多线程的同步，使得开发人员在多线程编程中更加简单和高效。理解同步器框架的实现与设计，有助于开发人员理解和分析其他同步器。

# 参考
- [aqs](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
- [并发编程网 aqs](http://ifeve.com/aqs/)
- [Java 并发编程的艺术](https://book.douban.com/subject/26591326/)












































































































