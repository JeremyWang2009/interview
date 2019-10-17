# 概述
任意一个 Java 对象，都拥有一组监视器方法（定义在 java.lang.Object 上），主要包括 wait()、notify()、notifyAll() 等。这些方法和 synchronized 同步关键字整合，可以实现通知/等待模式。Condition 接口也实现了类似 Object 的监视器方法，与 Lock 配合可以实现等待/通知模式。
![image](https://user-images.githubusercontent.com/12162133/55815884-bb8be000-5b23-11e9-8254-ec8e628df885.png)

# 实战
Condition 对象是关联锁的，Condition 对象是由 Lock 对象创建出来的。实例如下：
```
public class ConditionTest {

    Lock lock = new ReentrantLock();
    Condition condition = lock.newCondition();

    public void conditionWait() {
        try {
            lock.lock();
            System.out.println("conditionWait begin.");
            condition.await();
            System.out.println("conditionWait end.");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public void conditionSignal() {
        try {
            lock.lock();
            System.out.println("conditionSignal begin.");
            condition.signal();
            System.out.println("conditionSignal end.");
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ConditionTest conditionTest = new ConditionTest();
        Thread t1 = new Thread(new Runnable() {
            @Override
            public void run() {
                conditionTest.conditionWait();
            }
        });
        Thread t2 = new Thread(new Runnable() {
            @Override
            public void run() {
                conditionTest.conditionSignal();
            }
        });
        t1.start();
        Thread.sleep(2000);
        t2.start();
    }
}
```
如上，一般都会将 Condition 对象作为成员变量，当调用 await() 方法后会释放锁并在此等待，而其他线程调用其 signal() 方法，通知当前线程后，当前线程才会从 await() 方法返回，并且在返回前已经获取了锁。
![image](https://user-images.githubusercontent.com/12162133/55878254-a5872980-5bce-11e9-9de6-f2f217612cdb.png)
下面再通过一个有界队列来示例这个 Condition 的用法：
```
public class BoundedQueue<T> {

    private Object[] items;

    private int head;

    private int tail;

    private int count;

    private Lock lock = new ReentrantLock();

    private Condition notEmpty = lock.newCondition();

    private Condition notFull = lock.newCondition();

    public BoundedQueue(int size) {
        items = new Object[size];
    }

    private void add(T t) throws InterruptedException {
        lock.lock();
        try {
            // 使用 while 循环而非 if 判断，目的是防止过早或意外的通知，只有条件符合才能够退出循环
            while (count == items.length) {
                notFull.await();
            }
            items[head] = t;
            if (++head == items.length) {
                head = 0;
            }
            ++count;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    private T remove() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await();
            }
            Object x = items[tail];
            if (++tail == items.length) {
                tail = 0;
            }
            --count;
            notFull.signal();
            return (T) x;
        } finally {
            lock.unlock();
        }
    }
}
```
# 实现分析
从 ReentrantLock 的 newCondition 方法可以看出：
```
public Condition newCondition() {
    return sync.newCondition();
}
```
ConditionObject 是同步器 AbstractQueuedSynchronizer 的内部类，因为 Condition 的操作需要获取相关联的锁，所以作为同步器的内部类也较为合理，每个 Condition 对象都包含着一个队里（等待队里），该队列是 Condition 对象实现等待/通知的功能的关键。
## 等待队列
等待队列是一个 FIFO 队列，在队列中的每个节点都包含一个线程引用，该线程就是在 Condition 对象上等待的线程，如果一个线程调用了 Condition.await() 方法，那么线程将会释放锁，构造成节点将加入等待队列进入等待状态。事实上，节点的定义复用了同步器中节点的定义，也就是说同步队列和等待队列中节点类型都是同步器的静态内部类 java.util.concurrent.locks.AbstractQueuedSynchronizer.Node。
```
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```
![image](https://user-images.githubusercontent.com/12162133/55889569-d45bca80-5be3-11e9-9af7-8d712811cd4e.png)
如图所示，Condition 拥有首尾节点的引用，而新增节点只要将原有的尾节点 nextWaiter 指向它，并且更新尾节点即可。上述节点的更新并不需要使用 CAS 保证，原因在于调用 await() 方法的线程必定获取了锁的线程，也就是该过程是由锁来保证线程安全的。
在 Object 的监视器上，一个对象拥有一个同步队列和等待队列，而并发包中，Lock AQS 拥有一个同步队列和多个等待队列，对应关系如下：
![image](https://user-images.githubusercontent.com/12162133/55890998-5816b680-5be6-11e9-8940-7a6181414401.png)
## 等待
调用 Condition 的 await() 方法，会使当前线程进入等待队列并释放锁，同时线程状态变为**等待状态**。当从 await() 方法返回时，当前线程一定获取了 Condition 相关联的锁。
如果从队列（同步队列和等待队列）的角度看 await() 方法时，相当于同步队列的首节点（获取锁的节点）移动到 Condition 等待队列中。
```
public final void await() throws InterruptedException {
    //  检查是否中断，中断则抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 当前线程加入同步队列，加入到 condition 等待队列的最后一个节点
    Node node = addConditionWaiter();
    // 释放这个锁,并唤醒 AQS 队列中一个线程
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 判断这个节点是否在同步队列上，第一次总是返回为 false
    while (!isOnSyncQueue(node)) {
        // 第一次总是获取线程的许可，进入阻塞
        LockSupport.park(this);
        // 阻塞直到唤醒或中断，由于 LockSupport 响应线程中断，但是不抛出异常，如果中断，则跳出循环；如果没有中断，则继续判断是否在同步队列上
        // 如果在同步队列上，则退出循环，如果不在同步队列，则继续阻塞
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            // 中断，则跳出循环
            break;
    }
    // acquireQueued 使得在同步队列中等待，直到获取到锁，如果在等待同步队列中中断了则将 interruptMode 设置为 REINTERRUPT 状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    // 清楚等待队列中的 cancel 节点
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```
## 唤醒
调用 Condition 的 signal 方法，将会唤醒在等待队列中等待时间最长的节点（首节点），在唤醒节点之前，会将节点移到同步队列中。
![image](https://user-images.githubusercontent.com/12162133/56082477-d32cd680-5e4b-11e9-9704-79172775eaa4.png)
```
/**
 * Moves the longest-waiting thread, if one exists, from the
 * wait queue for this condition to the wait queue for the
 * owning lock.
 *
 * @throws IllegalMonitorStateException if {@link #isHeldExclusively}
 *         returns {@code false}
 */
public final void signal() {
    // 如果当前线程不是持有该锁的线程，抛出异常
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    // 获得等待队列上的第一个节点
    Node first = firstWaiter;
    if (first != null)
        // 唤醒当前节点
        doSignal(first);
}

/**
 * Removes and transfers nodes until hit non-cancelled one or
 * null. Split out from signal in part to encourage compilers
 * to inline the case of no waiters.
 * @param first (non-null) the first node on condition queue
 */
private void doSignal(Node first) {
    do {
        // 如果第一个节点的下一个节点是 null，最后一个节点也是 null
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```
从上面源代码看出，此方法 transferForSignal 肯定会有唤醒操作。
```
/**
 * Transfers a node from a condition queue onto sync queue.
 * Returns true if successful.
 * @param node the node
 * @return true if successfully transferred (else the node was
 * cancelled before signal)
 */
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    Node p = enq(node);
    int ws = p.waitStatus;
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```
从上面代码看出，这个方法主要操作是将 CONDITION 状态的节点从等待队列移到同步队列中去，返回 true 代表成功了。如果失败了，代表这个节点唤醒之前就取消了。
enq(node) 方法是将当前节点放入到 AQS 同步队列中去，并返回当前节点，如果节点的状态是为 CANCEL 或者设置当前节点为 SIGNAL 失败时候，则直接调用 LockSupport 唤醒当前节点。
![image](https://user-images.githubusercontent.com/12162133/56090966-6e20c180-5edb-11e9-88c8-3c1c8abbfd53.png)
![image](https://user-images.githubusercontent.com/12162133/56090976-8395eb80-5edb-11e9-9862-8624e3b1631d.png)
# Java 并发容器和框架
# ConcurrentHashMap
# Java 中的线程池
## 构造方法
合理使用线程池能够带来3个好处：
- 降低资源消耗
- 提高响应速度
- 提高线程的可管理性
![image](https://user-images.githubusercontent.com/12162133/56093191-09c02b00-5ef8-11e9-8e3e-964b90473b79.png)
![image](https://user-images.githubusercontent.com/12162133/56093192-12186600-5ef8-11e9-8ade-aa6c0cde15f8.png)

ThreadPoolExecutor 采取了上述步骤的总体设计思路，是为了执行 execute() 方法时，尽可能地避免获取全局锁（那将会是一个严重的可伸缩瓶颈）。线程池构造器代码如下：
```
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.acc = System.getSecurityManager() == null ?
            null :
            AccessController.getContext();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```
- corePoolSize：核心线程数，如果调用了 prestartAllCoreThreads() 方法，线程池会提前创建并启动所有基本线程
- maximumPoolSize：最大线程数，队列满了才会触发创建线程数直到最大线程数。如果是无界队列，这个参数则没有意义
- workQueue：工作队列
```
ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO 原则对元素进行排序
LinkedBlockingQueue：是一个基于链表结构的阻塞队列，此队列按 FIFO 原则对元素进行排序，此队列吞吐量高于 ArrayBlockingQueue ，静态工厂方法 Executors.newFixedThreadPool() 使用了这个队列
SynchronousQueue：一个不存储元素的阻塞队列，每个插入操作都必须等待另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量高于 LinkedBlockingQueue，静态工厂方法 Executors.newCachedThreadPool 使用了这个队列，如果你希望添加的任务不需要”排队“，直接执行，就可以使用该队列，但要求将 maximumPoolSize 设置得最大以便不会触发拒绝策略。
PriorityBlockingQueue：一个具有优先级的无限阻塞队列
```
- threadFactory：线程工厂，负责创建线程的工厂，可以通过线程工厂给每个创建出来的线程设置更有意义的名字。使用开源框架 gvava 提供的 ThreadFactoryBuilder 可以快速的给线程池的线程设置更有意义的名字。
```
new ThreadFactoryBuilder().setNameFormat("doMyTask-%d").build();
```
- handler：线程拒绝策略
```
AbortPolicy：抛出java.util.concurrent.RejectedExecutionException异常，默认的拒绝策略
CallerRunsPolicy：将待拒绝的任务交由调用者线程来执行
DiscardPolicy：什么都不做，相当于抛弃该任务
DiscardOldestPolicy：丢弃任务队列中最老的任务，再去重试执行待拒绝的任务
```
- keepAliveTime：线程活动保持时间，线程池的工作线程空闲后，保持存活的时间
- TimeUnit：线程活动保持的时间单位

## 提交任务
```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    // workerCountOf 方法获取线程池的工作线程数量，如果小于核心线程数则创建一个 worker 来执行任务
    if (workerCountOf(c) < corePoolSize) { 
        // 创建一个 worker 来执行任务，如果成功，则返回
        if (addWorker(command, true))
            return;
        // 未添加成功，再次获取当前线程
        c = ctl.get();
    }
    // 否则，如果线程池没有 shutdown 就把任务加入到阻塞队列
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```
从上面代码来看，分为四种情况：
- 情况（1）：获取线程池中的有效线程数，如果有效线程数小于核心线程数，则创建线程执行该任务；否则进入情况（2）
- 情况（2）：如果线程池内的有效线程数达到了核心线程数，并且线程池内的阻塞队列未满，则将提交任务到队列中
- 情况（3）：如果线程池内有效线程数大于核心线程数，并且线程池内的阻塞队列已满，那么就创建并启动一个线程来执行新提交的任务
- 情况（4）：如果线程池内有效线程数达到了最大线程数，并且线程池内阻塞队列已满，那么就根据线程拒绝策略处理
上面几种情况的代码分别调用了 addWorker() 方法，并且三次传参都不同，分别为：
```
addWorker(command, true)
addWorker(null, false)
addWorker(command, false)
```
接下来，我们就来正在分析下 addWorker() 方法的源代码：
```
/**
 * Checks if a new worker can be added with respect to current
 * pool state and the given bound (either core or maximum). If so,
 * the worker count is adjusted accordingly, and, if possible, a
 * new worker is created and started, running firstTask as its
 * first task. This method returns false if the pool is stopped or
 * eligible to shut down. It also returns false if the thread
 * factory fails to create a thread when asked.  If the thread
 * creation fails, either due to the thread factory returning
 * null, or due to an exception (typically OutOfMemoryError in
 * Thread.start()), we roll back cleanly.
 *
 * @param firstTask the task the new thread should run first (or
 * null if none). Workers are created with an initial first task
 * (in method execute()) to bypass queuing when there are fewer
 * than corePoolSize threads (in which case we always start one),
 * or when the queue is full (in which case we must bypass queue).
 * Initially idle threads are usually created via
 * prestartCoreThread or to replace other dying workers.
 *
 * @param core if true use corePoolSize as bound, else
 * maximumPoolSize. (A boolean indicator is used here rather than a
 * value to ensure reads of fresh values after checking other pool
 * state).
 * @return true if successful
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    // retry 是一个无限循环，当线程池处于 RUNNGING 状态，只有线程池中的有效线程数被加一，才会退出该循环。
    retry:
    // 无限循环
    for (;;) {
        // 获取线程池的状态
        int c = ctl.get();
        int rs = runStateOf(c);

        // 满足下面条件任何一种，就会返回 false
        // （1）线程池处于 SHUTDOWN、STOP、TIDYING、TERMINATED 状态就会结束
        // （2）线程池处于 SHUTDOWN 状态，并且参数 firstTask 不为空
        // （3）线程池处于 SHUTDOWN 状态，并且阻塞队列为空
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            int wc = workerCountOf(c);
            // 如果线程池内有效线程池数大于或等于最大容量；如果 core 为 true，则不能大于核心线程数
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // CAS 有效线程数增加 1，成功后退出循环
            if (compareAndIncrementWorkerCount(c))
                break retry;
            // CAS 失败的化则继续循环
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 根据参数 firstTask 创建 Worker 对象，并创建线程
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
                
                // 再次检查线程池状态，线程池中添加 worker，然后启动线程执行
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
Worker 类是 ThreadPoolExecutor 类的一个内部类，代码如下：
```
private final class Worker
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker  */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```
Worker 包含 Thread 对象和 Runnable 对象，同时 Wroker 实现了 Runnable 类，而线程的创建详见 Worker 的构造方法，默认调用 DefaultThreadFactory 类创建线程，入参为 Worker 对象，代码如下：
```
public Thread newThread(Runnable r) {
    Thread t = new Thread(group, r,
                          namePrefix + threadNumber.getAndIncrement(),
                          0);
    if (t.isDaemon())
        t.setDaemon(false);
    if (t.getPriority() != Thread.NORM_PRIORITY)
        t.setPriority(Thread.NORM_PRIORITY);
    return t;
}
```
那么 Thread.start() 方法时，实际上是调用 Runnable 的实现类 Worker 中的 run 方法，而 Worker 的 run 方法实际调用 runWorker 方法，代码如下：
```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {

        // task 即为 Worker 中的 firstTask 对象；task 不为空或 getTask() 不为空，实际上 getTask() 是从阻塞队列 workerQueue 中不断取出任务来执行，当阻塞队列的任务都被执行完了就结束下面的循环
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```
## 线程池复用原理
线程池所管理的线程本质上还是  Thread，我们可以看下 Thread.start() 方法代码注释：
```
/**
 * Causes this thread to begin execution; the Java Virtual Machine
 * calls the <code>run</code> method of this thread.
 * <p>
 * The result is that two threads are running concurrently: the
 * current thread (which returns from the call to the
 * <code>start</code> method) and the other thread (which executes its
 * <code>run</code> method).
 * <p>
 * It is never legal to start a thread more than once.
 * In particular, a thread may not be restarted once it has completed
 * execution.
 *
 * @exception  IllegalThreadStateException  if the thread was already
 *               started.
 * @see        #run()
 * @see        #stop()
 */
```
多次启动同一个线程是非法的；特别是，线程完成后可能不会重新启动执行。那么问题就来了，线程池如何实现线程复用，线程池如何实现线程保留 keepAliveTime。从我们上次分析 runWorker 方法得知，一个线程会循环从阻塞队列中获取任务来达到复用的目的。
```
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 当线程池已经关闭，并且任务队列没有任务，立即返回null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 是否考虑获取任务超时：当前线程数已经大于corePoolSize 或者 设置了特定标志
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        // 当前任务队列为空并且(线程数过多或者已经等待超时了)，返回null
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```
## 合理配置线程池
任务的性质：CPU 密集型任务、IO 密集型任务、混合型任务。
- CPU 密集型任务：应配置尽可能小的线程，如 N（CPU）+1 个线程的线程池
- IO 密集型任务：应配置尽可能多的线程，如 2*N(CPU)

## 线程池的监控
如果系统中大量使用线程池，则有必要对线程池进行监控。
![image](https://user-images.githubusercontent.com/12162133/56471863-f67a0600-6489-11e9-9536-d792943a98e7.png)
![image](https://user-images.githubusercontent.com/12162133/56471870-04c82200-648a-11e9-8515-3265a5b1ded8.png)
# Executor 框架
工作单元和执行单元分开，工作单元包括 Runnable 和 Callable，而执行机制由 Executor 框架提供。
![image](https://user-images.githubusercontent.com/12162133/56471902-71432100-648a-11e9-9272-532655aca584.png)
![image](https://user-images.githubusercontent.com/12162133/56471923-a18abf80-648a-11e9-95e7-ef014b318314.png)

Executor 框架主要由 3 大部分组成：
- 任务：包括执行任务需要实现的接口：Runnable 和 Callable
- 任务的执行：Executor  & ExecutorService(extends Executor)接口；及实现 ExecutorService 的 ThreadPoolExecutor 和 ScheduledThreadPoolExecutor。
- 异步计算的结果：包括接口 Future 和实现 Future 接口的 FutureTask 类。

![image](https://user-images.githubusercontent.com/12162133/56472013-f5e26f00-648b-11e9-950d-139dc33d6a56.png)
## ScheduledThreadPoolExecutor
它主要用来给定的延迟之后运行任务，或者定期执行任务，它的功能与 Timer 类似，Timer 对应的是单个后台线程，而 ScheduledThreadPoolExecutor 可以在构造函数中指定多个对应的后台线程数。
ScheduledThreadPoolExecutor 会把待调度的任务 ScheduledFutureTask 放到一个 DelayQueue 中，ScheduledFutureTask 主要包含 3 个成员变量：
- time：表示这个任务将要被执行的具体时间
- sequenceNumber：表示这个任务被添加到 ScheduledThreadPoolExecutor 中的序号
- period：表示任务执行的间隔周期

## DelayQueue
DelayQueue 封装了 PriorityQueue

## Callable Future
下面是关于 Callable Future 的例子：
```
List<Integer> lists = Arrays.asList(1,2,3,4,5,6,7,8,9,10);
List<Callable<Integer>> tasks = lists.stream().map(i -> (Callable<Integer>) ()->{
    int i1 = i + 10;
    return i1;
}).collect(Collectors.toList());
List<Future<Integer>> futures = poolExecutor.invokeAll(tasks);
for(Future<Integer> future : futures){
    System.out.println(future.get());
}
```

在 Java 中一般是用 Thread 类或者实现 Runnable 接口这两种方式来创建多线程，但是这两种不能执行任务完成后返回执行的结果。Callable 接口和 Runnable 接口比较类似，它有一个 call 方法，使用某个线程执行 Callable 接口实质上执行其 call 方法。
```
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```
Future 是对于 Runnable 或  Callable 任务的执行结果进行取消、查询是否完成、获取结果、设置结果操作，get 方法会阻塞。
FutureTask 实现了 Future 接口的方法，我们看 FutureTask 的构造方法：
```
/**
 * Creates a {@code FutureTask} that will, upon running, execute the
 * given {@code Callable}.
 *
 * @param  callable the callable task
 * @throws NullPointerException if the callable is null
 */
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // ensure visibility of callable
}

/**
 * Creates a {@code FutureTask} that will, upon running, execute the
 * given {@code Runnable}, and arrange that {@code get} will return the
 * given result on successful completion.
 *
 * @param runnable the runnable task
 * @param result the result to return on successful completion. If
 * you don't need a particular result, consider using
 * constructions of the form:
 * {@code Future<?> f = new FutureTask<Void>(runnable, null)}
 * @throws NullPointerException if the runnable is null
 */
public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```
可以看到，FutureTask 构造方法中，无论传入是 Runnable 或 Callable，最终都转换成 Callable。FutureTask 则是一个 RunnableFuture<V>，而 RunnableFuture 实现了 Runnable 和 Futrue<V> 这两个接口。由于 FutureTask 实现了 Runnable，因此可以通过 Thread 包装来直接执行。
```
// FutureTask
FutureTask<Integer> futureTask = new FutureTask<Integer>(new Callable<Integer>() {
    @Override
    public Integer call() throws Exception {
        return 10;
    }
});
poolExecutor.submit(futureTask);
System.out.println("futureTask:"+futureTask.get());
```
** FutureTask 的步骤是一个 Callable 设置为 FutureTask 的内置成员，执行 Callable 中的 call 方法，futureTask.get(timeout, TimeUnit) 方法，获取 call 的执行结果，超时的化就报 TimeoutException。**
要想弄清楚 FutureTask 里面的机制，需要看里面的源码，FutureTask.run() 源码如下：
```
public void run() {
    // 判断 state 是否是 NEW，防止并发重复执行
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 调用 call 方法执行
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                // 执行中抛出异常，更新 state 状态，释放等待的线程
                setException(ex);
            }
            if (ran)
                // 执行成功，进行赋值操作
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```
从上面来用到了 state ，代码如下：
```
/**
 * The run state of this task, initially NEW.  The run state
 * transitions to a terminal state only in methods set,
 * setException, and cancel.  During completion, state may take on
 * transient values of COMPLETING (while outcome is being set) or
 * INTERRUPTING (only while interrupting the runner to satisfy a
 * cancel(true)). Transitions from these intermediate to final
 * states use cheaper ordered/lazy writes because values are unique
 * and cannot be further modified.
 *
 * Possible state transitions:
 * NEW -> COMPLETING -> NORMAL
 * NEW -> COMPLETING -> EXCEPTIONAL
 * NEW -> CANCELLED
 * NEW -> INTERRUPTING -> INTERRUPTED
 */
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```
从上面来看，所有大于 COMPLETING 状态的任务都表示任务已经执行完成（正常完成或异常或任务取消），outcome 这个字段是用来存储结果的：
```
/** The result to return or exception to throw from get() */
private Object outcome;
```
接下来我们来看下 get(timeout,TimeUnit) 的源码：
```
/**
 * @throws CancellationException {@inheritDoc}
 */
public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    // 判断 state 状态是否完成，如果未完成，则调用 awaitDone 进行自旋
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    // 根据 state 的状态返回结果或抛出异常
    return report(s);
}
```

 
接下来我们看下 awaitDone 的详细源码：
```
/**
 * Awaits completion or aborts on interrupt or timeout.
 *
 * @param timed true if use timed waits
 * @param nanos time to wait, if timed
 * @return state upon completion
 */
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    // 死循环
    for (;;) {
        // 判断当前线程是否被其他线程中断，如果中断，则调用 removeWaiter() 方法删除当前等待节点
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }

        int s = state;
        // 判断当前状态大于 COMPLETING 表示任务已经完成，则把 thread 字段设置为 null 并返回。
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        // 如果当前状态等于 COMPLETING 表示任务已经执行完成，只是结果还没保存到 outcome 字段中，这个时候线程让步
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        // 如果等待节点为空，则构造一个等待节点
        else if (q == null)
            q = new WaitNode();
        // 如果新建的节点还没加入到队列，则 CAS 加入到队列
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
       // 如果已经超时，则删除对应节点并返回状态 
       else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        // 如果没有超时限制，则一直等待，直到当前任务完成调用 unpark() 方法，结束循环
        else
            LockSupport.park(this);
    }
}
```




























