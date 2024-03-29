olatile是怎样实现了？比如一个很简单的Java代码：

instance = new Instancce()  //instance是volatile变量

在生成汇编代码时会在volatile修饰的共享变量进行写操作的时候会多出Lock前缀的指令（具体的大家可以使用一些工具去看一下，这里我就只把结果说出来）。我们想这个Lock指令肯定有神奇的地方，那么Lock前缀的指令在多核处理器下会发现什么事情了？主要有这两个方面的影响：

将当前处理器缓存行的数据写回系统内存；
这个写回内存的操作会使得其他CPU里缓存了该内存地址的数据无效

为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存（L1，L2或其他）后再进行操作，但操作完不知道何时会写到内存。如果对声明了volatile的变量进行写操作，JVM就会向处理器发送一条Lock前缀的指令，将这个变量所在缓存行的数据写回到系统内存。但是，就算写回到内存，如果其他处理器缓存的值还是旧的，再执行计算操作就会有问题。所以，在多处理器下，为了保证各个处理器的缓存是一致的，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。因此，经过分析我们可以得出如下结论：

Lock前缀的指令会引起处理器缓存写回内存；
一个处理器的缓存回写到内存会导致其他处理器的缓存失效；
当处理器发现本地缓存失效后，就会从内存中重读该变量数据，即可以获取当前最新值。

这样针对volatile变量通过这样的机制就使得每个线程都能获得该变量的最新值。



## JMM
关于定义的理解这是一个仁者见仁智者见智的事情。出现线程安全的问题一般是因为主内存和工作内存数据不一致性和重排序导致的，而解决线程安全的问题最重要的就是理解这两种问题是怎么来的，那么，理解它们的核心在于理解java内存模型（JMM）。

一般在多个线程条件下，为了性能优化，还会涉及到编译器的指令重排和处理器的指令重排。我们知道CPU的处理速度和主存的读写速度不是一个量级的，为了平衡这种巨大的差距，每个CPU都会有缓存。因此，共享变量会先放在主存中，每个线程都有属于自己的工作内存，并且会把位于主存中的共享变量拷贝到自己的工作内存，之后的读写操作均使用位于工作内存的变量副本，并在某个时刻将工作内存的变量副本写回到主存中去。JMM就从抽象层次定义了这种方式，并且JMM决定了一个线程对共享变量的写入何时对其他线程是可见的。
![image](https://user-images.githubusercontent.com/12162133/65619318-86504d00-dff2-11e9-90dd-b6d3fdb842f4.png)
从横向去看看，线程A和线程B就好像通过共享变量在进行隐式通信。这其中有很有意思的问题，如果线程A更新后数据并没有及时写回到主存，而此时线程B读到的是过期的数据，这就出现了“脏读”现象。可以通过同步机制（控制不同线程间操作发生的相对顺序）来解决或者通过volatile关键字使得每次volatile变量都能够强制刷新到主存，从而对每个线程都是可见的。

重排序：
在不改变执行结果的情况下，尽可能提高并行度。
![image](https://user-images.githubusercontent.com/12162133/65619481-cca5ac00-dff2-11e9-858c-da7e4519ce81.png)

happens-before 规则：
1）如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。

程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
程序中断规则：对线程interrupted()方法的调用先行于被中断线程的代码检测到中断时间的发生。
对象finalize规则：一个对象的初始化完成（构造函数执行结束）先行于发生它的finalize()方法的开始。

- 原子性
- 一致性
- 顺序性





