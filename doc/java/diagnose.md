# JVM 诊断工具

JDK 官方提供了实验性JDK工具，虽然官方显示是**实验性质**的工具，但是在我们的实际应用中，通过 JDK 工具能够快速监控和诊断问题，所以熟练使用 JDK 工具是每个 Java 程序员的必备技能之一。

```
Experimental JDK Tools and Utilities
NOTE - The tools described in this section are unsupported and experimental in nature and should be used with that in mind. They might not be available in future JDK versions.
```

<a name="31347b51"></a>
# JDK 工具

<a name="fb10aba4"></a>
## 监控工具

- jps
- jstat
- jstatd

<a name="jps"></a>
### jps

jps 命令列出每个 Java 应用程序 lvmid（通常是 JVM 进程的操作系统进程标识符），然后是应用程序类名或 jar 文件名的简短形式。<br />**参数**：<br />-q：只显示 lvmid<br />-m：显示传递给主方法的参数<br />-l：显示应用程序的主类的完整包名或应用程序JAR文件的完整路径名<br />-v：输出传入 JVM 的参数<br />-V：和 -q 功能一样

<a name="jstat"></a>
### jstat

jstat 显示被检测的Java HotSpot VM的性能统计数据，通过 jstat -options 展示 jstat 的命令显示统计的信息。

```
# jstat -options
-class
-compiler
-gc
-gccapacity
-gccause
-gcmetacapacity
-gcnew
-gcnewcapacity
-gcold
-gcoldcapacity
-gcutil
-printcompilation
```

举例几个 jstat 最常用的用法，如下：

```html
sh-4.2# jstat -class 117
Loaded  Bytes  Unloaded  Bytes     Time
 15890 29413.1        0     0.0      13.26
显示类加载、卸载梳理、空间及耗费的时间
```

```
sh-4.2# jstat -gc 117 250 4
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
209664.0 209664.0  0.0   11638.0 1677824.0 451249.1 6291456.0   308488.1  105780.0 98840.0 11728.0 10459.8    271   18.170   4      0.713   18.882
209664.0 209664.0  0.0   11638.0 1677824.0 452330.1 6291456.0   308488.1  105780.0 98840.0 11728.0 10459.8    271   18.170   4      0.713   18.882
209664.0 209664.0  0.0   11638.0 1677824.0 452966.7 6291456.0   308488.1  105780.0 98840.0 11728.0 10459.8    271   18.170   4      0.713   18.882
209664.0 209664.0  0.0   11638.0 1677824.0 454123.7 6291456.0   308488.1  105780.0 98840.0 11728.0 10459.8    271   18.170   4      0.713   18.882
输出 GC 信息，时间间隔每 250 ms，采样 4 个
```

```html
sh-4.2# jstat -gccapacity 117
 NGCMN    NGCMX     NGC     S0C   S1C       EC      OGCMN      OGCMX       OGC         OC       MCMN     MCMX      MC     CCSMN    CCSMX     CCSC    YGC    FGC
2097152.0 2097152.0 2097152.0 209664.0 209664.0 1677824.0  6291456.0  6291456.0  6291456.0  6291456.0      0.0 1142784.0 105780.0      0.0 1048576.0  11728.0    283     4
输出年轻代、老年代、元空间对象使用和占用空间
```

```html
sh-4.2# jstat -gcutil 117
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   5.61  56.31   5.19  93.52  89.20    297   19.793     4    0.713   20.505
展示 GC 各个内存的使用占比、垃圾回收次数和时间，可以在后面加间隔时间和次数，循环打印
```


<a name="M6Ckd"></a>
### jstatd

jstatd 命令是一个 RMI Server 应用程序，提供了对 JVM 虚拟机监控的创建和终止，并提供接口供远程监视工具使用。

<a name="06694c70"></a>
## 诊断工具

- jinfo
- jhat
- jmap
- jsadebugd
- jstack

<a name="jinfo"></a>
### jinfo

展示 JVM 配置信息，配置信息包括 Java 系统属性和 JVM 命令行标志。例子如下：

```
sh-4.2# jinfo 128
展示 JVM 所有的配置信息
```

```
sh-4.2# jinfo -sysprops 128
展示 JVM 系统属性
```

