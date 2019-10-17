# 概述
该类提供了线程局部 (thread-local) 变量，在多线程环境下，通过 get 或 set 方法时能保证各个线程的变量相对独立于其他线程的变量。
# 设计
刚学习 ThreadLocal 的时候，写了一个 ThreadLocal 的例子：
```
public class ThreadLocalTest {

    public static ThreadLocal<String> threadLocal = new ThreadLocal<String>();

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            final int temp = i;
            final Thread t = new Thread(new Runnable() {
                public void run() {
                    try {
                        threadLocal.set(String.valueOf(temp));
                        Thread.sleep(1000);
                        System.out.println("thread:"+temp+"-"+threadLocal.get());
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
            t.start();
        }
    }
}
```
打印结果如下：
```
thread:2-2
thread:0-0
thread:1-1
thread:5-5
thread:7-7
thread:3-3
thread:6-6
thread:4-4
thread:8-8
thread:9-9
```
按打印结果显示，每个线程的 get 或 set 都是独立的，相互不影响。如果不看源码了解的化，觉得 ThreadLocal 其实底层就是实现了一个 hash map 结构，key 为当前 Thread，value 为要 set 的值。按照这样的设计每个线程都有一个属于自己的变量，在多线程访问情况下，可以使用线程安全的 ConcurrentHashMap。
后面看了 ThreadLocal 的源码，发现 ThreadLocal 根本不是这样设计的。设计图如下：
![image](https://user-images.githubusercontent.com/12162133/55681303-613e2400-5957-11e9-9633-03985f35f90a.png)
从上面图看出，ThreadLocal 的实现原理是这样的，每个 Thread 维护一个 ThreadLocalMap 映射表，这个 Map 的 key 是 ThreadLocal 实例本身，value 是所需要存储的值。ThreadLocal 本身并不存储各个 Thread 的变量值，它只是提供一个 key 来让线程从 ThreadLocalMap 中获取值。从图中的虚线表示，ThreadLocalMap 是使用 ThreadLocal 的弱引用作为 key，弱引用对象在 GC 时会被回收掉。
# 源码
## ThreadLocalMap
ThreadLocalMap 类属于 java.lang.ThreadLocal 包下，是一个静态类，主要成员变量包含如下：
```
# 底层哈希表 table，必要时需要进行扩容，底层哈希表的长度为 2 的 n 次方
private Entry[] table;
# 实际存储键值对元素个数 entries
private int size = 0;
# 下一次扩容时的阈值
private int threshold; // Default to 0
```
其中 Entry[] table ，Entry 包含了 ThreadLocal<?> k 和 Object value。Entry 继承了弱引用 WeekReference ，所以在使用 ThreadLocalMap 的时候，如果 key 为 null，则意味着弱引用的 ThreadLocal 不再使用，需要将其从 ThreadLocalMap 哈希表中删除。
```
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```
## ThreadLocal set method
接下来再看下 ThreadLocal set  method ：
```
/**
 * Sets the current thread's copy of this thread-local variable
 * to the specified value.  Most subclasses will have no need to
 * override this method, relying solely on the {@link #initialValue}
 * method to set the values of thread-locals.
 *
 * @param value the value to be stored in the current thread's copy of
 *        this thread-local.
 */
public void set(T value) {
    # 获取当前线程，然后获取当前线程对象中维护的 ThreadLocalMap 对象
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    # 判断是否为空，如果为空，则调用 createMap 方法进行 ThreadLocalMap 对象的初始化
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```
如果为空，则调用 createMap 方法初始化当前线程 ThreadLocalMap，ThreadLocalMap 是延迟加载的，所以会在至少有一个 entry 存放时候才会初始化，初始化长度为 16。
```
/**
 * Create the map associated with a ThreadLocal. Overridden in
 * InheritableThreadLocal.
 *
 * @param t the current thread
 * @param firstValue value for the initial entry of the map
 */
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
/**
 * Construct a new map initially containing (firstKey, firstValue).
 * ThreadLocalMaps are constructed lazily, so we only create
 * one when we have at least one entry to put in it.
 */
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}
```
如果不为空，则调用 set 方法，如下：
```
/**
 * Set the value associated with key.
 *
 * @param key the thread local object
 * @param value the value to be set
 */
private void set(ThreadLocal<?> key, Object value) {

    // We don't use a fast path as with get() because it is at
    // least as common to use set() to create new entries as
    // it is to replace existing ones, in which case, a fast
    // path would fail more often than not.

    Entry[] tab = table;
    int len = tab.length;
    // 获取对于 threadLocal 的哈希值
    int i = key.threadLocalHashCode & (len-1);
    
    // 循环遍历 table 对应该位置的实体，查找对应的threadLocal
    // 若 key 不一致，且 key 也不为空，则遍历下一个位置，继续查找
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 如果当前 key 和 threadLocal 一致，则赋予新值
        if (k == key) {
            e.value = value;
            return;
        }
        // 如果当前 key 为 null，则替换当前位置的实体
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 如果当前位置为空，则创建新的实体 entry，并存放到当前位置
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```
从上面发现，若存在 hash 冲突，ThreadLocalMap 没有像 HashMap 那样维护着数组链表，而是采用的是**开发地址法**，两者比较如下：
- 开发地址法：容易产生堆积问题；不适于大规模的数据存储；散列函数的设计对冲突会有很大的影响；插入时可能会出现多次冲突的现象，删除的元素是多个冲突元素中的一个，需要对后面的元素作处理，实现较复杂；结点规模很大时会浪费很多空间；
- 链地址法：处理冲突简单，且无堆积现象，平均查找长度短；链表中的结点是动态申请的，适合构造表不能确定长度的情况；相对而言，拉链法的指针域可以忽略不计，因此较开放地址法更加节省空间。插入结点应该在链首，删除结点比较方便，只需调整指针而不需要对其他冲突元素作调整。

ThreadLocal 为什么会选择使用开发地址法呢？因为 ThreadLocalMap 的 key 是 ThreadLocal，这样就造成了这个线程里面的实体不会太多，第二个原因是 ThreadLocalMap 的 hash code 的精妙设计，使得 hash 冲突比较少。
上面说过 ThreadLocalMap 的 Entry 继承了弱引用 WeakReference，如果 ThreadLocal 不再有强引用，则 ThreadLocal 可以被内存回收，则造成了 Entry 中的 key 为 null，这样就没有办法再访问这些 key 为 null 的 Entry 的 value 值了，如果当前线程不结束的化，会造成内存泄露。
那么我们是否可以总结是弱引用带来的内存泄露呢？如果 ThreadLocalMap 的 Entry 继承了强引用，那么如果 ThreadLocal 使用完了，当时线程还没有结束（可能线程池中线程），由于 ThreadLocalMap 属于 Thread，所以生命周期也是一样，因为是强引用，那就会造成 ThreadLocal 也不能内存回收。所以综上所述，造成根本原因并不是引用了弱引用，而是 ThreadLocalMap 属于 Thread。
那么 ThreadLocalMap 的设计中其实已经考虑了这个问题，ThreadLocal get()、set()、remove() 的时候都会清除 ThreadLocalMap 里所有 key 为 null 的 value。
接下来我们继续看 ThreadLocalMap set() 中末尾的一段代码，调用 cleanSomeSlots() 方法，代码如下：
```
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```
若无hash冲突，则先向后检测log2(N)个位置，发现过期 slot 则清除，如果没有任何 slot 被清除，则判断 size >= threshold，超过阀值会进行 rehash()，rehash()会清除所有过期的value。
但是以上方法有点被动，并不能一定保证不会出现内存泄露：
- ThreadLocal 声明为 static，ThreadLocal的生命周期比较长，可能导致的内存泄漏
- 分配使用了 ThreadLocal，又不再调用 get()、set()、remove() 方法

所以要想安全的使用 ThreadLocal，我们可以像使用锁一样来使用 ThreadLocal，每次使用完后都调用 remove() 方法。

 









