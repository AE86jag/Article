>Java和C++之间有一堵由内存分配和垃圾收集技术所围成的高墙，墙外面的人想进去，墙里面的人却想出来。对于从事C、C++的程序开发人员，在内存管理上，他们对每个对象担负着从开始到终结的维护责任，但是对于Java程序员来说，在虚拟机自动内存管理机制的帮助下，不容易出现内存泄漏和内存溢出问题，但是一旦出现内存泄漏和内存溢出，如果不了解虚拟机是怎样使用内存的，那么排查错误、修复问题会变得异常艰难。
>
>本文是《深入理解Java虚拟机（第三版）》内存管理部分的总结与整理。

# 一、Java内存区域

为了方便管理和程序执行，Java虚拟机所管理的内存包括以下几个部分：程序计数器、Java虚拟机栈、本地方法栈、Java堆、方法区。为了方便理解，简单画了一个草图，完善了堆的组成（HotSpot虚拟机），整个Java运行时数据区如下：

<img src="https://img-blog.csdnimg.cn/3d2972a3419d4712883ee8f744e753a3.png" alt="Java内存区域" style="zoom:50%;" />

* 程序计数器：当前线程所执行的字节码行号指示器，字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令。线程私有。

* Java虚拟机栈：Java方法执行的内存模型，每个方法从调用到执行完毕，就对应一个栈帧在虚拟机栈中入栈到出栈的过程。线程私有。

* 本地方法栈：与虚拟机栈作用类似，本地方法栈是为本地（Native）方法服务。线程私有。

* Java堆：存放对象实例，几乎所有对象实例都在堆中分配。线程共享。

* 方法区：用于存储已被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。线程共享。

* 运行时常量池：class文件中常量池表在类加载后存放到方法区运行时常量池，符号引用翻译出来的直接引用也存在运行时常量池中

# 二、内存分配策略

首先介绍下一些名词，新生代回收（Minor GC/Young GC），老年代回收（Major GC/Old GC），混合回收（Mixed GC），整个Java堆回收（Full GC）。其中Full GC是会有较大的停顿时间的，称为“Stop the World”。

* 对象是如何创建的？

  首先遇到new指令，先会检查在常量池中能否找到这个类的符号引用，如果没找到，说明该类没有被加载，那么必须先执行类的加载过程；然后为对象分配内存，对象需要多大的内存在类加载阶段就可以确定；设置对象的零值；设置对象头信息；执行构造函数；

* 对象优先在Eden区分配

  大多数情况下，对象在新生代Eden区中分配。当Eden区没有足够空间进行分配时，虚拟机将发起一次Minor GC。

* 大对象直接进入老年代

  大对象是指需要大量连续内存空间的Java对象，大对象对虚拟机分配来说是个坏消息，比大对象更坏的消息是"朝生夕灭"的大对象。需要避免大对象的原因是，在分配内存空间时，它容易导致明明还有不少内存空间，就提前触发垃圾收集，以便获取足够的连续空间安置他们。HotSpot虚拟机有一个参数-XX:PretenureSizeThreshold，指定大于该设置值的对象直接分配在老年代。

* 长期存活的对象进入老年代

  虚拟机给每个对象定义了一个年龄计数器，存储在对象头中，对象通常在Eden区诞生，如果经过一次Minor GC后仍然存活，并且能被Survivor区容纳的话，该对象会被移动到Survivor空间中，并且将其对象年龄设置为1，每在Survivor中熬过一次Minor GC，年龄就加1，当他的年龄增加到一定程度（默认为15），就会被晋升到老年代中。对象老年代的年龄阈值可以通过参数-XX:MaxTenuringThreshold设置。

# 三、内存分配策略实验

实验条件

```
$ java -XX:+PrintCommandLineFlags -version
-XX:InitialHeapSize=266532416 -XX:MaxHeapSize=4264518656 -XX:+PrintCommandLineFlags -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:-UseLargePagesIndividualAllocation -XX:+Us
eParallelGC
java version "1.8.0_121"
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)
```

* 对象优先分配到Eden区

使用的虚拟机参数如下，指定收集器，设置初始堆大小为20M，最大堆大小为20M（堆不可扩展），新生代大小10M，新生代中Eden区和Survivor区比例为8:1，当发生GC时打印GC日志。