```html
sh-4.2# jinfo -flags 128
sh-4.2# jinfo -flags 139
Attaching to process ID 139, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.201-b09
Non-default VM flags: -XX:CICompilerCount=4 -XX:CMSInitiatingOccupancyFraction=60 -XX:CMSTriggerRatio=70 -XX:ConcGCThreads=8 -XX:ErrorFile=null -XX:GCLogFileSize=524288 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=null -XX:InitialHeapSize=6872367104 -XX:MaxHeapSize=6872367104 -XX:MaxNewSize=2576351232 -XX:MaxTenuringThreshold=6 -XX:MinHeapDeltaBytes=196608 -XX:NewSize=2576351232 -XX:NumberOfGCLogFiles=10 -XX:OldPLABSize=16 -XX:OldSize=4296015872 -XX:ParallelGCThreads=8 -XX:+PrintGC -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseConcMarkSweepGC -XX:+UseFastUnorderedTimeStamps -XX:+UseGCLogFileRotation -XX:+UseParNewGC
Command line:  -Djava.security.egd=file:/dev/./urandom -DAPPID=lpd_team.gpoint -XX:ErrorFile=//data/log/app/lpd_team.gpoint/hs_err_20190731_012858.log -Xms6553m -Xmx6553m -Xmn2457m -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:CMSInitiatingOccupancyFraction=60 -XX:CMSTriggerRatio=70 -XX:ParallelGCThreads=8 -XX:ConcGCThreads=8 -Xloggc:/data/log/app/lpd_team.gpoint/gc20190731_012858.log -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=10 -XX:GCLogFileSize=512K -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/data/log/app/lpd_team.gpoint/heapdump_20190731_012858.hprof
展示 JVM 的扩展参数，主要包含 VM Flags 信息
```

> Oracle recommends setting the minimum heap size (-Xms) equal to the maximum heap size (-Xmx) to minimize garbage collections.
> By default, the Application Server is invoked with the Java HotSpot Server JVM. The default NewRatio for the Server JVM is 2: the old generation occupies 2/3 of the heap while the new generation occupies 1/3. The larger new generation can accommodate many more short-lived objects, decreasing the need for slow major collections. The old generation is still sufficiently large enough to hold many long-lived objects.


<a name="3IJkz"></a>
### jhat

jhat（JVM Heap Analysis Tool），用来分析 Java 堆的工具。jhat 命令用来解析 Java 堆存储文件并启动 web 服务器，允许用户通过浏览器查看生成 Java dump 的分析结果。生成 Java dump 文件目前有这几种方式：

