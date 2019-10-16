# JVM 内存结构
参考这篇博客 [JVM](https://github.com/JeremyWang2009/blogs/issues/1)
# 对象已死吗？
在堆里面存放着几乎所有的对象实例，垃圾收集器回收对象之前第一件事情就是判断这些对象是否存活着，哪些对象已经死去，可以被垃圾收集器回收。常用的垃圾检查算法有两种：（1）引用计数算法（2）可达性分析算法。
## 引用计数算法
给对象添加一个引用计数器，每当该对象被引用，它的计数器 +1，当引用失效后，计数器 -1，在任何情况下，当计数器为 0，表示对象不再被使用。
缺点就是它很难解决对象之间的相互引用，引起循环引用问题，会导致这些对象无法释放内存。因此，主流的 JVM 没有选择该算法监测对象实例。
## 可达性分析算法
在主流的 JVM 基本都使用可达性分析算法来判断对象是否存活，这个算法的基本思想是通过一系列“GC Roots”的对象作为起始点，搜索所走过的路径称为引用链（Reference Chain）， 当一个对象没有任何引用链与 GC Roots 相连时，代表对象不再引用，将其判定为可回收的对象。
![image](https://user-images.githubusercontent.com/12162133/57173198-9195b700-6e5e-11e9-88a1-bbd3f6a5fd85.png)
GC Roots 对象包括以下几种：
 - 虚拟机栈中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 本地方法栈中 JNI 引用的对象

总结来说，就是方法中引用的对象；类的静态变量引用的对象；类的常量引用的对象；Native 方法中引用的对象。
# 垃圾收集算法
## 标记-清除算法
分为两个阶段，标记阶段和清除阶段，首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。
缺点主要有两个：（1）一个是效率问题，标记和清除两个过程效率都不高（2）一个是空间问题，标记清除之后会产生大量不连续的内存碎片。
## 复制算法
它将可用内存按容量划分大小相等的两块，每次只能使用其中的一块，当这一块内存用完时，就将存活的对象复制到另一块上面，然后将已经使用过的内存空间一次性清理掉。
缺点：将内存缩小为原先的一半，对内存空间耗费比较大
## 标记-整理算法
将原有标记-清除算法进行改造，不是直接对可回收对象进行清理，而是让所有存活对象都向一端移动，然后直接清理掉端边界以外的内存。
![image](https://user-images.githubusercontent.com/12162133/57174705-839f6080-6e75-11e9-806e-2fecd487ca08.png)
## 分代收集算法
根据对象存活周期的不同将内存分为几块，分为**新生代和老年代**，这样就可以根据各个年代的特点采用最适当的收集算法。在新生代中，每次垃圾收集时都发现有大批的对象死去，只有少量对象存活，那就选用复制算法，只需要付出少量存活对象的复制成本就可以完成收集。而老年代中因为对象存活率较高，没有额外的空间对它进行分配担保，就必须使用标记-清理或标记-整理算法来进行内存回收。
# HotSpot 算法的实现
## 枚举 GC Roots
可作为 GC Roots 的节点主要在全局性的引用（例如常量或类静态属性）与执行上下文（栈帧中本地变量表）中，现在很多应用仅仅方法区就有几百 M，如果要逐个检查里面的引用，那么必然会耗费很多的时间，
另外，可达性分析对执行时间的敏感还体现在 GC 停顿上，因为这项工作必须在一个能确保一致性的快照中执行，不可以出现分析过程中对象引用关系还在不断地变化的情况，该点不满足的话分析结果准确性就无法得到保证。这点是导致 GC 进行时必须停顿所有 Java 线程（Sun 将这件事情称为“Stop-The-World”）的一个重要的原因，即使是在号称几乎不会停顿的 CMS 收集器中，枚举根节点时也是必须要停顿的。
 
## 理解 GC 日志
![image](https://user-images.githubusercontent.com/12162133/57979029-13363900-7a4a-11e9-8aba-9362d541fa1d.png)
最前面的数字代表 GC 发生的时间，这个数字的含义代表了是从 Java 虚拟机启动以来经历过的秒数；GC 和 Full GC 说明了这次垃圾收集停顿的类型，如果包含 Full，则说明这次 GC 是发生了 Stop The World，如下日志里面显示新生代收集器 ParNew 的日志也会出现 Full GC（一般是因为出现了分配担保失败之类的问题，所以才导致 STW）
![image](https://user-images.githubusercontent.com/12162133/57979247-aae95680-7a4d-11e9-8b5d-7f701c166d55.png)
DefNew、Tenured、Perm 代表了 GC 发生的区域，当然区域名称的显示和 GC 收集器是密切相关的。后面括号里面 3324K->152K(3721K) 表示 “GC 前该内存区域已使用容量->GC 后该内存区域已使用容量（该内存区域总容量）”。再往后，0.0025925 secs 表示该内存区域 GC 所占用的时间，单位是秒。

## Object.finalize
```
RFR 9: 8165641 : Deprecate Object.finalize
Roger Riggs Roger.Riggs at Oracle.com 
Fri Mar 10 21:40:30 UTC 2017
Previous message: RFR 8176195/9, Fix misc module dependencies in jdk_core tests
Next message: RFR 9: 8165641 : Deprecate Object.finalize
Messages sorted by: [ date ] [ thread ] [ subject ] [ author ]
Finalizers are inherently problematic and their use can lead to 
performance issues,
deadlocks, hangs, and other problematic behavior.

The problems have been accumulating for many years and the first step to
deprecate Object.finalize and the overrides in the JDK to communicate the
issues, recommend alternatives, and motivate changes where finalization 
is currently used.

The behavior of finalization nor any uses of finalize are not modified 
by this change.
Most of the changes are to suppress compilation warnings within the JDK.

Please review and comment.

Webrev:
http://cr.openjdk.java.net/~rriggs/webrev-finalize-deprecate-8165641/

Issue:
    https://bugs.openjdk.java.net/browse/JDK-8165641

Thanks, Roger


Previous message: RFR 8176195/9, Fix misc module dependencies in jdk_core tests
Next message: RFR 9: 8165641 : Deprecate Object.finalize
Messages sorted by: [ date ] [ thread ] [ subject ] [ author ]
More information about the core-libs-dev mailing list
```
弃用 Object 类的方法将会是一件非常不寻常的事情。Java 从 1.0 开始就有了 finalize() 方法，不过这个方法一直被认为是一个糟糕的设计，也是 Java 平台的一个遗留的大“毒瘤”。
垃圾回收器会特别对待覆盖了 finalize() 方法的对象。一般情况下，在垃圾回收期间，一个无法触及的对象会立即被销毁。不过，覆盖了 finalize() 方法的对象会被移动到一个队列里，一个独立的线程遍历这个队列，调用每一个对象的 finalize() 方法。在 finalize() 方法调用结束之后，这些对象才成为真正的垃圾，等待下一轮垃圾回收。
不过，析构并不能安全地实现资源的自动管理，因为垃圾回收器并没有运行时间上的保证。也就是说，并不存在任何一种机制可以把资源的释放与对象的生命周期完全绑定在一起，如果处理不好还会耗尽资源。
多年来，Oracle（以及之前的 Sun）建议开发者避免在一般的应用里使用析构。弃用析构意味着向彻底移除迈出了第一步，不过现在能做的也就是在使用析构时给出编译警告。









