```
-XX:+UseParallelGC -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
```

1、首先初始情况下，不分配对象

```java
public class JvmTest1 {
    private static final int _1MB = 1024 * 1024;
    /**
     * -XX:+UseParallelGC -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:SurvivorRatio=8
     */
    public static void main(String[] args) {
        byte[] a, b, c, d;
        //a = new byte[2 * _1MB];
        //b = new byte[2 * _1MB];
        //c = new byte[2 * _1MB];
        //d = new byte[4 * _1MB];
    }
}
```

```
Heap
 PSYoungGen      total 9216K, used 4612K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 56% used [0x00000000ff600000,0x00000000ffa811c8,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 0K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 0% used [0x00000000fec00000,0x00000000fec00000,0x00000000ff600000)
 Metaspace       used 3517K, capacity 4498K, committed 4864K, reserved 1056768K
  class space    used 387K, capacity 390K, committed 512K, reserved 1048576K
```

初始状态，新生代已经使用了4612K的内存，并且没有发生垃圾收集

2、把为变量a分配内存的注释去掉，执行代码，打印日志如下

```
Heap
 PSYoungGen      total 9216K, used 6660K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 81% used [0x00000000ff600000,0x00000000ffc811d8,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 0K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 0% used [0x00000000fec00000,0x00000000fec00000,0x00000000ff600000)
 Metaspace       used 3516K, capacity 4498K, committed 4864K, reserved 1056768K
  class space    used 387K, capacity 390K, committed 512K, reserved 1048576K
```

新生代的使用情况从之前的4612K增长到了6660K，也就是增长了2M，就是a的大小。Eden区使用率也从原来的56%增长到81%，刚好也是2M，这也证明了对象优先分配在Eden区。继续往下操作。注意，这时Eden只剩下1532K。

3、把为变量b分配内存的注释去掉，执行代码，打印日志如下

```
[GC (Allocation Failure) [PSYoungGen: 6496K->1016K(9216K)] 6496K->3173K(19456K), 0.0022893 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 9216K, used 3146K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 26% used [0x00000000ff600000,0x00000000ff814930,0x00000000ffe00000)
  from space 1024K, 99% used [0x00000000ffe00000,0x00000000ffefe010,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 2157K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 21% used [0x00000000fec00000,0x00000000fee1b5c0,0x00000000ff600000)
 Metaspace       used 3518K, capacity 4498K, committed 4864K, reserved 1056768K
  class space    used 387K, capacity 390K, committed 512K, reserved 1048576K
```

这时候，出现了一个GC日志，因为Eden区只有1532K，不能够给b分配2M的内存，这时候虚拟机会发起了一次Minor GC，因为新生代是基于标记-复制算法，虚拟机会将存活的对象复制到survivor区，但是存活的对象a是2M而且还有Eden区无法回收的1016K，survivor区放不下，所以会将之前分配的2M的a对象通过空间分配担保转移到老年代，所以老年代使用了2157K，然后将新分配2M的b对象分配到Eden区，最终Eden使用了8192K * 26% = 2M，from（其中一个survivor区）使用了1016K，老年代使用了2157K。

4、把为变量c分配内存的注释去掉，执行代码，打印日志如下

```
[GC (Allocation Failure) [PSYoungGen: 6496K->1016K(9216K)] 6496K->3220K(19456K), 0.0018073 secs] [Times: user=0.09 sys=0.02, real=0.00 secs] 
Heap
 PSYoungGen      total 9216K, used 5349K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 52% used [0x00000000ff600000,0x00000000ffa3b720,0x00000000ffe00000)
  from space 1024K, 99% used [0x00000000ffe00000,0x00000000ffefe010,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 2204K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 21% used [0x00000000fec00000,0x00000000fee27058,0x00000000ff600000)
 Metaspace       used 3517K, capacity 4498K, committed 4864K, reserved 1056768K
  class space    used 387K, capacity 390K, committed 512K, reserved 1048576K
```

对象c会继续被分配到Eden区

5、把为变量d分配内存的注释去掉，执行代码，打印日志如下