- 通过 jmap -dump 命令获得运行时的 dump 文件
- 通过 jconsole 在运行时通过 HotSpotDiagnosticMXBean 获得 dump 文件，参考：[http://docs.oracle.com/javase/8/docs/jre/api/management/extension/com/sun/management/HotSpotDiagnosticMXBean.html](http://docs.oracle.com/javase/8/docs/jre/api/management/extension/com/sun/management/HotSpotDiagnosticMXBean.html)
- 通过 JVM 运行参数 -XX:+HeapDumpOnOutOfMemoryError ，当 JVM 抛出 OutOfMemoryError 时将生成 dump 文件
- 通过 hprof 命令，参考：[http://docs.oracle.com/javase/8/docs/technotes/samples/hprof.html](http://docs.oracle.com/javase/8/docs/technotes/samples/hprof.html)

```
sh-4.2# jhat -port 8005 dump.dat
```

jhat 通过上述命令分析 dump 文件，然后 `http://localhost:8005`，就可以在浏览器中查看堆的报表信息了。

<a name="jmap"></a>
### jmap

jmap（JVM Memory Map），查看内存映射和堆内存的详细信息。一般通过 jmap 生成 heap dump 文件，然后通过 jhat 命令分析 heap dump 文件。下面看下 jmap 的常用用法：

```html
## 显示堆中的详细信息，包括堆配置参数、各代中堆内存、GC 算法
sh-4.2# jmap -heap 124
Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 8589934592 (8192.0MB)
   NewSize                  = 2147483648 (2048.0MB)
   MaxNewSize               = 2147483648 (2048.0MB)
   OldSize                  = 6442450944 (6144.0MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
New Generation (Eden + 1 Survivor Space):
   capacity = 1932787712 (1843.25MB)
   used     = 1636627632 (1560.8097381591797MB)
   free     = 296160080 (282.4402618408203MB)
   84.67705076138232% used
Eden Space:
   capacity = 1718091776 (1638.5MB)
   used     = 1627902408 (1552.4887161254883MB)
   free     = 90189368 (86.01128387451172MB)
   94.75060824690193% used
From Space:
   capacity = 214695936 (204.75MB)
   used     = 8725224 (8.321022033691406MB)
   free     = 205970712 (196.4289779663086MB)
   4.063991225246108% used
To Space:
   capacity = 214695936 (204.75MB)
   used     = 0 (0.0MB)
   free     = 214695936 (204.75MB)
   0.0% used
concurrent mark-sweep generation:
   capacity = 6442450944 (6144.0MB)
   used     = 147469624 (140.63799285888672MB)
   free     = 6294981320 (6003.362007141113MB)
   2.289029831687609% used
```

```html
## 生成 heap dump 文件
sh-4.2# jmap -dump:format=b,file=dump.dat pid
```

```html
sh-4.2# jmap -histo:live 119 | more
 num     #instances         #bytes  class name
----------------------------------------------
   1:        180218       45507008  [C
   2:        304676        9749632  java.util.concurrent.ConcurrentHashMap$Node
   3:         18045        8270360  [B
   4:        187893        6012576  java.lang.ThreadLocal$ThreadLocalMap$Entry
   5:         58058        5109104  java.lang.reflect.Method
   6:        204412        4905888  java.lang.Long
   7:        174203        4180872  java.lang.String
   8:          1683        3939088  [Ljava.util.concurrent.ConcurrentHashMap$Node;
   9:         45020        3137808  [Ljava.lang.Object;
  10:        175569        2809104  me.ele.ejdbc.pool.filter.MethodFilterChainImpl$IndexWrapper
  11:          1652        2778560  [Ljava.lang.ThreadLocal$ThreadLocalMap$Entry;
  12:         70252        2248064  java.util.HashMap$Node
  13:         16971        1889008  java.lang.Class
  14:         19669        1719600  [Ljava.util.HashMap$Node;
  15:         25728        1440768  java.util.LinkedHashMap
  16:         32025        1281000  java.util.LinkedHashMap$Entry
  17:         50334        1208016  java.util.concurrent.atomic.AtomicLong
  18:         37182         892368  java.util.ArrayList
  19:          7016         869208  [I
  20:         16829         807792  java.util.HashMap
  21:         47297         756752  java.lang.Integer
  22:         15578         747744  org.aspectj.weaver.reflect.ShadowMatchImpl
  23:         29991         648824  [Ljava.lang.Class;
  24:         19728         631296  java.lang.ref.WeakReference
打印堆的直方图，对于每个类，打印对象的数量、以字节为单位的内存大小和完全限定的类名，如果加了 :live 这个可选参数，则只打印活跃对象，并伴随着一次 Full GC（线上谨慎操作）。
```


<a name="UnmV4"></a>
### jsadebugd

jsadebugd命令附加到Java进程或核心文件，并充当调试服务器。jstack、jmap和jinfo等远程客户机可以通过Java远程方法调用(RMI)附加到服务器。

<a name="jstack"></a>
### jstack

jstack 主要打印 Java 进程、core 文件或远程调试服务器的 Java 线程的堆栈信息，对于每个 Java 帧，打印出完整的类名、方法名、字节码索引（BCI）和行号。生成线程快照的主要目的是定位线程长时间停顿的原因，如线程死锁、死循环、请求外部资源长时间等待等问题。

```html
-F：当正常输出的请求不被响应时，强制输出线程堆栈 
-m：如果调用到本地方法的话，可以显示C/C++的堆栈 
-l：打印关于锁的附加信息，如拥有的java.util列表。并发ownable某个浏览器。请参阅AbstractOwnableSynchronizer类描述
```

```html
# jstack 生成 thread dump 文件
sh-4.2# jstack -l pid > thread_dump.txt
```

<a name="7479d999"></a>
# 排障分析

<a name="864d490f"></a>
## CPU 高

模拟 CPU 高的程序代码如下：

```
public class Test {

    public static void main(String[] args) {
        testCPU();
    }

    public static void testCPU() {
        Long num = 0L;
        for (; ; ) {
            num = num + 1;
            if (num == Integer.MAX_VALUE) {
                System.out.println("num reach max value,reset 0");
                num = 0L;
            }
        }
    }

}
```

第一步，通过 top 命令查找占用 cpu 最高的进程，如下图，进程 64207 占用 CPU 达到 98.5

```
[root@046a51733199 project]# top
top - 07:38:55 up 20 days,  6:44,  0 users,  load average: 1.27, 1.37, 1.82
Tasks:  10 total,   1 running,   9 sleeping,   0 stopped,   0 zombie
Cpu(s): 49.6%us,  1.2%sy,  0.0%ni, 49.1%id,  0.0%wa,  0.0%hi,  0.2%si,  0.0%st
Mem:   2047036k total,  1911436k used,   135600k free,    82204k buffers
Swap:  1048572k total,   900540k used,   148032k free,   126644k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
64207 root      20   0 2633m 189m  15m S 98.5  9.5   1:48.36 java
26430 root      20   0 3726m 589m 8184 S  0.3 29.5  58:29.52 java
64348 root      20   0 14948 1924 1708 R  0.3  0.1   0:00.01 top
    1 root      20   0 11364 1136 1040 S  0.0  0.1   2:52.76 sh
20914 root      20   0 3508m 104m 8736 S  0.0  5.2  11:32.14 java
21034 root      20   0 3526m 166m  10m S  0.0  8.3 149:04.16 java
52176 root      20   0 3509m 175m  15m S  0.0  8.8   0:48.74 java
58684 root      20   0 11500 2716 2300 S  0.0  0.1   0:00.22 bash
58738 root      20   0 11500 2708 2288 S  0.0  0.1   0:00.24 bash
64357 root      20   0  4128  460  388 S  0.0  0.0   0:00.00 sleep
```

第二步，通过 top -Hp pid 命令查找该进程中占用 CPU 最高的线程，发现线程 64208 占用 CPU 达到了 94.3

```
[root@046a51733199 project]# top -Hp 64207
top - 07:40:16 up 20 days,  6:45,  0 users,  load average: 1.17, 1.32, 1.77
Tasks:  13 total,   1 running,  12 sleeping,   0 stopped,   0 zombie
Cpu(s): 49.8%us,  3.4%sy,  0.0%ni, 46.4%id,  0.0%wa,  0.0%hi,  0.3%si,  0.0%st
Mem:   2047036k total,  1912168k used,   134868k free,    82292k buffers
Swap:  1048572k total,   900540k used,   148032k free,   126684k cached

  PID USER      PR  NI  VIRT  RES  SHR S %CPU %MEM    TIME+  COMMAND
64208 root      20   0 2633m 189m  15m R 94.3  9.5   3:00.70 java
64211 root      20   0 2633m 189m  15m S  2.0  9.5   0:04.78 java
64209 root      20   0 2633m 189m  15m S  0.7  9.5   0:01.04 java
64210 root      20   0 2633m 189m  15m S  0.3  9.5   0:00.97 java
64207 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.00 java
64212 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.00 java
64213 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.00 java
64214 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.00 java
64215 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.01 java
64216 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.01 java
64217 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.00 java
64218 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.22 java
64314 root      20   0 2633m 189m  15m S  0.0  9.5   0:00.00 java
```

第三步，将线程 ID 转换为 16 进制

```
[root@046a51733199 project]# printf '%x\n' 64208
fad0
```

第四步，通过 jstack 命令打印线程的堆栈信息

```
[root@046a51733199 project]# jstack 64207 | grep fad0 -A 30
"main" #1 prio=5 os_prio=0 tid=0x00007fe1b004b000 nid=0xfad0 runnable [0x00007fe1b699a000]
   java.lang.Thread.State: RUNNABLE
	at Test.testCPU(Test.java:14)
	at Test.main(Test.java:7)

"VM Thread" os_prio=0 tid=0x00007fe1b00d1000 nid=0xfad3 runnable

"GC task thread#0 (ParallelGC)" os_prio=0 tid=0x00007fe1b005e000 nid=0xfad1 runnable

"GC task thread#1 (ParallelGC)" os_prio=0 tid=0x00007fe1b005f800 nid=0xfad2 runnable

"VM Periodic Task Thread" os_prio=0 tid=0x00007fe1b011f000 nid=0xfada waiting on condition

JNI global references: 5
```

从 Thread Dump log 的时效性很高，可以通过命令 jstack 多次打印当前线程的 Dump log，如果多次打印的 Dump log 日志都如图显示 Test 类 14 行，说明该程序代码中含有死循环，导致 CPU 高。

<a name="ea6b7012"></a>
## Dead Lock

死锁是指两个或两个以上的进程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。<br />模拟线程死锁的代码如下：

```
/**
 * Created by shanguang.wang on 2019-04-25
 */
public class Test {

    private static String resourceOne = "1";

    private static String resourceTwo = "2";

    public static void main(String[] args) {
        testDeadLock();
    }

    public static void testDeadLock() {
        Thread threadOne = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    synchronized (resourceOne) {
                        System.out.println(
                                String.format("%s resourceOne at lock", Thread.currentThread().getName()));
                        Thread.sleep(1000);
                        synchronized (resourceTwo) {
                            System.out.println(
                                    String.format("%s resourceTwo at lock", Thread.currentThread().getName()));
                        }
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });

        Thread threadTwo = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    synchronized (resourceTwo) {
                        System.out.println(
                                String.format("%s resourceTwo at lock", Thread.currentThread().getName()));
                        Thread.sleep(1000);
                        synchronized (resourceOne) {
                            System.out.println(
                                    String.format("%s resourceOne at lock", Thread.currentThread().getName()));
                        }
                    }
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        // two thread start
        threadOne.start();
        threadTwo.start();
    }
}
```

运行代码，显示效果如下：

```
[root@046a51733199 project]# java Test
Thread-1 resourceTwo at lock
Thread-0 resourceOne at lock
```

第一步，通过 jps -l 命令显示当前的 Java 进程 pid

```
[root@046a51733199 project]# jps
65968 Test
66152 Jps
```

第二步，通过 jstack 命令查看存在 deadlock 的线程堆栈信息

```
[root@046a51733199 project]# jstack 65968 |grep 'deadlock' -A 30
Found one Java-level deadlock:
=============================
"Thread-1":
  waiting to lock monitor 0x00007f66c0003828 (object 0x00000000f59dbde0, a java.lang.String),
  which is held by "Thread-0"
"Thread-0":
  waiting to lock monitor 0x00007f66c0006218 (object 0x00000000f59dbe10, a java.lang.String),
  which is held by "Thread-1"

Java stack information for the threads listed above:
===================================================
"Thread-1":
	at Test$2.run(Test.java:43)
	- waiting to lock <0x00000000f59dbde0> (a java.lang.String)
	- locked <0x00000000f59dbe10> (a java.lang.String)
	at java.lang.Thread.run(Thread.java:748)
"Thread-0":
	at Test$1.run(Test.java:24)
	- waiting to lock <0x00000000f59dbe10> (a java.lang.String)
	- locked <0x00000000f59dbde0> (a java.lang.String)
	at java.lang.Thread.run(Thread.java:748)

Found 1 deadlock.
```

在 Thread Dump log，有几个重要的信息如下：

- locked <地址>：使用 synchronized 申请对象锁成功，monitor 的 owner
- waiting to lock <地址>：使用 synchronized 申请对象锁未成功，block 在 monitor 的 entry set 队列中
- waiting on <地址>：使用 synchronized 申请对象锁成功，在 monitor 的 wait set 队列中等待

有了以上知识，就可以分析上述的 Thread Dump log 了，"Thread-1" block 在 monitor 的 entry set 队列中，"Thread-0" block 在 monitor 的 entry set 队列中，并且 "Thread-1" 等待的资源被 "Thread-0" 占有，"Thread-0" 等待的资源被 "Thread-1" 占有，详见 Test 类代码第 43 和 24 行。

Thread Dump 中的 Thread State 如下：

| Thread State | Description |
| :---: | --- |
| NEW | The thread has not yet started. |
| RUNNABLE | The thread is executing in the JVM. |
| BLOCKED | The thread is blocked waiting for a monitor lock. |
| WAITING | The thread is waiting indefinitely for another thread to perform a particular action. |
| TIMED_WAITING | The thread is waiting for another thread to perform an action for up to a specified waiting time. |
| TERMINATED | The thread has exited. |


简单的应用程序可以通过 jstack 命令直接分析，但是在生产环境下，有从百上千个线程，而且线程的代码执行速度很快的，如果需要定位一个问题，往往需要生成不同时间对应的 thread dump 文件，且至少两份，然后分析它，而且一般是通过工具来分析 thread dump 文件，常用的工具是 JDK 自带的 jvisualvm，通过装入 thread dump 文件可以分析：<br />
![image](https://user-images.githubusercontent.com/12162133/63209951-4906be80-c11a-11e9-9b02-4d81f215c8cf.png)

<a name="e82cee44"></a>
## OOM 异常

<a name="b5139c13"></a>
### Java heap space

```
public class OOMTest {

    public static void main(String[] args) {
        heapError(Integer.MAX_VALUE);
    }

    public static void heapError(int floor) {
        List<OOMObject> oomObjects = new ArrayList<>();
        for (int i = 0; i < floor; i++) {
            try {
                Thread.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            oomObjects.add(new OOMObject());
            System.out.println("new object i:" + i);
        }

    }

    static class OOMObject {
        public byte[] oom = new byte[64 * 1024];
    }

}
```

```html
java -Xms1g -Xmx1g -Xmn200m -XX:+UseConcMarkSweepGC -XX:+UseParNewGC -XX:CMSInitiatingOccupancyFraction=60 -XX:CMSTriggerRatio=70 -Xloggc:/root/project/tmp/gc_1.log -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/root/project/tmp/heapdump_1.hprof OOMTest
```

基于 JDK 1.8，堆内存为 1g，年轻代 200m，年轻代为 ParNew 垃圾收集器，年老代为 CMS 垃圾收集器，当出现 OOM 异常的时候，记录 OOM 异常信息到文件中。通过上述 Java 命令来运行 OOMTest main 方法，接下来，我们通过如下排查思路来定位程序中存在内存泄露的代码。<br />观察 Java heap 中各代中的内存占比情况

```
[root@046a51733199 tmp]# jps -l
83610 OOMTest
83626 sun.tools.jps.Jps
```

jps 确定运行的 JVM pid 为 83610。

```html
[root@046a51733199 tmp]# jstat -gcutil 83610
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  0.00  93.17   0.00  19.47  19.39      0    0.000     0    0.000    0.000
[root@046a51733199 tmp]# jstat -gcutil 83610
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00 100.00   8.01  16.22  56.56  50.33      1    0.140     0    0.000    0.140
[root@046a51733199 tmp]# jstat -gcutil 83610
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 99.92   0.00  14.36  35.27  56.57  50.33      2    0.309     0    0.000    0.309
[root@046a51733199 tmp]# jstat -gcutil 83610
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00  99.95  40.52  54.30  56.57  50.33      3    0.405     2    0.014    0.419
[root@046a51733199 tmp]# jstat -gcutil 83610
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
 99.96   0.00  18.20  73.40  56.57  50.33      4    0.547     7    0.025    0.572
```

通过 jstat -gcutil 命令来查看 JVM heap 中各个代中的占比，发现年轻代的占比逐渐上升到 99.96%，老年代的占比逐渐上升到 73.40%，YGC 4次年轻代的内存还是回收不掉，FGC 7次老年代的内存也还是回收不掉。

```
[root@046a51733199 tmp]# jmap -dump:format=b,file=dump.dat 83610
Dumping heap to /root/project/tmp/dump.dat ...
Heap dump file created
```

接下来，通过 jmap dump 命令生成当刻的 Heap dump 文件，如上，已经创建。分析内存的工具有 jhat 和 mat 等，接下来分别用这两个工具分析。<br />**jhat 分析 heap dump**

```
[root@046a51733199 project]# jhat -port 8005 dump.dat
Reading from dump.dat...
Dump file created Sun May 19 17:52:59 CST 2019
Snapshot read, resolving...
Resolving 42637 objects...
Chasing references, expect 8 dots........
Eliminating duplicate references........
Snapshot resolved.
Started HTTP server on port 8005
Server is ready.
```

通过 jhat 命令分析 dump.dat 文件，jhat 自带 Web Server，接着可以在浏览器中输入 `http://localhost:8005/` ，展示如下：<br />
![](https://user-images.githubusercontent.com/12162133/57983264-60360180-7a82-11e9-837c-9eeb634d57ea.png#align=left&display=inline&height=1006&originHeight=1006&originWidth=1816&status=uploading&width=1816)
<br />如上，对我们最重要的有两项，“Show heap histogram” 和 “QQL”，“Show heap histogram” 如下：<br />
![](https://user-images.githubusercontent.com/12162133/57983350-5791fb00-7a83-11e9-8abf-da3c26058720.png#align=left&display=inline&height=1238&originHeight=1238&originWidth=2062&status=uploading&width=2062)
<br />"Heap Histogram" 显示了驻留在内存中的所有类、它们的数量和所有对象的总大小，默认情况下，所有类都按照它们占用的总内存大小排序，作为泄漏对象，count将会很高。从上图可以看到，OOMTest$OOMObject 的 count 和 size 最高，然后 B、C、D、F、I、J 代表什么呢，如下：

```
Element Type  	Encoding
boolean	   	Z
byte	   	B
char	   	C
class 	   	Lclassname;
double	   	D
float	   	F
int	   	I
long	   	J
short	   	S
If this class object represents a class of arrays, then the internal form of the name consists of the name of the element type preceded by one or more '[' characters representing the depth of the array nesting.
```

从上面图看到，OOMTest 中 OOMObject 的数量为 12025 个，结合程序来看，代码 `List<OOMObject> oomObjects = new ArrayList<>()`存在内存泄露。<br />**mat 分析 heap dump**<br />mat(Memory Analyzer Tool)，一个基于 Eclipse 的内存分析工具，是一个快速、功能丰富 Java Heap 分析工具，它可以帮助我们查找内存泄露或其他内存问题。<br />
![](https://user-images.githubusercontent.com/12162133/57984247-8e204380-7a8c-11e9-9782-4afd8b318e2d.png#align=left&display=inline&height=1476&originHeight=1476&originWidth=2560&status=uploading&width=2560)

Histogram：从工具栏中选择直方图，列出每个类的实例数、shallow 和 retained 堆的大小<br />Dominator Tree：显示堆转储中最大的对象<br />Leak Report：内存分析器可以检查堆转储是否有泄漏嫌疑，例如对象或一组大得令人怀疑的对象

如上图，一般选择 Reports->Leak Suspects，展示如下图，显示 752M 的内容是有问题的：
#####1
从上图的 Dominator Tree 可以看出，内存泄露的地方为一个 List，里面存储 OOMObject，通过此结合程序定位程序中存在内存泄露的代码的位置。

<a name="M22ya"></a>
## Full GC 频繁
案例：重写 Object.finalize() 方法

<a name="wP9fX"></a>
## Metaspace 异常
在了解 Metaspace 异常之前，对不熟悉 Metaspace 的人可以看 [Metaspace](https://github.com/JeremyWang2009/blogs/issues/1)
### 系统刚刚启动引发 FULL GC
Metaspace 虽然属于堆外内存，但是一些 CMS、G1 垃圾收集器 FULL GC 的时候会回收这块内存，一般 Metaspace 中有两个参数 -XX:MetaspaceSize、-XX:MaxMetaspaceSize 比较重要。
-  -XX:MetaspaceSize：初始元空间大小，控制元空间发生GC的初始阈值，这个值的增加或者减少取决于已使用元数据大小，默认大小取决于平台，范围从12MB到20MB
- -XX:MaxMetaspaceSize：控制元空间最大值，并不会在JVM启动时候分配这么大内存。默认无大小限制，受限于系统本地内存大小。为防止无限制而导致metaspace内存泄漏而被OS杀掉，建议设置默认值

一般的系统中，存放元空间的数据都会大于 20MB，所以元空间就会扩容，扩容就会引起 FULL GC，这样就能解释了在系统刚刚启动过程中会引发 FULL GC，可以看下 gc log:
```
2019-08-18T16:18:45.852+0800: 2.965: [GC (Allocation Failure) 2019-08-18T16:18:45.852+0800: 2.965: [ParNew: 503040K->12493K(565888K), 0.0278516 secs] 503040K->12493K(1614464K), 0.0279763 secs] [Times: user=0.03 sys=0.00, real=0.03 secs]
2019-08-18T16:18:47.427+0800: 4.540: [GC (Allocation Failure) 2019-08-18T16:18:47.427+0800: 4.540: [ParNew: 515533K->62848K(565888K), 0.6903477 secs] 515533K->122056K(1614464K), 0.6904743 secs] [Times: user=0.65 sys=0.03, real=0.69 secs]
2019-08-18T16:18:48.118+0800: 5.231: [GC (CMS Initial Mark) [1 CMS-initial-mark: 59208K(1048576K)] 126310K(1614464K), 0.0232695 secs] [Times: user=0.03 sys=0.00, real=0.02 secs]
2019-08-18T16:18:48.142+0800: 5.254: [CMS-concurrent-mark-start]
2019-08-18T16:18:48.214+0800: 5.326: [CMS-concurrent-mark: 0.072/0.072 secs] [Times: user=0.17 sys=0.01, real=0.07 secs]
2019-08-18T16:18:48.214+0800: 5.327: [CMS-concurrent-preclean-start]
2019-08-18T16:18:48.215+0800: 5.328: [CMS-concurrent-preclean: 0.002/0.002 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]
2019-08-18T16:18:48.215+0800: 5.328: [CMS-concurrent-abortable-preclean-start]
```
- 无论 -XX:MetaspaceSize 怎样配置，Metaspace 的初始容量为 21807104（约 20.8m，linux 系统）
- Metaspace 由于使用不断扩充到 -XX:MetaspaceSize，如果达到，就会发生 FULL GC
- 如果年老代配置 CMS 回收，那么上面触发 FULL GC 将会使用 CMS 算法
- Metaspace 的容量范围为 [20.8m, MaxMetaspaceSize]
- 如果 MaxMetaspaceSize 配置太小，将会频繁导致 FULL GC

```
sh-4.2# jstat -gc 142
 S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT
62848.0 62848.0  0.0   2262.2 503040.0 111487.7 1048576.0   375477.6  80640.0 78248.1 9472.0 8976.1     91    3.773   0      0.000    3.773
sh-4.2# jstat -gcutil 142
  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   3.60  21.89  35.81  97.03  94.76     91    3.773     0    0.000    3.773
```
MC：Klass Metaspace以及NoKlass Metaspace两者总共committed的内存大小
MU：Klass Metaspace以及NoKlass Metaspace两者已经使用了的内存大小
M：MC/MU

# 垃圾收集器
## 垃圾收集算法
### 标记-清除算法
Mark-Sweep 算法，分为标记和清除两个阶段，最基础的回收算法。其主要缺点有两个：一是效率问题，标记和清除的效率过程都不高；另外一个是空间问题，标记清除后会产生大量不连续的碎片。
![image](https://user-images.githubusercontent.com/12162133/63345008-4f986e80-c384-11e9-88a8-204ba0fca1ef.png)
### 复制算法
为了解决效率问题，Copying 算法出现了，它将可用内存按容量划分为大小相等的两块，每次只使用其中一块，这一块内存用完了，就将还存活的对象复制到另外一块内存中，然后再把已使用过的内存空间一次性清理掉，不用考虑内存碎片问题了。其主要缺点是使用内存缩小为原来的一半，代价有点高。
![image](https://user-images.githubusercontent.com/12162133/63345326-18768d00-c385-11e9-8bd1-614245ab3daf.png)

IBM 的专门研究表明，新生代中的对象 98% 都是朝生夕死的，所以并不需要按照 1:1 的比例来划分内存空间，而是将内存分为一块较大的 Eden 区和两块比较小的 Survivor 区，每次使用 Eden 和其中一块 Survivor 区，当回收时，将 Eden 区和 Survivor 区还存活的对象一次性地拷贝到另外一个 Survivor 区中，最后清理掉 Eden 区和已使用 Survivor 区的空间。当然，如果另外一个 Survivor 区没有足够的空间存放上一次新生代收集回来的对象，这些对象直接通过分配担保机制进入老年代。
### 标记-整理算法
复制算法在对象存活率比较高的情况下就要执行更多的复制操作，效率将会变得很低。更关键的是，不想浪费 50%  的存活空间，就需要额外的空间进行分配担保，所以在老年代一般不用复制算法。
根据老年代的特点，提出了另外一种 Mark-Compact 算法，跟 Mark-Sweep 算法差不多，但是第二步是让存活的对象都向一端移动，然后直接清理掉端边界以外的内存。
![image](https://user-images.githubusercontent.com/12162133/63346932-a3a55200-c388-11e9-9ca8-82c5d14987ff.png)

### 分代收集算法
分代收集的算法是基于根据对象的村后周期不同将内存划分为几块，一般是将 Java 堆分为年轻代和年老代，年轻代选择复制算法，年老代采用标记-清理或标记-整理算法。

## 垃圾收集器
![image](https://user-images.githubusercontent.com/12162133/63347151-20d0c700-c389-11e9-9a3d-6a8aab870546.png)
### CMS
CMS（Concurrent Mark Sweep）是一种以获取最短回收停顿时间为目标的收集器，尤其重视服务的响应速度，希望系统停顿时间最短，是基于 Mark-Sweep 的算法实现的，主要步骤包括：
- 初始标记（STW）：初始标记仅仅只是标记一下 GC Roots 能直接关联到的对象，速度很快
- 并发标记：进行 GC Roots Tracing  的过程，
- 重新标记（STW）：重新标记是为了修正并发标记期间因用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间会比初始标记阶段稍长一点，但远比并发标记的时间短
- 并发清除：将标记的对象清除掉



## CMS






















```
sh-4.2# jstat -gcutil 129 1000
    S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
  0.00   3.68  77.17  43.80  95.44  91.84  14809  187.383     8    2.680  190.063
  0.00   3.68  80.48  43.80  95.44  91.84  14809  187.383     8    2.680  190.063
  0.00   3.68  83.70  43.80  95.44  91.84  14809  187.383     8    2.680  190.063
  0.00   3.68  87.03  43.80  95.44  91.84  14809  187.383     8    2.680  190.063
  0.00   3.68  90.27  43.80  95.44  91.84  14809  187.383     8    2.680  190.063
  0.00   3.68  93.58  43.80  95.44  91.84  14809  187.383     8    2.680  190.063
  0.00   3.68  97.17  43.80  95.44  91.84  14809  187.383     8    2.680  190.063
  3.97   0.00   0.89  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
  3.97   0.00   4.23  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
  3.97   0.00   7.81  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
  3.97   0.00  11.43  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
  3.97   0.00  14.85  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
  3.97   0.00  18.23  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
  3.97   0.00  21.56  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
  3.97   0.00  24.92  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
  3.97   0.00  28.43  43.81  95.44  91.84  14810  187.395     8    2.680  190.075
```
从上面来看  S0 和 S1 是交换的












参考：<br />[https://dzone.com/articles/how-to-read-a-thread-dump](https://dzone.com/articles/how-to-read-a-thread-dump)<br />[https://help.eclipse.org/2019-06/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html](https://help.eclipse.org/2019-06/index.jsp?topic=/org.eclipse.mat.ui.help/welcome.html)<br />[https://blog.csdn.net/pursuer211/article/details/83892918](https://blog.csdn.net/pursuer211/article/details/83892918)<br />[https://wiki.ele.to:8090/pages/viewpage.action?pageId=85857656](https://wiki.ele.to:8090/pages/viewpage.action?pageId=85857656)<br />[https://wiki.ele.to:8090/pages/viewpage.action?pageId=118621514](https://wiki.ele.to:8090/pages/viewpage.action?pageId=118621514)





