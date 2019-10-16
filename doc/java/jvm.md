## JVM是什么？
      在Java中，理解JDK、JRE、JVM的区别至关重要,利用JDK开发Java程序，通过JDK中的javac将.java文件编译成.class文件，在JRE上运行.class文件，JVM解析字节码，映射到CPU指令集或OS的系统调用。JVM是整个Java实现跨平台的最核心部分，实现Java程序 write once，run anywhere.
     Java语言和JVM相对独立，还可以在JVM上运行的语言如Groovy、Scala等。
![image](https://user-images.githubusercontent.com/12162133/38684601-a27ed8fe-3ea2-11e8-8812-12e577c7c72b.png)
     1.基础类库：java.lang、java.io、java.sql等
     2.基本组件：javac、jar、javadoc、jdb、java、javap、jconsole、jvisualvm等
延伸阅读：https://docs.oracle.com/javase/8/docs/

# JVM Memory Management
![image](https://user-images.githubusercontent.com/12162133/38685497-a4b0bafa-3ea4-11e8-8b9c-3bbbcef4fecf.png)
## 程序计数器
       当前线程所执行的字节码的行号指示器，分支、循环、跳转、异常处理、线程恢复等基础功能都需要这个计数器来完成。Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个时刻，处理器的一个内核都只会执行一条线程的指令。
       如果线程执行一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令地址；如果正在执行Native方法，这个计数器值为空

## 虚拟机栈
      每个线程拥有自己的栈，栈包含每个方法执行的栈帧，栈帧包含局部变量表、操作数栈、动态链接、方法出口等信息。栈是一个后进先出（LIFO）的数据结构，因此当前执行的方法在栈的顶部。每次方法调用时，一个新的栈帧创建并压栈到栈顶。当方法正常返回或抛出未捕获的异常时，栈帧就会出栈。 
    （1）局部变量表：基本数据类型、对象引用类型（reference类型，它不等同于对象本身，可能是一个指向对象起始地址的引用指针，也可能是句柄 ）和 returnAddress类型（指向了一条字节码指令的地址 ），其中64位长度long、double类型的数据会占用2个局部变量空间，其余数据类型会占用一个。局部变量表所需的内存空间在编译期确定。
    （2）操作数栈：后进先出LIFO，最大深度由编译期确定，用于存放JVM从局部变量表复制的常量或变量，也用于存储调用方法需要的参数及接受方法返回的结果
    （3）动态链接：每个栈帧都指向了运行时常量池中该栈帧所属方法的引用，
    （4）方法出口：有两种方式退出该方法，遇到了方法返回的字节码或没有捕获的异常。无论何种退出方式，都需要返回到方法调用的位置，程序才能继续执行。方法退出的过程相当于把当前栈帧出栈，恢复上层方法的局部变量表和操作数栈，如果有返回值，则把它压入调用者栈帧的操作数栈中。  

## 本地方法栈
      本地方法栈则为虚拟机使用的Native方法服务

## 堆
      所有的对象实例以及数组都要在堆上分配，堆是垃圾收集管理的主要区域，又被称为GC堆（Garbage Collected Heap），从内存回收的角度来看，现在收集器基本都采用分代收集算法： 新生代（Eden S0 S1）、老年代、持久代。根据Java虚拟机规范的规定，Java堆可以处理物理上不连续的内存空间，只要逻辑上连续就可以，当前主流的虚拟机都是按照可扩展实现的。

## 方法区
     用于存储加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。对于习惯在HotSpot虚拟机上开发、部署程序的开发者说，很多人都愿意把方法区称为“永久代”，本质上两者不等价，仅仅是因为Hotspot的垃圾收集分代收集扩展至方法区，目前来说，使用方法区来实现永久代，更容易出现内存溢出问题（-XX:MaxPermSize）的上限，最典型的场景就是在jsp页面比较多的情况下。对这块区域回收很难，主要集中在常量池的回收和对类型的卸载。

## 直接内存
    JDK1.4中加入了NIO，非阻塞的IO方式。DirectByteBuffer中的unsafe.allocteMemory(size)是一个native方法，这个方法分配的是直接内存（堆外内存），底层是通过C的malloc分配的。分配的内存是系统本地的内存，不属于JRE

![image](https://user-images.githubusercontent.com/12162133/38686308-9cebdfaa-3ea6-11e8-9344-89ac9aa95b9f.png)

1.运行时常量池
    Class文件除了有类的版本、字段、方法、接口等描述信息外，还有一项信息是常量池，用于存放编译期生成的各种字面量和符号引用，这部门内容将在类加载后进入方法区的运行时常量池存放。避免频繁的创建对象和销毁对象而影响系统性能，实现了对象的共享，例如字符串常量池，在编译阶段就把所有字符串文字放到一个常量池中。

  （1）字面量：文本字符串、被声明为final常量值、基本数据类型的值和其他

  （2）符号引用：类和结构的完全限定名、字段名称和描述符、方法名称和描述符

# 元空间？
![image](https://user-images.githubusercontent.com/12162133/38686393-c2ec57a2-3ea6-11e8-940d-6d80b0c18249.png)

## 取消永久代
     JDK1.8已经从HotSpot JVM中移除了永久代，方法区中的类信息被移至本地内存(native memory)，字符串常量、静态变量被移到堆中。取消永久代，原因大致如下：
   （1）由于 PermGen Space 由 -XX:MaxPermSize 参数来决定大小，JVM在启动时会分配一块连续的内存块，但随着动态类加载情况越来越多，经常会出现OOM Error:PermGen 。如果配置过小，容易出现OOM异常；配置过大，造成内存浪费
   （2）这是JRockit JVM和HotSpot JVM融合努力的一部分，JRockit JVM没有永久代一说

## 概述
      方法区中的类信息被移至本队内存，这块空间称为元空间(Metaspace)。元空间的背后一个思想，类和它的元数据的生命周期和它的类加载器的生命周期是一致的；每个加载器的存储区叫做“a metaspace”，这些“metaspace”一起总体称为“the metaspace”。仅仅当类加载器被回收，该类加载器对应的元空间才可以回收。

     元空间使用块分配器(chunk allocator)来管理元空间的内存分配，块(chunk)的大小依赖于类加载器的类型，其中有一个全局空闲块列表(a global free list of chunks)。当类加载器需要一个块的时候，类加载器从全局块列表中取出一个块，添加到它自己维护的块列表中。当类加载器死亡的时候，它的块将被释放，归还给全局空闲块列表。

     块会进一步被划分成blocks，每个block存储一个元数据单元(a unit of metadata)，元空间使用由mmap分配。

![image](https://user-images.githubusercontent.com/12162133/38686442-e491f9e8-3ea6-11e8-8f22-a448958fad68.png)

## 参数

-XX:MetaspaceSize=size | 初始元空间大小，控制元空间发生GC的初始阈值，这个值的增加或者减少取决于已使用元数据大小，默认大小取决于平台，范围从12MB到20MB。
-- | --
-XX:MaxMetaspaceSize=size | 控制元空间最大值，并不会在JVM启动时候分配这么大内存。默认无大小限制，受限于系统本地内存大小。为防止无限制而导致metaspace内存泄漏而被OS杀掉，建议设置默认值
-XX:MinMetaspaceFreeRatio=size | 最小的Metaspace剩余空间容量百分比，当执行Metaspace GC之后，会计算当前Metaspace的空闲时间比，如果空闲比小于这个数，那么虚拟机将增长Metaspace的大小。
-XX:MaxMetaspaceFreeRatio=size | 最大的Metaspace剩余空间容量百分比，当执行Metaspace GC之后，会计算当前Metaspace的空闲时间比，如果空闲比大于这个数，那么虚拟机会释放Metaspace的部分空间。

延伸阅读：http://java-latte.blogspot.hk/2014/03/metaspace-in-java-8.html