```
[GC (Allocation Failure) [PSYoungGen: 6496K->1016K(9216K)] 6496K->3178K(19456K), 0.0033980 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 PSYoungGen      total 9216K, used 5515K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 54% used [0x00000000ff600000,0x00000000ffa64ee0,0x00000000ffe00000)
  from space 1024K, 99% used [0x00000000ffe00000,0x00000000ffefe010,0x00000000fff00000)
  to   space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
 ParOldGen       total 10240K, used 6258K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 61% used [0x00000000fec00000,0x00000000ff21c858,0x00000000ff600000)
 Metaspace       used 3493K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 384K, capacity 388K, committed 512K, reserved 1048576K
```

因为Eden区只剩下3M左右，无法分配4M的d对象，而且survivor区也被占用满了，所以4M的d对象直接被分配到老年代，最终老年代占用了6M。



* 大对象直接分配到老年代

因为当前实验环境使用的是-XX:+UseParallelGC这个配置，对于-XX:PretenureSizeThreshold这个参数不生效，只要Eden区能分配的下，就在Eden区分配内存，需要指定Serial收集器才能生效，虚拟机参数配置为-XX:+UseSerialGC。这里-XX:PretenureSizeThreshold参数值必须为数字，不能为1M这种，这里使用1.9M。

```
-XX:+UseSerialGC -Xms20M -Xmx20M -Xmn10M -XX:+PrintGCDetails -XX:PretenureSizeThreshold=1992294
```

```
public class JvmTest2 {
    private static final int _1MB = 1024 * 1024;
    public static void main(String[] args) {
        byte[] a, b, c, d;
        //a = new byte[2 * _1MB];
    }
}
```

1、首先注释掉为a分配内存的语句，日志打印如下

```
Heap
 def new generation   total 9216K, used 4612K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  56% used [0x00000000fec00000, 0x00000000ff081160, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 0K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,   0% used [0x00000000ff600000, 0x00000000ff600000, 0x00000000ff600200, 0x0000000100000000)
 Metaspace       used 3516K, capacity 4498K, committed 4864K, reserved 1056768K
  class space    used 387K, capacity 390K, committed 512K, reserved 1048576K
```

从日志前缀可以看出，收集器已经和上节不一样。Eden区使用了56%，大约还剩3.5M，正常是可以分配2M的a变量，继续往下操作

2、将为a分配内存的语句注释去掉，执行代码，日志打印如下

```
Heap
 def new generation   total 9216K, used 4612K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  56% used [0x00000000fec00000, 0x00000000ff081190, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 2048K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  20% used [0x00000000ff600000, 0x00000000ff800010, 0x00000000ff800200, 0x0000000100000000)
 Metaspace       used 3516K, capacity 4498K, committed 4864K, reserved 1056768K
  class space    used 387K, capacity 390K, committed 512K, reserved 1048576K
```

由于设置了对象大小限制为1.9M，2M的a就直接分配到老年代了，老年代占比从0%增加到20%



# 四、垃圾回收

垃圾回收需要考虑的三件事：哪些内存需要回收？什么时候回收？如何回收？

1、哪些内存需要回收？

程序计数器、虚拟机栈、本地方法栈这几个区域都随线程而生，随线程而灭，他们所分配的内存大体认为是在编译期是可知的，内存的分配和回收具有确定性，不需要过多考虑回收，随着方法结束或者线程结束，内存自然就跟着回收了。而Java堆和方法区具有不确定性，这部分的内存回收和分配是动态的，垃圾收集器所关注的正是这部分内存。

2、如何判断对象死亡

* 引用计数法

  在对象中添加一个引用计数器，每当一个地方引用它时，计数器值加一，当引用失效时，计数器减一，任何时刻计数器值为零的对象就是不可能被使用的。客观说引用计数法原理简单，判定效率也很高效，大多数情况下是一个不错的算法。

  但是Java领域都没有使用引用计数法来管理内存，主要原因是这个看似简单的算法有很多例外的情况要考虑，必须要配合大量额外处理才能保证正确的工作，比如单纯的引用计数法就很难解决对象之间相互循环引用的问题。

* 可达性分析算法

  主流商用程序语言的内存管理子系统都是通过可达性分析算法来判断对象是否存活的。这个算法的基本思路是通过一系列成为“GC Root”的根对象为起始节点集，从这些节点开始，根据引用关系向下搜索，所搜过程所走过的路径成为“引用链”，如果对象到GC ROOT间没有引用链相连，则证明对象是不可能再被使用的。

  * 在虚拟机栈中引用的对象

  * 在方法区中静态属性引用的对象

  * 在方法区中常量引用的对象

  * 在本地方法栈中JNI引用的对象

  * Java虚拟机内部的引用，如基本数据类型对应的Class对象、异常对象等

  *  所有被同步锁持有的对象

3、对象的两次标记

​	在可达性算法中判定为不可达的对象也不是“非死不可”的，要真正宣告一个对象死亡，最多会经历两次标记的过程：如果对象在可达性分析后发现没有与GC ROOT相连接的引用链，那么它将会被第一次标记，随后进行一次筛选，筛选条件是此对象是否有必要执行finalize()方法，如果对象没有重写finalize()方法，或者已经执行过，那么这个对象会被直接回收。如果对象被判定有必要执行finalize()方法，那么对象将会被放置在一个名为F-Queue的队列中，并稍后由一条由虚拟机自动建立的、低调度优先级的Finalizer线程去执行他们的finalize()方法，稍后收集器将对F-Queue中的对象进行第二次小规模标记，如果对象在finalize()方法中重新与引用链上任何一个对象建立关联，那么第二次标记时，对象将被移出即将回收的集合，否则对象会被真正回收。



4、安全点、安全区域

* 安全点

  因为枚举完GC Root的引用链，引用关系可能还会发生变化，所以有了安全点的设定，也就决定了用户程序执行时，并非在代码指令流的任意位置都能够停顿下来开始垃圾收集，而是强制要求必须执行到达安全点后才能够暂停。

  让所有线程跑到最近的安全点然后停顿下来，有两种方案，抢先式中断、主动式中断。抢先式中断不需要线程的执行代码主动去配合，在垃圾收集发生时，系统首先把所有用户线程全部中断，如果发现有用户线程中断的地方不在安全点上，就恢复这条线程执行，让它一会再重新中断，直到跑到安全点上。现在几乎没有虚拟机实现采用抢先式中断来暂停线程响应GC 事件。

  主动式中断的思想是当垃圾收集需要中断线程的时候，不直接对线程操作，仅仅简单地设置一个标志位，各个线程执行过程时会不停地主动去轮询这个标志—旦发现中断标志为真时就自己在最近的安全点上主动中断挂起。

* 安全区

  安全区域是指确保在某一段代码片段中，引用关系不会发生变化，因此在这个区域中任意地方开始垃圾收集都是安全的。

  当用户线程执行到安全区城里面的代码时。首优会标识自己已经进人了安全区域。那样当这段时问里虚拟机要发起垃圾收集时就不必去管这些已声明自己在安全区域内的线程了。当线程要离开安全区城时，它要检查虚拟机是否已经完成了根节点枚举（或者垃圾收集过程中其他需要暂停用户线程的阶段），如果完成了，那线程就当作没事发生过，继续执行；否则它就必须一直等待，直到收到可以离开安全区域的信号为止。

5、分代收集理论

弱分代假说：绝大多数对象都是朝生夕灭的。

强分代假说：熬过越多次垃圾收集过程的对象就越难以消亡。

跨代引用假说：跨代引用相对于同代引用来说仅占极少数。

这些奠定了垃圾收集器的一致设计原则：收集器应该将Java堆划分出不同的区域，然后将回收对象依据其年龄分配到不同的区域中存储。

6、垃圾收集算法

|      | 标记-清除                          | 标记-复制                              | 标记-整理                          |
| ---- | ---------------------------------- | -------------------------------------- | ---------------------------------- |
| 优点 | 简单                               | 没有内存碎片                           | 没有内存碎片；整体性能好           |
| 缺点 | 产生内存碎片；随着对象增多效率降低 | 浪费一部分内存；对象存活率高会影响效率 | 移动对象需要更新引用，对系统有负担 |

7、垃圾收集器

Serial、ParNew、Parallel Scavenge、Serial Old、Parallel Old、CMS、G1、Shenandoah、ZGC

他们各有优点和目标，有些是单线程、有些是多线程，有些是适合老年代，有些适合新生代，有些是以吞吐量为目标，有些是以低停顿时间为目标，他们也被嵌入到不同的JDK版本中，书中有每个收集器的详细介绍，关于垃圾收集器本文就省略了。
