---
title: JVM调试工具使用
date: 2020-05-07 14:03:53
tags: Java
---

# JVM调试工具使用简介

JVM调试工具是解决线上问题的重要方法。一些常见的线上问题比如说运行时单线程耗时过长、GC无法申请到足够的空间而产生OOM错误。这些线上的错误十分恼人，目前我们对于响应时间过长的处理，大多数是增加log，然后分析哪一步耗时比较长，这种做法并不是总是管用，因为有些多线程的问题可能很难复现，比如死锁，第二是并不是所有的生产环境都跟现在一样自由，我们可以随意部署、重启。至于OOM的处理，我们一直就没有好的办法，现在只能说从数据方面找问题，看是不是数据有问题？或者通过阅读代码对数据量进行估算，这些方法一是非常耗时，二是并不准确。

因此，掌握一些通用的JVM调试工具还是非常必要的。

## JVM内存模型

上面说到的两个问题，都与JVM的内存管理和线程运行有关系，JVM主要的调试工具和很大一部分参数，都是用来调整JVM的堆栈表现，因此，必须先了解JVM的内存模型，才能更好地理解之后调试工具的意义。

在深入了解JVM内存模型之前，有几个问题需要提前明确：

1. JVM内存粗略分为堆内存、栈内存、本地内存三个部分。其中堆内存和栈内存都是使用虚拟机内存，而本地内存则直接使用了宿主机物理内存，JVM内存和虚拟机内存加起来受到物理内存和交换页的限制。在主机主要跑一个JAVA进程的时候，要注意不可能给JVM分配所有的内存，以免本地内存不够用而引发OOM。
2. JVM内存模型并非一成不变，随着JDK版本和垃圾回收机制的不同，JVM的内存实际状况都可能发生变化。比如说，在使用G1垃圾回收器的时候，堆内存就会被分为一块一块的，和使用其他回收期时堆内存连续分配有很大不同。并且自JDK1.8开始，原来存在于hotspot虚拟机中的永久代被移除，原来在永久代的方法区被移动到直接内存中，而运行时常量池被移动到堆中。
3. 堆是JVM管理的最大一部分内存，远远超过栈空间的大小，在绝大多数情况下，也远超使用的本地内存的大小。因此，在JVM启动参数中，根据合理的评估准确设置堆内存大小，是JVM内存管理的一个重要手段。
4. 根据IBM的研究，98%的java对象都在建立以后很快就死亡，没有活过下一次的垃圾回收。

### 内存模型简介

![JVM内存模型](https://www.history-of-my-life.com/imgs-for-md/jvm-memory-20190811.jpg)

**注意**，上面的图是JDK1.7及之前的实现。但是大概意思是这样。

java运行时的数据区域包括程序计数器、java虚拟机栈、本地方法栈、java堆、方法区、运行时常量池和直接内存。

#### 程序计数器

虽然商用虚拟机在实现这个概念时可能比较复杂，但从概念上来理解，程序计数器就是指示下一条命令的地址的一个计数器而已。显然，因为每个栈有自己的运行进度，因此程序计数器显然是栈私有的。因为它基本上只记录一个地址，因此可想而知，它占得内存空间非常小，在我们讨论内存问题时，几乎可以忽略不计，因此，这个区也是唯一一个没有定义什么内存溢出错误的区。

#### java虚拟机栈

一看名字就知道是线程私有的。保存的主要是局部变量、操作数栈、动态链接、方法出口等。这些数据本身并不会占用太多空间，因为new出来的对象原则上仍然是在堆里面分配，根据一些大公司的研究，一般1M的默认栈空间大小足以负担1000-2000的栈深度，基本是够用了。

但是并不是说栈空间在所有情况下都不需要调整，与栈空间相关的异常有两个，一个是真的递归超过了最大栈空间，那就会出现StackOverflowError，如果允许虚拟机栈动态扩展，那么当其向堆扩展时发现没有空间，就会抛出OOM异常。其实这是一个问题的两个表现形式，都是栈空间用的太多了，只有一种情况除外，就是不停地申请线程，导致线程栈的数量太多而导致OOM，在这种情况下可能需要将栈空间调小一点。

与虚拟机栈相关的虚拟机参数有： 

+ -Xss 设置每个线程的栈大小，比如-Xss128k， -Xss1m

#### 本地方法栈

当JVM调用本地方法时，即调用使用其他语言编写，并且编译为物理机上可运行的程序的接口时，就会用到本地方法栈。这只是个概念，一些虚拟机把这个栈和虚拟机栈合二为一了，也没什么问题，比如最常用的hotsopt虚拟机，下面的介绍也主要是基于这个虚拟机。

#### 堆内存

对绝大多数应用来说，这是虚拟机管理的最大的一部分内存，所有线程共享，这个区域唯一的目的就是存放对象，从概念上来讲，所有new出来的对象都会在这里分配内存（有一些优化手段会改变这个事实，但从概念上仍然可以这么认为）。

堆内存的具体结构和垃圾回收的方式紧密相连。比如说，现在主流的分代垃圾回收方式，对新生代采取复制算法，对老年代采取标记-整理算法，因此新生代和老年代一般是两整块区域，而因为新生代采取了复制算法，因此又分为eden区（伊甸园）和两个survivor区，其比例大约是8:1:1。

在大部分情况下，新生代和老年代都是各自占据连续的一整块内存（逻辑上的一连续一整块，物理上未必），在采用了G1垃圾回收器的JVM虚拟机中，eden、survivor、old被划分成大小一样的一些块，一个块默认是1M大小，JVM用空闲列表来维护这些块，其内存构造又有所不同。

堆内存可以扩展，也可以不扩展。固定大小的堆比较稳定，稳定的堆对于维护稳定的老年代大小比较有利，在一定程度上可以减少GC的次数，但是固定大小的堆在实际利用率不高的情况下，会增加每次GC的时长。

与堆内存相关的虚拟机参数茫茫多，整理如下：

+ -Xms 初始堆大小，比如 -Xms1024m。默认情况为物理内存的1/64，在使用容器时比较复杂。
+ -Xmx 最大堆大小，比如 -Xmx8196m。默认情况为物理内存的1/4，在使用容器时比较复杂。当-Xms=-Xmx的值时，堆不会进行扩展和收缩，是稳定的。
+ -Xmn 年轻代大小，比如-Xmn2g。整个堆大小=年轻代大小+年老代大小+永久代大小（要注意到1.8以后就没有永久代了），因此调整年轻代的大小，就会挤占老年代的空间，这个值对系统的影响比较大，需要根据情况进行设置，一般来说，年轻代大小是老年代大小1/2比较稳妥，在较老的JVM设置中，hotspot建议的值是`年轻代/堆总大小=3/8`。比如如果系统中充满了大量短命的大对象，那么增加年轻代的大小，甚至使其大于老年代，都是可以考虑的。
+ -XX:NewRatio 设置年轻代与老年代大小的比值，如-XX:NewRatio=4代表年轻代是老年代的1/4，在设置了Xms=Xmx并且设置了Xmn的情况下，该参数不需要进行设置。
+ -XX:SurvivorRatio eden和survivor之间的大小比例，鉴于98%的对象都是朝生夕死，因此这个比值一般都设置成8:1:1，这也是默认值。

在使用到容器时，JDK1.8_131之前的JVM不能识别容器对于硬件资源的限制，总是能读到物理机的内存，因此，在不设置-Xms和-Xmx时，容器中的JVM虚拟机会按照物理机内存的1/64和1/4去申请，很容易在超出限制后被杀掉。于是需要手动设置这两个值。

自动JDK1.8_131以后，JVM增加了一个实验特性，自动感知容器大小，需要加两个参数以开启，这两个参数如下：

+ -XX:+UnlockExperimentalVMOptions 打开JDK的实验特性。
  + -XX:+UseCGroupMemoryLimitForHeap 要求感知container的大小
  

同时，增加了一个参数来调整优化默认表现。

+ -XX:MaxRAMFraction 比如-XX:MaxRAMFraction=1，表示JVM管理的堆大小是虚拟机内存的多少分之一，默认情况下是4。显然这个值太保守了，在很多情况下，我们的容器主要就跑一个JVM，只用1/4的内存还是太浪费了。于是一般需要调整这个值到2，但是随着容器内存的增大，1/2的浪费也不能接受了，但一旦申请成1，因为元空间的存在，就必然爆内存。
  

针对这种情况，有两个解决方案，一个是手动设置-Xms和-Xmx。但这又太不灵活了，假如我改了容器的内存，就又要手改参数，所以最好是用容器的EntryPoint来解决这个问题。

第二种解决方案依赖于更高版本的JDK，至少在我们使用的JDK1.8_191中可以使用下列参数来配置：

+ XX:+UseContainerSupport  作用如其名
  
  + XX:MaxRAMPercentage 比如XX:MaxRAMPercentage=90.00，意思是使用容器内存的90%作为JVM管理的堆。具体使用多少还需要根据实际环境来调整。
  
#### 方法区

它用于存储已被加载的虚拟机的类信息、常量、静态变量、即时编译器编译后的代码等数据。这些都是java加载时产生的一些数据。在JDK1.7以前，这个区在所谓的永久代，但是JDK1.8以后，这个区被移动到直接内存中去了，直接内存也有个好听的名字，叫元空间（metadataSpace）

一般来说，这个区域不怎么需要垃圾回收，垃圾回收的效果也不是很令人满意。但是在大量使用反射、动态代理或者字节码织入技术的框架中，因为可能生成比较多的动态类，这部分的空间也有可能面临压力，如果没有空间了，就会抛出异常。

现在因为这个区使用了直接内存，因此要牢记JVM内存+直接内存一起受到物理内存的限制，如果给JVM内存分配的空间过大，挤占了直接内存的大小，那么有可能会出现堆明明不怎么满，但却OOM的问题，或者出现因为内存换页，导致运行速度大幅下滑的问题。

在一些大量使用了NIO的框架中，也要注意直接内存的分配大小。

与此相关的参数包括：

+ -XX:MaxDirectMemorySize 调整直接内存大小。
  
#### 运行时常量池

JDK1.7之前，运行时常量池放在永久代，和方法区放在一起，JDK1.7之后，已经把运行时常量池移动到堆中了。经查询，网文中关于这一块的细节部分讲的不多，《深入理解JVM虚拟机》那么书又是基于1.7的。所以下面这段话请存疑地看。

JDK1.8以后，字符串常量移动到了堆中，但是基本类型包装类等运行时常量池还是在方法区，也就是在元空间。在java中，所有的基本类型包装类都有常量池，比如：

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}
```

显然用到了常量池，而这个上限是可以通过虚拟机参数来调整的。

```
-Djava.lang.Integer.IntegerCache.high=xxxx
```

但是下限不可调整。

特别的，应该关注一下字符串常量池，目前它在堆中，而且可以通过`String.internal()`进行动态地添加，因此需要关注OOM问题。

#### 直接内存-元空间

主要在JDK1.8以后用于存放方法区、字符串常量以外的常量池，以及被使用了NIO的框架使用。在设置JVM的堆内存时，应当考虑到元空间的大小，避免堆内存对着物理内存上限去设置，否则将会出现比较难以排查的OOM问题。

与此相关的JVM参数包括，注意这些参数只有在JDK1.8以上才可用

+ -XX:MetaspaceSize=128m  元空间的初始化大小
+ -XX:MaxMetaspaceSize=512m 元空间的最大值

### 内存回收机制简介

#### 啥样的对象算是死亡了

##### 引用计数器

为每个对象维护一个引用计数器，引用计数器为0就认为死了。这个思想很常用，比如说Cpp的智能指针。但用在垃圾回收上有个很大的问题，就是循环引用时会导致发生循环引用的对象引用计数器永远不是0，实际上不能访问的对象也不能清除。

##### 可达性分析法

即从一组根节点出发进行遍历，可达的对象我们都认为存活，不可达的对象当然就是死了。这个方法听起来不错，这一组根节点被称为`GC roots`。这个方法最重要的是定义`GC roots`有哪些。

+ System Class（由bootstrap装载的类）

  Class loaded by bootstrap/system class loader. For example, everything from the rt.jar like java.util.*

+ JNI Local（native code的局部变量）

  Local variable in native code, such as user defined JNI code or JVM internal code.

+ JNI Global（native code的全局变量）

  Global variable in native code, such as user defined JNI code or JVM internal code. 

+ Thread Block（JVM虚拟机线程栈）

  Object referred to from a currently active thread block. 

+ Thread（状态不是TERMINTED的线程）

  A started, but not stopped, thread. 

+ Busy Monitor（被用作锁的对象）

  Everything that has called wait() or notify() or that is synchronized. For example, by calling synchronized(Object) or by entering a synchronized method. Static method means class, non-static method means object. 

+ Java Local（局部变量）

  Local variable. For example, input parameters or locally created objects of methods that are still in the stack of a thread. 

+ Native Stack（本地方法栈）

  In or out parameters in native code, such as user defined JNI code or JVM internal code. This is often the case as many methods have native parts and the objects handled as method parameters become GC roots. For example, parameters used for file/network I/O methods or reflection. 

+ Finalizable（在Finalizable队列中等待的对象）

  An object which is in a queue awaiting its finalizer to be run. 

+ Unfinalized（重写了finalize方法，但是还没有被finalize过的对象）

  An object which has a finalize method, but has not been finalized and is not yet on the finalizer queue. 

+ Unreachable（像是自定义的根这样的）

  An object which is unreachable from any other root, but has been marked as a root by MAT to retain objects which otherwise would not be included in the analysis. 

+ Java Stack Frame（虚拟机栈）

  A Java stack frame, holding local variables. Only generated when the dump is parsed with the preference set to treat Java stack frames as objects. 

+ Unknown

  An object of unknown root type. Some dumps, such as IBM Portable Heap Dump files, do not have root information. For these dumps the MAT parser marks objects which are have no inbound references or are unreachable from any other root as roots of this type. This ensures that MAT retains all the objects in the dump.

##### Live or die?

还有一些对象，在内存充足的情况下允许他们存在，在内存不足时就可以回收，典型的比如说缓存。JDK也考虑到了这些用途，因此推出了集中不同强度的引用。

+ 强引用。只要引用存在，永远不会回收。

  ```java
  Object o = new Object();
  ```

+ 软引用。在GC时，会将这些对象列入回收范围，但放在第二次回收中，如果第一次回收还没有足够的内存，才会回收他们。注意最好直接不要有其他引用引到`new Object()`

  ```java
  SoftReference<Object> softReference = new SoftReference<>(new Object());
  // 下面的写法就不好
  Object o = new Object();
  SoftReference<Object> softReference = new SoftReference<>(o);
  o = null;	// 如果忘记，就有一个强引用指向该object
  ```

+ 弱引用。只能存活到下一次GC，下一次GC必然会回收。(有趣的是，还有一个WeakHashMap类)

  ```java
  WeakReference<Object> weakReference = new WeakReference<>(new Object());
  ```

+ 虚引用。这玩意对GC没有任何影响，也没法通过一个虚引用获得一个对象实例。为一个对象设置了虚引用后，该对象被回收时能收到一个系统通知而已。

##### 可达性分析的实际操作问题与safepoint

[找出栈上的指针/引用](https://rednaxelafx.iteye.com/blog/1044951)

在上面我们看到，可达性分析法需要首先去枚举gc roots，因为gc roots除了静态变量这种在类加载时已经确定的部分以外，还有一些像在线程栈中的引用这些不断发生变化的部分，因此，每一次要进行可达性分析，从理论上讲都需要先枚举gc roots

这其实是一项比较耗费时间的工程，而且必须要停止虚拟机其他线程的运行。因为如果允许在枚举的时候gc roots还在不断发生变化，那枚举的结果显然不准。这个暂停的动作，在hotspot虚拟机中称为Stop The World（STW）

那么有没有办法加速这个过程呢？有，首先，对于那些在类加载时期就已经能确定的引用，我们当然可以用一些数据结构把它记录下来，而对于运行时的一些数据，实际上也可以在某些位置把相关栈帧里的引用数据保留下来，gc roots的枚举速度就会提升。显然商用的虚拟机都这么做了（不然其实我也想不出来这些优化），保存这些gc roots位置的数据结构，称为OopMap（HotSpot的称呼）。

但这里就有个新问题，能引起栈帧变化的指令非常多，如果每一条指令都生成一个OopMap，那虚拟机的效率得低成什么样子。因此，JVM实际上只在特殊的指令处生成OopMap，这些指令处就称为“safePoint”，safepoint的选择，既要避免每一条指令来一下，也要避免太长时间到达不了safepoint，因此，在一些执行时间较长的任务前后进行保存似乎是个好主意，这样至少避免了长时间无法到达safepoint，这就包括：

+ 循环的末尾

+ 方法返回前

+ 调用方法的call指令之后

+ 可能抛出异常的位置

在这样的指令处，java可能会生成一个操作码，指令线程主动去更新自己的OopMap。在等待所有线程到达自己的safepoint时，JVM一般是设置一个GC的标志位，线程会轮询这个标志位，发现标志位被置位后，就自己等待，所有线程都等待后，就是Stop the word，标记GC roots开始。

##### 可达性分析的实际操作问题与JNI

上面又出现了一个新问题，那就是OopMap的维护，需要JVM的参与，实际上在java的字节码中加了一点料。但是当使用JNI的时候，那些本地方法的调用并不需要JVM的参与，他们怎么去维护这个OopMap呢？当然其实可以不维护，采取每次全扫一遍本地方法栈的方式来解决问题。

但是Hotspot虚拟机选择了另一种方式，即所有经过JNI调用边界的引用，都必须通过一个中间对象——"句柄"（handler），JNI要调用java API的时候也需要用句柄包装指针。而句柄表是可以由JVM维护的，这样需要知道JNI方法使用的java对象只需要扫描句柄表就可以了。

带来的代价是JNI调用需要进行句柄的中转，效率有所下降。

#### 几种不同的思想

##### 标记 - 清除

就是对所有已死亡的对象进行标记，然后把他们从内存中清除出去。标记的办法着实也不怎么高效，因为要标记已死亡对象，所以需要遍历堆内存的对象头，而清除出去主要是维护空闲列表，这两者消耗都比较大。

而且，这还很容易造成内存碎片，导致在内存明明还有很多空间时，大内存无法分配，而频繁引起GC，甚至Full GC。

##### 标记 - 整理

就是对所有还存活的对象进行标记，然后把他们移动到内存的一端去，之后将指示空闲内存开始位置的指针直接移动到最后一个存活对象尾部。这样就避免了内存碎片。

##### 复制算法

从概念上讲，是将内存分为相等大小的两个部分，当一个部分满了以后，就将这个部分的存活对象移动到另外一个部分去。这样最大的问题是内存使用率太低。但是根据前面的介绍，其实98%左右的对象都是朝生夕死，那么实际上真正需要复制的对象大多数情况下都非常少，所以，商用的虚拟机比如hotspot，在使用复制算法时并没有真的把空间分成相等的两个大小，而是分为eden、from survivor、to survivor三个区域，同时只使用eden和其中一个survivor，当两个都到了回收阈值的时候，就把存活的对象复制到另一个survivor中去，再把eden和from survivor清空。

比如说，在极端情况下，存活的对象非常多，to survivor放不下怎么办。这时候一般会直接把一些对象放到老年代去。因此，实际上在每一次GC开始前，都会查看一下老年代的空间，看老年代是不是足够放下极端情况下的所有新生代对象，如果足够，那么这次新生代GC可以无风险进行，万一to survivor放不下了，反正老年代够，就扔进去好了。那如果老年代如果空间不足以放得下新生代所有对象呢？这就涉及到一个新的问题，就是full gc和冒险策略。当允许冒险时，只要老年代的空间大于以往放入老年代的平均空间，那就进行新生代GC，万一失败，就进行full gc，如果不允许冒险或者老年代的空间小于以往放入老年代的平均空间，就直接进行full gc。

一般将新生代的gc成为Minor GC，而带老年代一起的GC称为Full gc。

##### 分代算法

因为新生代对象绝大多数都是朝生夕死，因此适合使用复制算法，但一些对象总是会逐渐稳定下来，那么这些对象来回复制的话代价就很高了，因此现在一般商用虚拟机都实现了分代算法，对新生代和老年代采取不同的回收策略，比如新生代用复制算法，老年代用标记-整理算法。

##### 分代算法引发的新问题与Remembered Set

‼️**以下内容有很多自己的理解，如果错漏概不负责，请抱着审慎的态度阅读。**

如果每一次都做Full GC，那么我们的思路就比较简单，就是从GC roots开始遍历探寻可达性即可，标记所有的可达数据，标记为存活。之后开始整理就完了。

但如果是分代处理的话，就会有一些新问题。比如从GC roots遍历是个比较耗时的操作，特别是假如我们只想回收新生代的垃圾，那么遍历老年代就是一个不必要的开销，特别是老年代堆很大，遍历花费的时间还不少。于是，商用虚拟机在新生代收集时，都不扫描老年代的对象，即，从GC roots开始遍历时，一旦遇到老年代的对象，就直接打住（判断是不是老年代对象其实比较简单，老年代的内存空间有上下界），不再继续深入。这样就只遍历了新生代对象。

但这又有一个问题。假如GC roots连接了一个老年代对象，但这个老年代对象持有了一个新生代对象的引用，但是上面说的方式就扫描不到，这就会把一个实际存活的新生代对象漏掉。为了解决这个问题，又引入了一个新概念，就是Remembered Set。

每当老年代对象引用新生代对象时，在新生代旁边开辟一块单独的空间，把这个引用关系记录下来，当引用断开时，清除这个引用关系。于是，GC收集器就可以在不完整扫描老年代对象的情况下，直接去读取这些记录就知道新生代的哪些对象被老年代引用了。换言之，GC收集器将这个记录也作为GC roots就可以了。

这个记录就叫做Remembered Set。在一些文章中我们会看到Card Table这个概念，这是Remebered Set的一种具体实现。

可见，safe point和Remembered Set技术的主要思想都是用空间换时间，尽量减少GC时要扫描的范围。

### 垃圾收集器的实现

垃圾收集器有很多种不同的实现，没有一种垃圾收集器适应所有的应用场景，要合理选择垃圾收集器，就要首先了解关于垃圾收集器和GC的一些基本指标。

#### 两个需要注意的指标

除了垃圾回收要确实干净、安全以外。有两个有关时间的指标需要考量。

+ 吞吐量。

  即JVM运行后，用于执行用户代码的时间和CPU消耗的总时间的比例，这个比例越高，说明CPU更多地用于用户代码执行，比如说JVM运行了100分钟，垃圾回收花掉了1分钟，那么吞吐量就是99%。

+ 线程停顿时间

  因为在枚举根节点等情况下，需要STW，因此用户线程会被停顿，这个停顿时间，当然我们希望是越短越好，这样就能尽量不影响用户操作了。

不同的垃圾收集器侧重点不同，不同的应用的需求也不同。比如说，CPU运算密集型的应用（这种应用为啥要用java写），显然更关注吞吐量，希望有更多时间花在用户工作上。而一个服务器，更希望用户线程被停顿的时间越短越好，尽量不要影响用户操作比较好。

因此，不同的应用下选择合理的垃圾收集器，也是程序员的一种能力。

#### 两种需要理解的GC类别

根据 `JVM界唯一认证扛把子·他说的不一样的都是错的·真·R大` 的文章：[Major GC和Full GC的区别是什么？](https://www.zhihu.com/question/41922036/answer/93079526)

> 针对HotSpot VM的实现，它里面的GC其实准确分类只有两大种：
>
> Major GC通常是跟full GC是等价的，收集整个GC堆。但因为HotSpot VM发展了这么多年，外界对各种名词的解读已经完全混乱了，当有人说“major GC”的时候一定要问清楚他想要指的是上面的full GC还是old GC。
>
> 最简单的分代式GC策略，按HotSpot VM的serial GC的实现来看，触发条件是：
>
> HotSpot VM里其它非并发GC的触发条件复杂一些，不过大致的原理与上面说的其实一样。
> 当然也总有例外。Parallel Scavenge（-XX:+UseParallelGC）框架下，默认是在要触发full GC前先执行一次young GC，并且两次GC之间能让应用程序稍微运行一小下，以期降低full GC的暂停时间（因为young GC会尽量清理了young gen的死对象，减少了full GC的工作量）。控制这个行为的VM参数是-XX:+ScavengeBeforeFullGC。这是HotSpot VM里的奇葩嗯。可跳传送门围观：[JVM full GC的奇怪现象，求解惑？ - RednaxelaFX 的回答](https://www.zhihu.com/question/48780091/answer/113063216)
>
> 并发GC的触发条件就不太一样。以CMS GC为例，它主要是定时去检查old gen的使用量，当使用量超过了触发比例就会启动一次CMS GC，对old gen做并发收集。

#### 具体的垃圾收集器

首先，这些垃圾收集器包括：

![hotspot垃圾收集器](https://img-blog.csdn.net/20160505170035450)

红色区域表示新生代垃圾收集器，而绿色区域表示老年代垃圾收集器。其中有连线表示可以配合使用。

##### Serial

就是单线程，Stop The World后实现垃圾收集。虽然说STW不太友好，单线的整体程效率也不高。但他因为简单，所有具有最高效率的单线程回收效率。所以至今仍然广泛用在java的Client模式（适用于GUI的开发）下，这个场景一般内存占用不大，收集几十兆上百兆的新生代还是没什么明显的停顿感。

+ 插播：Jvm的Client与Server模式

  ```
  > ~ java -version
  java version "1.8.0_131"
  Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
  Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)
  ```

  在机器上运行`jave -version`就能看到模式，在最后一行。一般来说，Client模式启动更快，但启动后运行时和垃圾回收表现都稍差；Server编译更彻底，在启动时稍慢，但运行后速度和垃圾回收效果都比较好。一般可以简单认为，Client适合开发Gui，而Server适合做服务器。

与Serial收集器相关的虚拟机参数包括：

+ -XX:SurvivorRatio=8，意思是eden/survivor区的比例，默认为8，上面都说了无数次了。
+ -XX:MaxTenuringThreshold  这是与垃圾回收器紧密相关的一个参数，意思是当一个新生代的对象经过一次gc之后如果存活下来了，那么他的年龄会+1，当达到这个参数指定的年龄后，就会被移动到老年代。对象从survivor移动到老年代还有一个可能，就是当survivor区中同年龄的对象达到或超过一半survivor空间大小，那么大于等于此年龄的对象都会被移动到老年区。该参数只有用串行GC回收器才有效。
+ -XX:PretenureSizeThreshold=\<byte size> 意思是当对象大于这个值时直接在老年代分配，以避免大对象的来回复制。但有利有弊吧，因为老年代不经常被回收，所以如果短命大对象直接在老年代分配，会导致不断需要触发full gc，得不偿失。这个参数默认是0，也就是所有对象都在eden分配。
+ -XX:HandlePromotionFailure 就是上面所说的复制算法的老年代担保是否允许冒险。但在JDK1.6以后，这个参数已经无效，JVM总是会选择冒险策略。

触发时机：

eden区不够分配对象了。

##### ParNew

Serial收集器的多线程版本，复用了Serial的大多数代码，和相关虚拟机参数。

在单CPU环境下，其变现绝对不可能超过Serial，甚至双CPU也未必能超过，但多CPU下性能表现一般会更好。而且因为只有它能与CMS配合使用，因此在server模式下经常使用。

触发时机：

eden区不够分配对象了

##### Parallel Svavenge

这也是个新生代的收集器，使用复制算法，并且多线程。看起来和parNew是一样的。但是他的关注点和ParNew不一样，这个垃圾收集器更加关注吞吐量。因此，他提供了两个参数来设置吞吐量和最大垃圾收集停顿时间。

+ -XX:MaxGCPauseMills=100 意思是**请求**垃圾收集器每次垃圾回收时暂停的时间尽可能不超过这个值。但只是尽可能，不能完全保证，而且每次暂停时间短有可能造成GC的次数增多，造成吞吐量下降。
+ -XX:GCTimeRatio=99 就是CPU执行用户任务的时间/GC时间的值，数值越大，允许的GC时间就越短。比如99，就是说只允许1%的时间用于垃圾收集。但这个也只能是建议，不可能完全保证。
+ -XX:UseAdaptiveSizePolicy 当这个参数打开以后，就不用手动指定新生代大小、Eden和Survivor区的比例，晋升老年代的年龄值等细节参数了。虚拟机会动态监控系统运行状况，得到一个合适的值，并设置前两个参数，给虚拟机一个目标，虚拟机就会自动调整以上参数。

触发时机：

eden区不够分配了

##### Serial Old

Serial的老年代版本，主要意义除了和Serial一起用在client模式下以外，还有作为CMS收集器的后备方案。

触发时机：

1. 在Client模式下使用时，触发时机是老年代空间不足
2. 作为CMS的后备方案时，触发时机是：当CMS收集器在并发清理环节浮动垃圾撑爆了剩余的老年代空间，就会抛出ConcurrentModeFailure。从而触发Serial Old收集。

##### Parallel Old

是Parallel Svavenge收集器的老年代版本，和新生代的Parallel Svavenge收集器配合，能达到合理控制吞吐量的效果。

触发时机：

老年代空间不足。

##### CMS（Concurrent Mark Swap）

[简书：图解CMS垃圾回收机制，你值得拥有](https://www.jianshu.com/p/2a1b2f17d3e4)

从名字可以看出，这个收集器是基于标记-清除算法实现的，而且多线程。

这个收集器的不同之处，在于它以获取最短停顿时间为目标，并且实现了垃圾回收线程（的一些工作阶段）与用户的任务线程并发执行。

为了实现这个目标，它分为四个阶段：

+ 初始标记。即只标记与GC roots直接关联的对象，需要STW
+ 并发标记。从初始标记中的结果进行可达性分析。不需要STW，可以和用户线程并发执行。
+ 重新标记。这个过程也需要STW，这是为了修正在STW期间因为用户操作而变动的那一部分标记，速度比初始标记慢一点，但是比并发标记阶段要快得多。
+ 并发清除。把标记出来的垃圾清除掉。这个也可以和用户线程并发。

因为并发标记和并发清除阶段和用户线程并发了，因此暂停时间比较短。但是这个收集器也有几个缺点：

+ 它的单线程收集效率显然不如之前介绍的那些收集器。而且，因为在并发过程中，它也占用了CPU，因此在单CPU或者双CPU情况下，会导致有很大的CPU资源被用作垃圾回收，用户的代码能使用的CPU资源就会受到很大的挤占。

+ 在并发清除阶段，因为与用户线程并发，因此还会生成新的垃圾，为了应对这些新建的对象，CMS收集器必须预留足够的空间给用户线程使用，因此不能像其他收集器一样等到老年代几乎填满了再进行收集，在JDK1.5的默认情况下，当老年代使用到68%，就会激活CMS收集器，后来提升到92%。如果CMS运行期间预留的空间不足以供用户线程新申请的对象使用，那么JVM就会启动Serial Old重新处理老年代的垃圾收集，这会非常慢。

  与这个行为相关的参数为：

  + -XX:CMSInitiatingOccupancyFranction。即设置上面的百分比。设置的值如果太高，就会经常导致并发收集失败，启用Serial Old，性能会下滑明显。

+ CMS基于并发-清除算法，意味着内存碎片会增多。因此可能对大对象分配带来麻烦。因此，在进行Full GC时，就进行内存的整理，内存整理无法并发，因此必须STW，停顿时间会变长。

而且仔细考虑CMS的并发清除阶段，实际上还有这样一个小问题：即并发清除阶段如果发生了young GC，有新对象进入了老年区，他们是没有被标记的白对象，会不会被清除？就是[这个问题](https://stackoverflow.com/questions/43789967/java-cms-gc-can-a-minor-gc-occur-between-final-remark-and-concurrent-sweep)所担心的情况。这个问题的解答是，因为CMS对老年区维护了可用空间的列表，因此只要保证新进入老年区的对象放到这些可用空间里，而并发清除不扫描这些可用内存区域即可（以上有自己的猜测在里面，不一定准确）。即：并发清除阶段扫描的时候只扫描在最终标记后标记为需要清理的空间。这也是为什么CMS不能等到老年区几乎填满才进行垃圾回收的原因。（深入理解JAVA虚拟机第二版P83）说的也就是这个问题。

与CMS相关的参数非常多，简单列几个如下：

+ -XX:+UseConcMarkSweepGC  开启CMS

+ -XX:UseCMSCompactAtFullCollection 默认开启，用于FullGC时的内存整理。
+ -XX:CMSFullGCSBeforeCompaction=1 几次不带整理的FullGC后来一次整理的，默认值为0，意味着每次进入FullGC都进行碎片整理。
+ -XX:ParallelCMSThreads=20 CMS默认启动的回收线程数目是 (ParallelGCThreads + 3)/4) ，如果你需要明确设定，可以通过-XX:ParallelCMSThreads=20来设定,其中ParallelGCThreads是年轻代的并行收集线程数
+ -XX:+UseCMSInitiatingOccupancyOnly  
+ -XX:CMSInitiatingOccupancyFraction=92.0 老年代空间占用达到多少时进行一次老年代垃圾回收。

触发时机：

1. 如果没有设置-XX:+UseCMSInitiatingOccupancyOnly，虚拟机会根据收集的数据决定是否触发。
2. 老年代使用率达到阈值 `CMSInitiatingOccupancyFraction`，默认92%。
3. 新生代的晋升担保失败。

#### 上述几种收集器的可能组合

摘自[知乎：Major GC和Full GC的区别是什么](https://www.zhihu.com/question/41922036/answer/144566789)

1. Serial GC算法：Serial Young GC ＋ Serial Old GC （实际上它是全局范围的Full GC）
2. Parallel GC算法：Parallel Young GC ＋ 非并行的PS MarkSweep GC / 并行的Parallel Old GC（这俩实际上也是全局范围的Full GC），选PS MarkSweep GC 还是 Parallel Old GC 由参数UseParallelOldGC来控制；
3. CMS算法：ParNew（Young）GC + CMS（Old）GC （piggyback on ParNew的结果／老生代存活下来的object只做记录，不做compaction）＋ Full GC for CMS算法（应对核心的CMS GC在老年代给浮动垃圾预留的内存不足导致并发清除失败后full gc，开销很大）；

4. 下面即将介绍的，新生代和老年代一起搞了的，G1收集器。

#### G1（Garbage-First）垃圾优先收集器

[Garbage First Collector 理解](https://ylgrgyq.github.io/2016/07/03/garbage-first-collector-understanding/)

G1是一款比较新的垃圾收集器。在JDK8时，需要加参数`-XX:+UseG1GC`，在JDK11时，默认使用这款垃圾收集器。这款垃圾收集器与之前介绍的一些收集器相比，有以下特点：

1. 能够同时处理新生代和老年代。虽然在内部也区分了不同代的不同处理方式。这也不算最大的特点，因为之前的比如说Serial收集器，就可以同时收集新生代和老年代。

2. 对内存逻辑结构的改变很大。不像之前的收集器处理的新生代和老年代是逻辑上连续的一整块内存，启用了G1垃圾收集器的JVM，从顶层来看，虽然仍然区分了新生代和老年代，也有eden、survivor和old区，但是在具体的分配上不再是各自连续的一整块，而是划分为一个一个的Region，每一个Region的大小是固定的，最小是1M，可以以2的倍数递增，最大是32M。同时，新增了一个Humongous区，用来存储大对象，所谓的大对象，是指超过region大小一半的对象。这么做是为了防止对大对象进行反复拷贝，其堆内存结构如下图：

   ![G1HeapLayOut](https://www.history-of-my-life.com/imgs-for-md/JVM-G1-20190809.png)

3. 从顶层来看，G1收集器是基于`标记-清除`的算法，因为它更像是发现哪些Region没有存活对象了，就把这些Region的空间标记为可用。但就细节而观察，又是基于复制算法，即将一些对象从这个Region复制到另外一个Region里面去，从而把其中一些Region空出来。

4. 创造性地建立了可预测停顿时间的机制，可以指定在M毫秒的时间片段内，消耗在垃圾回收上的时间不得超过N毫秒。（当然也不一定会满足，只是尽量满足）。这种机制的实现，是依赖它维护了每个Region回收后能清理出的空间的大小（垃圾回收价值）的优先列表，在每次垃圾回收时未必都回收，而是根据允许的垃圾回收时间，优先回收价值大的Region。这就是所谓垃圾优先（Gabage First）名称的由来。

在实际收集过程中，G1也分为四个阶段：

1. 初始标记

   标记GC roots中直接关联的对象，需要STW。在这个阶段完成以后，之后的用户线程新建的对象会被放到一些确定这一次不会回收的Region里面去。

2. 并发标记

   进行可达性分析，找出存活的对象。因为G1甚至不是按照新生代老年代来回收垃圾，而是按照回收的价值来回收Region。因此上面所说的分代带来的老年代（不想浪费时间扫描的对象）引用新生代的问题在这里也会暴露出来。即我们不想回收的Region里（价值低的Region），存在指向我们想回收的Region里的对象。因此，G1为每一个Region都维护了一个Remembered Set。

   这一阶段和用户的线程并发。这就导致了对象引用图有可能会发生变化。

3. 最终标记

   对并发过程中用户程序修改的对象图发生的变化情况进行修正，需要STW。

4. 筛选回收

   根据用户期望时间，查询Region垃圾回收价值优先队列，确定最终要回收哪些Region（筛选），并回收。

G1垃圾收集器看起来和CMS垃圾收集器很像，在一些缺点上也比较像，就是肯定会有一些浮动垃圾，而且在CPU数量较少时也会有一些性能挤占问题。

相关参数：

+ -XX:G1HeapRegionSize  设置Region大小，取值范围从1m到32m，必须以2的倍数递增。如果不指定，将会根据Heap的大小自动设置，以期达到2048块region。
+ -XX:MaxGCPauseMillis=n  每次GC的期待最大暂停时间，以ms为单位。
+ -XX:G1PrintRegionLivenessInfo debug参数，默认false。在清理阶段的并发标记环节,输出堆中的所有 regions 的活跃度信息
+ -XX:G1PrintHeapRegions	debug参数，默认false，G1将输出哪些regions被分配和回收的信息
+ -XX:G1RSetUpdatingPauseTimePercent  G1收集器在GC时，更新Rset的目标时间值。

G1的特点与优势

+ 适合大堆，因为不像 CMS 和 Parallel GC 在对老代进行收集的时候需要将整个老代全部收集，G1 收集老代一次只收集老代的一部分 Region
+ G1 的 Heap 划分为多个 Region，Young Generation 和 Old Generation 都只是逻辑概念，不是物理上隔离又连续的空间
+ G1 的新老代划分不是固定的，一个新代的 Region 在被回收之后可以作为老代 Region 使用，Young Generation 和 Old Generation 大小也会随着系统运行而调整
+ G1 的新生代收集和 Parallel、CMS GC 一样是并发的 STW 收集，且每次 YGC 会将整个 Young Generation 收集
+ G1 的 Old Generation 收集每次只收集一部分 Old Region，且这部分 Old Region 是和 YGC 一起进行的，所以称为 Mixed GC
+ 和 CMS 一样，G1 也有 fail-safe 的 FGC，单线程且会做 compaction
+ G1 的 Old Generation GC (Mixed GC) 也是自带 compaction 的
+ G1 没有永久代的概念

触发时机：

因为G1可以同时支持新生代和老年代的收集，所以它的触发时机更复杂一些。

对年轻代来说，触发时机就是Eden区被沾满。

对老年代来说，和CMS类似，当Old Generation空间占用**整个Heap**比例超过目标值(-XX:InitiatingHeapOccupancyPercent, IHOP)后触发。

#### CMS和G1的最终标记阶段都做了些什么

[知乎:关于CMS、G1垃圾回收器的重新标记、最终标记疑惑](https://www.zhihu.com/question/37028283)

因为CMS和G1都有并发标记阶段，也就是说，因为允许可达性分析阶段和用户线程并行。那么就必然带来一个问题，在可达性分析时，一些引用关系已经发生了变化。这些变化主要包含这两种情况：

1. 新增了部分GC roots，这些GC roots引用了一些新建的对象。
2. 一些已有的引用关系发生了变化。比如将一个引用赋值为null，或者将一个引用引用到另外一个对象上去。其实引用到另外一个对象上，也无外乎三种情况，第一种，是使一个引用指向null，这有可能造成一个对象成为垃圾，我们称这个操作为引用删除；第二种，是使一个引用指向另外一个已存在的对象，这其实对垃圾回收没有任何印象；第三种，是新建一个对象，让这个引用指向新建的对象，这等价于发生了两步操作，一步是使引用指向null，第二步是相当于GC roots引用了新建对象。

综合起来看，总共就两类操作：

1. 新建引用-对象关联，有引用指向新建对象；
2. 删除引用-对象关联，有对象在并发标记期间不再有引用指向了。

而最终标记阶段就是为了处理这些问题，CMS和G1都使用了一种叫做Write Barrier的技术，来对初始标记以后发生的变化进行记录。在这个阶段，每一次引用变化都会经过Write Barrier，记录相关的变动，最后将这些记录合并到清理对象里面去就可以了。

回忆一下，CMS的并发清除阶段因为是将死亡对象的空间直接标记为可用，因此一定不能出现漏标的情况，如果漏标了存活对象，就会导致程序出现错误。（错标虽然会造成垃圾增多，但是下次收集就完了，不会造成程序错误）。因此，CMS的最终标记阶段关注的问题是：新建引用对象必须要记录下来，删除引用倒无所谓，大不了等下一次收集。

所以，CMS采用了叫做INC write barrier的技术，就是记录每次引用新建，在最终标记阶段还需要重新扫描一遍GC roots，因此这个remark时间还比较长，一般占到一次CMS两次STW总时间的80%。

而G1就略有不同，G1使用了 Snapshot-At-The-Beginning Write Barrier(SATB)，在初始标记阶段结束后会打上一个快照，之后所有的新分配的对象通通视为活跃，然后分配到这次保证不回收的Region里面去，反正是按Region回收嘛。它更关注引用的删除，以更加准确地评估垃圾回收的价值。

追根究底，似乎CMS和G1都不非要有重新标记阶段，因为只要保证新进入老年区/新分配的内存分配在此次不回收的内存区就完了，代价是更多的浮动垃圾。但实际上还是做了重新标记，可能是JVM的开发团队评估后认为这样性能还是更好。**（这一段包含自己的猜测，请审慎阅读）**

## JVM调试工具

### 概述

#### 工具箱里有什么

JVM调试工具做的事情，就是监控虚拟机的性能，并且发现发生在运行时的故障比如说死锁、内存溢出等。大部分JVM工具以命令行工具的方式提供，也有一些可视化工具，比如JConsole和VisualVM。

其中，命令行工具有：

+ jps

  查看主机运行的JVM的ID（和操作系统的PID相同）以及运行参数

+ jstat

  查看虚拟机的各项统计数据。主要是一些内存占用状况，GC的情况。

+ jinfo

  查看虚拟机的各项配置信息，以及环境变量。

+ jmap

  Java内存映像工具，即将JVM的一块内存以二进制的形式dump成为文件，方便用其他内存分析工具进行分析。还有个骚操作，直接将目前JVM的状态完整保存成一个可运行的jar，可以直接从当前状态运行。

+ jhat

  用来读取jmap dump出来的内存文件，或者设置了`-XX:+HeapDumpOnOutOfMemoryError`的虚拟机在内存溢出时自动dump出来的内存文件。他会启动一个web server，用户可以用浏览器查看分析结果。

+ jstack

  java堆栈跟踪工具。能够将JVM线程栈的运行情况打印出来，包括线程基本信息如tid等，还包括其所处的状态比如Waiting、Runable等，以及其持有的锁信息等。在追踪线上服务某些线程长时间卡顿时非常好用。

+ jcmd

  一个多功能的工具，其功能覆盖了上面介绍的好几个工具，比如它也能导出堆、查看Java进程、导出线程信息、执行GC、还可以进行采样分析。自JDK1.7引入。

可视化工具包括：

+ Jconsole

  在命令行运行会启动一个GUI界面，显示一些统计数据。

+ VisualVM

  一个支持插件的可视化JVM性能检测工具，是随JDK发布的功能最强大的debug工具，并且官方称之为`All-In-One`，可见官方对其寄予厚望。

#### 获得heap Dump的几种方式

上面可以看到，很多调试工具都是围绕JVM内存管理展开的，也确实，OOM是我们经常遇到又让人非常恼火的错误。因此我们先介绍一下获得heap dump的几种方式。

+ 使用jmap

  `jmap -dump:live,format=b,file=d:\dump\heap.hprof <pid>`

+ 使用jcmd

  `jcmd <pid> GC.heap_dump d:\dump\heap.hprof`

+ 使用JVM参数获取dump文件

  + -XX:+HeapDumpOnOutOfMemoryError

    OOM时导出内存镜像。可能是比较常用的debug参数了，因为毕竟一般不OOM，我们也不太关注内存使用情况。

  + -XX:+HeapDumpBeforeFullGC

    每次Full GC之前导出内存镜像。在性能调优的情况下可能用的比较多。

  + -XX:+HeapDumpAfterFullGC

    当 JVM 执行 FullGC 后执行 dump。

  + -XX:+HeapDumpOnCtrlBreak

    交互式的操作，当按下ctrl+break时产生镜像。可能在debug时经常会用到吧。

  + -XX:HeapDumpPath=d:\test.hprof

    指定内存dump文件的保存位置。

### 详解

注：因为不同JDK版本的问题，以下命令的细节可能有些变化，每个命令都可以用--help参数查看帮助，具体的表现请以自己的版本为准。以下的输出基于`JDK 1.8_131`， `> ~ `表示命令行的输入。因为在macOS下，1.8更高版本的调试工具无法正常使用，会抛以下异常，查了一些资料好像是官方bug，暂时也没办法，降到131后正常。

```
Attaching to process ID 1014, please wait...
Error attaching to process: sun.jvm.hotspot.debugger.DebuggerException: Can't attach symbolicator to the process
sun.jvm.hotspot.debugger.DebuggerException: sun.jvm.hotspot.debugger.DebuggerException: Can't attach symbolicator to the process
	at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal$BsdDebuggerLocalWorkerThread.execute(BsdDebuggerLocal.java:169)
	at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal.attach(BsdDebuggerLocal.java:287)
	at sun.jvm.hotspot.HotSpotAgent.attachDebugger(HotSpotAgent.java:671)
	at sun.jvm.hotspot.HotSpotAgent.setupDebuggerDarwin(HotSpotAgent.java:659)
	at sun.jvm.hotspot.HotSpotAgent.setupDebugger(HotSpotAgent.java:341)
	at sun.jvm.hotspot.HotSpotAgent.go(HotSpotAgent.java:304)
	at sun.jvm.hotspot.HotSpotAgent.attach(HotSpotAgent.java:140)
	at sun.jvm.hotspot.tools.Tool.start(Tool.java:185)
	at sun.jvm.hotspot.tools.Tool.execute(Tool.java:118)
	at sun.jvm.hotspot.tools.JInfo.main(JInfo.java:138)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at sun.tools.jinfo.JInfo.runTool(JInfo.java:108)
	at sun.tools.jinfo.JInfo.main(JInfo.java:76)
Caused by: sun.jvm.hotspot.debugger.DebuggerException: Can't attach symbolicator to the process
	at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal.attach0(Native Method)
	at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal.access$100(BsdDebuggerLocal.java:65)
	at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal$1AttachTask.doit(BsdDebuggerLocal.java:278)
	at sun.jvm.hotspot.debugger.bsd.BsdDebuggerLocal$BsdDebuggerLocalWorkerThread.run(BsdDebuggerLocal.java:144)
```

#### jps

```
> ~ jps --help
usage: jps [--help]
       jps [-q] [-mlvV] [<hostid>]
```

具体效果请自测。-l参数输出主类的全限定名；-m输出main method的参数；-v输出jvm参数；输出通过flag文件传递到JVM中的参数(.hotspotrc文件或-XX:Flags=所指定的文件；

#### jinfo

[简书：jinfo命令详解](https://www.jianshu.com/p/c321d0808a1b)

```
> ~ jinfo --help
Usage:
    jinfo [option] <pid>
        (to connect to running process)
    jinfo [option] <executable <core>
        (to connect to a core file)
    jinfo [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    -flag <name>         to print the value of the named VM flag
    -flag [+|-]<name>    to enable or disable the named VM flag
    -flag <name>=<value> to set the named VM flag to the given value
    -flags               to print VM flags
    -sysprops            to print Java system properties
    <no option>          to print both of the above
    -h | -help           to print this help message
```

这个命令可以查看和修改JVM的一些参数。具体效果自己测试吧，会打印出来一大堆。

#### jstat

这个主要是输出一些内存相关的统计数据。有很多子参数。

```
> ~ jstat --help
Usage: jstat --help|-options
       jstat -<option> [-t] [-h<lines>] <vmid> [<interval> [<count>]]

Definitions:
  <option>      An option reported by the -options option
  <vmid>        Virtual Machine Identifier. A vmid takes the following form:
                     <lvmid>[@<hostname>[:<port>]]
                Where <lvmid> is the local vm identifier for the target
                Java virtual machine, typically a process id; <hostname> is
                the name of the host running the target Java virtual machine;
                and <port> is the port number for the rmiregistry on the
                target host. See the jvmstat documentation for a more complete
                description of the Virtual Machine Identifier.
  <lines>       Number of samples between header lines.
  <interval>    Sampling interval. The following forms are allowed:
                    <n>["ms"|"s"]
                Where <n> is an integer and the suffix specifies the units as
                milliseconds("ms") or seconds("s"). The default units are "ms".
  <count>       Number of samples to take before terminating.
  -J<flag>      Pass <flag> directly to the runtime system.
  -? -h --help  Prints this help message.
  -help         Prints this help message.
```

##### -class

显示加载class的数量，及所占空间等信息，效果如下：

```
> ~ jstat -class 1758
Loaded       Bytes    Unloaded     Bytes      Time
 15704     28719.1           1       0.9     10.09
```

##### -compiler

显示VM实时编译的数量等信息。

```
> ~ jstat -compiler 1758
Compiled Failed Invalid    Time   FailedType   FailedMethod
    7957      0       0    29.31           0
```

##### -gc

显示gc的消息，包括gc的次数、时间等。

```
> ~ jstat -gc 1758
S0C       S1C    S0U      S1U       EC       EU       OC         OU       MC         MU       CCSC       CCSU    YGC     YGCT    FGC    FGCT    CGC       CGCT      GCT
0.0   14336.0    0.0   14336.0 139264.0 129024.0  89088.0    21987.5   80000.0   78249.0   10880.0    10193.5     14    0.170      0   0.000      8      0.030    0.199
```

+ S0C:  survivor0 capacity（单位kb）
+ S0U: survivor0 used
+ S1C:  survivor1 capacity
+ S1U: survivor1 used
+ EC:  Eden capacity
+ EU: Eden used
+ OC: old capacity
+ OU: old used
+ MC: method capacity
+ MU: method used
+ CCSC: 压缩类空间大小
+ CCSU: 压缩类空间使用大小
+ YGC: 年轻带垃圾回收次数
+ YGCT: 年轻代垃圾回收耗时
+ FGC: Full GC 次数
+ FGCT: Full GC耗时
+ CGC: Concurrent GC次数
+ CGCT: Concurrent GC耗时
+ GCT: GC总耗时

##### -gccapacity

堆内存统计

```
> ~ jstat -gccapacity 57889
NGCMN       NGCMX       NGC     S0C      S1C         EC   OGCMN       OGCMX        OGC         OC       MCMN        MCMX       MC     CCSMN      CCSMX      CCSC     YGC    FGC   CGC
  0.0   2097152.0   149504.0    0.0   12288.0  137216.0     0.0    2097152.0    88064.0    88064.0       0.0   1118208.0   80000.0      0.0   1048576.0  10880.0     15      0     8
```

部分和-gc重复的输出就不解释了，看一下不同的输出：

+ NGCMN		新生代最小容量（单位kb）
+ NGCMX         新生代最大容量
+ NGC              当前新生代容量
+ OGCMN        老年代最小容量
+ OGCMX         老年代最大容量
+ MCMN           方法区最小容量
+ MCMX            方法区最大容量
+ CCSMN          压缩区最小容量
+ CCSMX           压缩去最大容量

##### -gccause

最近一次GC统计和原因

```
> ~ jstat -gccause 57889
  S0       S1        E       O      M       CCS    YGC     YGCT    FGC    FGCT    CGC    CGCT     GCT      LGCC               GCC
0.00   100.00    52.99   27.49   97.66    93.52     15    0.167     0    0.000     8    0.022    0.189 G1 Evacuation Pause  No GC
```

其中的S0什么的，都是指使用了多少。

##### -gcmetacapacity

metaSpace中对象的信息和占用量

```
> ~ jstat -gcmetacapacity 57889
MCMN       MCMX        MC       CCSMN      CCSMX       CCSC     YGC   FGC    FGCT    CGC    CGCT     GCT
  0.0  1118208.0    80000.0        0.0  1048576.0    10880.0    15     0    0.000     8    0.022    0.189
```

##### -gcnew

年轻代对象的信息

```
> ~ jstat -gcnew 57889
S0C    S1C    S0U     S1U    TT  MTT     DSS        EC       EU    YGC   YGCT
0.0 12288.0   0.0  12288.0    5   15   8704.0  137216.0  72704.0    15  0.167
```

+ TT       对象在新生代存活的次数（年龄）
+ MTT    对象在新生代存活的最大次数

+ DSS     幸存者区所需空间大小

##### -gcnewcapacity

年轻代容量信息，如下：

```
> ~ jstat -gcnewcapacity 57889
NGCMN      NGCMX       NGC      S0CMX     S0C     S1CMX     S1C       ECMX        EC      YGC   FGC   CGC
  0.0  2097152.0   149504.0      0.0      0.0 2097152.0  12288.0  2097152.0   137216.0    15     0     8
```

##### -gcold

老年代对象信息，如下：

```
> ~ jstat -gcold 57889
     MC       MU      CCSC     CCSU       OC          OU        YGC    FGC    FGCT    CGC    CGCT     GCT
80000.0  78128.8    10880.0  10174.9     88064.0     24207.3     15     0    0.000     8    0.022    0.189
```

##### -gcoldcapacity

```
> ~ jstat -gcoldcapacity 57889
OGCMN       OGCMX        OGC         OC       YGC   FGC    FGCT    CGC    CGCT     GCT
  0.0   2097152.0     88064.0     88064.0    15     0     0.000     8    0.022    0.189
```

##### -gcutil

```
> ~ jstat -gcutil 57889
  S0       S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT    CGC    CGCT     GCT
0.00   100.00  55.97  27.49  97.66  93.52     15    0.167     0    0.000     8    0.022    0.189
```

输出了数据后怎么分析，还是看自己需求。比如说我看了我们一个线上JVM的服务，YGC总共一百多次，FGC竟然运行了一千多次，显然是哪里出了问题，内存分配策略有问题。

#### jstack

这个命令用于打印指定java进程或者核心文件中所有java线程当前时刻正在执行的方法堆栈追踪情况，也就是线程的snapshot，生成线程的快照主要用于定位线程长时间出现停顿的原因，如死锁、等待外部资源等。jstack在分析死锁，阻塞等性能问题上非常有用，根据打印的堆栈信息可以定位到出问题的代码段。定位问题的思路根据要解决的问题而发生不同，比如可以首先找到java进程中最耗cpu的线程，再根据线程id在jstack的输出中定位，或者使用指定的线程名称定位。

输出特别有意思，比如说：

```
> ~ jstack -l 57889
2019-08-08 17:20:10
Full thread dump Java HotSpot(TM) 64-Bit Server VM (11.0.4+10-LTS mixed mode):

Threads class SMR info:
_java_thread_list=0x00007fc141522820, length=31, elements={
0x00007fc14288d800, 0x00007fc14202c000, 0x00007fc142028800, 0x00007fc142894000,
0x00007fc142897000, 0x00007fc143020800, 0x00007fc141821000, 0x00007fc142903000,
0x00007fc1462d9000, 0x00007fc1463a5800, 0x00007fc144071000, 0x00007fc1450e7800,
0x00007fc144ef4800, 0x00007fc141d54800, 0x00007fc14379a800, 0x00007fc143551800,
0x00007fc145134800, 0x00007fc141fd4000, 0x00007fc141deb000, 0x00007fc144270000,
0x00007fc144164800, 0x00007fc141bde000, 0x00007fc144156000, 0x00007fc141c6f800,
0x00007fc146422800, 0x00007fc14657d800, 0x00007fc1440ca000, 0x00007fc14453e000,
0x00007fc1450e6800, 0x00007fc145f02000, 0x00007fc14552d800
}

"Reference Handler" #2 daemon prio=10 os_prio=31 cpu=6.75ms elapsed=6997.49s tid=0x00007fc14288d800 nid=0x3303 waiting on condition  [0x000070000dfc6000]
   java.lang.Thread.State: RUNNABLE
	at java.lang.ref.Reference.waitForReferencePendingList(java.base@11.0.4/Native Method)
	at java.lang.ref.Reference.processPendingReferences(java.base@11.0.4/Reference.java:241)
	at java.lang.ref.Reference$ReferenceHandler.run(java.base@11.0.4/Reference.java:213)

   Locked ownable synchronizers:
	- None
	
... // 以下省略
```

之后的输出就类似于上面最后一段的输出，解读如下：

+ "Reference Handler" :     线程名称

+ daemon:	                       守护线程 

+ prio:	                               线程优先级

+ os_prio:	                        系统的线程优先级
+ cpu:                                   占用的CPU时间
+ elapsed:                           启动多久了
+ tid:                                    jvm中的线程标识符
+ nid:                                   系统中的本地线程标识符
+ waiting on condition:     等待条件满足
+ [0x000070000dfc6000]  线程的起始地址

下面显示的是函数调用栈和持有锁的情况。

比如说下面这段代码，就一定会造成线程Waiting卡死，注：**这是错误的写法，代码里不要用。**

```java
public static void main(String[] args) {

    ExecutorService executorService = Executors.newFixedThreadPool(5);
    for (int i = 0; i != 5; ++i) {
        executorService.execute(() -> 
            System.out.println("hello, I'm " + Thread.currentThread().getName()));
    }
  
    //没有这一行的话下面的awaitTermination会等超时，详情见awaitTermination的注释
  	// executorService.shutdown();   
    try {
        executorService.awaitTermination(10, TimeUnit.DAYS);
    } catch (InterruptedException e) {
        //
    }

}
```

之后进程就hang住了。jstack查看一下

```
> ~ jstack 9527
Full thread dump Java HotSpot(TM) 64-Bit Server VM (25.191-b12 mixed mode):

"Attach Listener" #17 daemon prio=9 os_prio=31 tid=0x00007fe5e8a0b000 nid=0x2907 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

"pool-1-thread-5" #16 prio=5 os_prio=31 tid=0x00007fe5e92ff800 nid=0x5703 waiting on condition [0x000070000887a000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x0000000795c2d4b0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
	
// 之后省略
```

可以看到，线程池里面的线程处于等待之中，并且是在ThreadPoolExecutor中等待，等待的原因是从LinkedBlockingQueue中take时hang住了。首先我们从线程名中获得了等待线程的名字，知道他是线程池里面的线程，于是就缩小了出错的范围。然后又知道是线程池试图去取任务时发生了等待，那为什么线程池还要去取任务呢？因为我们没有shutdown()线程池。

再举个几乎必然会死锁的例子：

```java
public static void main(String[] args) {

    ExecutorService executorService = Executors.newFixedThreadPool(2);
    executorService.execute(new DeadLock(1, 2));
    executorService.execute(new DeadLock(2, 1));
    executorService.shutdown();
    try {
        executorService.awaitTermination(10, TimeUnit.DAYS);
    } catch (InterruptedException e) {
        //
    }

}

static class DeadLock implements Runnable {

    private final int a, b;

    public DeadLock(int a, int b) {
        this.a = a;
        this.b = b;
    }

    @Override
    public void run() {
        for (int i = 0; i != 200; ++i) {
            try {
                TimeUnit.MILLISECONDS.sleep(100);
            } catch (InterruptedException e) {
                //
            }
            synchronized (Integer.valueOf(a)) {
                synchronized (Integer.valueOf(b)) {
                    System.out.println("我来了，又走了2");
                }
            }
        }
    }

}
```

因为`Integer.valueof(a)`在a比较小时会使用缓存，因此实际上加锁的对象都是同一个对象。所以以上代码几乎必然发生死锁。jstack结果如下：

```
> ~ jstack 9527
// 前半部分忽略。只看最后
Found one Java-level deadlock:
=============================
"pool-1-thread-2":
  waiting to lock monitor 0x00007fb7173f0758 (object 0x00000007955c0430, a java.lang.Integer),
  which is held by "pool-1-thread-1"
"pool-1-thread-1":
  waiting to lock monitor 0x00007fb7173f0808 (object 0x00000007955c0440, a java.lang.Integer),
  which is held by "pool-1-thread-2"

Java stack information for the threads listed above:
===================================================
"pool-1-thread-2":
	at com.richinfoai.umbrella.pipeline.publicFacility.utlis.Pair.lambda$main$1(Pair.java:43)
	- waiting to lock <0x00000007955c0430> (a java.lang.Integer)
	- locked <0x00000007955c0440> (a java.lang.Integer)
	at com.richinfoai.umbrella.pipeline.publicFacility.utlis.Pair$$Lambda$2/2110121908.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
"pool-1-thread-1":
	at com.richinfoai.umbrella.pipeline.publicFacility.utlis.Pair.lambda$main$0(Pair.java:27)
	- waiting to lock <0x00000007955c0440> (a java.lang.Integer)
	- locked <0x00000007955c0430> (a java.lang.Integer)
	at com.richinfoai.umbrella.pipeline.publicFacility.utlis.Pair$$Lambda$1/204349222.run(Unknown Source)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)

Found 1 deadlock.
```

下面再举个例子，线程执行时间很长的情况。（linux下模拟的，因为macOS下我竟然找了一下午都没有找到一个能显示线程ID和线程使用CPU状况的命令，linux下的`top -H 、ps -eLf`macOS下通通不支持。）测试机器状态如下：

```java
> ~ cat /proc/version
Linux version 3.10.0-957.el7.x86_64 (mockbuild@kbuilder.bsys.centos.org) 
  (gcc version 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) ) #1 
    SMP Thu Nov 8 23:39:32 UTC 2018
```

我们在java中开个线程，运行一个无限循环。

代码很简单:

```java
public class InfinityLoop{

    public static void main(String[] args){

            int i = 0;
            while (true){
                    System.out.println(i++);
            }
    }

}
```

我们搞到测试机上跑起来。

```
> ~ java -cp . InfinityLoop > output.txt &
> ~ jps
14889 InfinityLoop
> ~ jstack 14889
// 前后省略没用的输出
"main" #1 prio=5 os_prio=0 tid=0x00007fe510009000 nid=0x3a2a runnable [0x00007fe519804000]
   java.lang.Thread.State: RUNNABLE
	at java.io.FileOutputStream.writeBytes(Native Method)
	at java.io.FileOutputStream.write(FileOutputStream.java:326)
	at java.io.BufferedOutputStream.flushBuffer(BufferedOutputStream.java:82)
	at java.io.BufferedOutputStream.flush(BufferedOutputStream.java:140)
	- locked <0x0000000080214020> (a java.io.BufferedOutputStream)
	at java.io.PrintStream.write(PrintStream.java:482)
	- locked <0x0000000080214000> (a java.io.PrintStream)
	at sun.nio.cs.StreamEncoder.writeBytes(StreamEncoder.java:221)
	at sun.nio.cs.StreamEncoder.implFlushBuffer(StreamEncoder.java:291)
	at sun.nio.cs.StreamEncoder.flushBuffer(StreamEncoder.java:104)
	- locked <0x0000000080214140> (a java.io.OutputStreamWriter)
	at java.io.OutputStreamWriter.flushBuffer(OutputStreamWriter.java:185)
	at java.io.PrintStream.write(PrintStream.java:527)
	- locked <0x0000000080214000> (a java.io.PrintStream)
	at java.io.PrintStream.print(PrintStream.java:597)
	at java.io.PrintStream.println(PrintStream.java:736)
	- locked <0x0000000080214000> (a java.io.PrintStream)
	at InfinityLoop.main(InfinityLoop.java:7)
```

其nid即linux本地线程id为`0x3a2a`，转化为十进制为`14890`，我们继续排查。

```
> ~ ps -mp 14889 -o THREAD,tid,time
USER     %CPU PRI SCNT WCHAN  USER SYSTEM   TID     TIME
root     99.8   -    - -         -      -     - 00:02:27
root      0.0  19    - futex_    -      - 14889 00:00:00
root     99.3  19    - -         -      - 14890 00:02:27
root      0.0  19    - futex_    -      - 14891 00:00:00
root      0.0  19    - futex_    -      - 14892 00:00:00
root      0.0  19    - futex_    -      - 14893 00:00:00
// 省略其他输出
```

可见这个线程疯狂占内存了。当然，这也是我们期待的结果咯。一般排查的话应该是反过来，先ps看哪个线程耗资源比较大，然后去jstack查看具体的java线程，看其线程栈到底在做什么。

最后放一个Spring web请求的stack，实际任务只有一个，就是调用Spring Repository JPA保存一个entity，其线程栈如下：

```
"http-nio-8080-exec-1" #30 daemon prio=5 os_prio=31 tid=0x00007fb2a94c3000 nid=0x5c03 runnable [0x0000700009cae000]
   java.lang.Thread.State: RUNNABLE
    at java.net.SocketInputStream.socketRead0(Native Method)
    at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
    at java.net.SocketInputStream.read(SocketInputStream.java:171)
    at java.net.SocketInputStream.read(SocketInputStream.java:141)
    at com.mysql.cj.protocol.ReadAheadInputStream.fill(ReadAheadInputStream.java:107)
    at com.mysql.cj.protocol.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:150)
    at com.mysql.cj.protocol.ReadAheadInputStream.read(ReadAheadInputStream.java:180)
    - locked <0x0000000740afa5e0> (a com.mysql.cj.protocol.ReadAheadInputStream)
    at java.io.FilterInputStream.read(FilterInputStream.java:133)
    at com.mysql.cj.protocol.FullReadInputStream.readFully(FullReadInputStream.java:64)
    at com.mysql.cj.protocol.a.SimplePacketReader.readHeader(SimplePacketReader.java:63)
    at com.mysql.cj.protocol.a.SimplePacketReader.readHeader(SimplePacketReader.java:45)
    at com.mysql.cj.protocol.a.TimeTrackingPacketReader.readHeader(TimeTrackingPacketReader.java:52)
    at com.mysql.cj.protocol.a.TimeTrackingPacketReader.readHeader(TimeTrackingPacketReader.java:41)
    at com.mysql.cj.protocol.a.MultiPacketReader.readHeader(MultiPacketReader.java:54)
    at com.mysql.cj.protocol.a.MultiPacketReader.readHeader(MultiPacketReader.java:44)
    at com.mysql.cj.protocol.a.NativeProtocol.readMessage(NativeProtocol.java:549)
    at com.mysql.cj.protocol.a.NativeProtocol.checkErrorMessage(NativeProtocol.java:725)
    at com.mysql.cj.protocol.a.NativeProtocol.sendCommand(NativeProtocol.java:664)
    at com.mysql.cj.protocol.a.NativeProtocol.sendQueryPacket(NativeProtocol.java:979)
    at com.mysql.cj.protocol.a.NativeProtocol.sendQueryString(NativeProtocol.java:914)
    at com.mysql.cj.NativeSession.execSQL(NativeSession.java:1150)
    at com.mysql.cj.jdbc.ConnectionImpl.setAutoCommit(ConnectionImpl.java:2064)
    - locked <0x0000000740a6e9e8> (a com.mysql.cj.jdbc.ConnectionImpl)
    at com.zaxxer.hikari.pool.ProxyConnection.setAutoCommit(ProxyConnection.java:388)
    at com.zaxxer.hikari.pool.HikariProxyConnection.setAutoCommit(HikariProxyConnection.java)
    at org.hibernate.resource.jdbc.internal.AbstractLogicalConnectionImplementor.begin(AbstractLogicalConnectionImplementor.java:67)
    at org.hibernate.resource.jdbc.internal.LogicalConnectionManagedImpl.begin(LogicalConnectionManagedImpl.java:263)
    at org.hibernate.resource.transaction.backend.jdbc.internal.JdbcResourceLocalTransactionCoordinatorImpl$TransactionDriverControlImpl.begin(JdbcResourceLocalTransactionCoordinatorImpl.java:236)
    at org.hibernate.engine.transaction.internal.TransactionImpl.begin(TransactionImpl.java:80)
    at org.springframework.orm.jpa.vendor.HibernateJpaDialect.beginTransaction(HibernateJpaDialect.java:183)
    at org.springframework.orm.jpa.JpaTransactionManager.doBegin(JpaTransactionManager.java:401)
    at org.springframework.transaction.support.AbstractPlatformTransactionManager.getTransaction(AbstractPlatformTransactionManager.java:378)
    at org.springframework.transaction.interceptor.TransactionAspectSupport.createTransactionIfNecessary(TransactionAspectSupport.java:474)
    at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:289)
    at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:98)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
    at org.springframework.dao.support.PersistenceExceptionTranslationInterceptor.invoke(PersistenceExceptionTranslationInterceptor.java:139)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
    at org.springframework.data.jpa.repository.support.CrudMethodMetadataPostProcessor$CrudMethodMetadataPopulatingMethodInterceptor.invoke(CrudMethodMetadataPostProcessor.java:135)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
    at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:93)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
    at org.springframework.data.repository.core.support.SurroundingTransactionDetectorMethodInterceptor.invoke(SurroundingTransactionDetectorMethodInterceptor.java:61)
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
    at org.springframework.aop.framework.JdkDynamicAopProxy.invoke(JdkDynamicAopProxy.java:212)
    at com.sun.proxy.$Proxy79.save(Unknown Source)
    at com.lifeStory.study.test.TestThreadLongTimeRun.longTimeRun(TestThreadLongTimeRun.java:20)
    at com.lifeStory.study.controller.TestController.testThreadRunTooLong(TestController.java:26)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:189)
    at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:138)
    at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:102)
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:895)
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:800)
    at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87)
    at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1038)
    at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:942)
    at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1005)
    at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:897)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:634)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:882)
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:741)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:231)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:99)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:92)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.HiddenHttpMethodFilter.doFilterInternal(HiddenHttpMethodFilter.java:93)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:200)
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:107)
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:193)
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:166)
    at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:199)
    at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:96)
    at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:490)
    at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:139)
    at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92)
    at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:74)
    at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:343)
    at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:408)
    at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:66)
    at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:834)
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1417)
    at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49)
    - locked <0x00000007a2ba9450> (a org.apache.tomcat.util.net.NioEndpoint$NioSocketWrapper)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61)
    at java.lang.Thread.run(Thread.java:748)
```

在没有异常的情况下看看线程栈，也是个很爽的事情不是。

#### jmap

```
> ~ jmap -h
Usage:
    jmap [option] <pid>
        (to connect to running process)
    jmap [option] <executable <core>
        (to connect to a core file)
    jmap [option] [server_id@]<remote server IP or hostname>
        (to connect to remote debug server)

where <option> is one of:
    <none>               to print same info as Solaris pmap
    -heap                to print java heap summary
    -histo[:live]        to print histogram of java object heap; if the "live"
                         suboption is specified, only count live objects
    -clstats             to print class loader statistics
    -finalizerinfo       to print information on objects awaiting finalization
    -dump:<dump-options> to dump java heap in hprof binary format
                         dump-options:
                           live         dump only live objects; if not specified,
                                        all objects in the heap are dumped.
                           format=b     binary format
                           file=<file>  dump heap to <file>
                         Example: jmap -dump:live,format=b,file=heap.bin <pid>
    -F                   force. Use with -dump:<dump-options> <pid> or -histo
                         to force a heap dump or histogram when <pid> does not
                         respond. The "live" suboption is not supported
                         in this mode.
    -h | -help           to print this help message
    -J<flag>             to pass <flag> directly to the runtime system
```

前面讲的几个dump内存的方法，其中一个就是jmap导出。那么具体怎么操作呢？

我们启动一个Spring Boot服务，试用一下jmap

```
> ~ jmap -heap 9527
Attaching to process ID 9527, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.131-b11

using thread-local object allocation.	// 使用了Thread Local Allocation Buffer
Parallel GC with 4 thread(s)	
// JDK1.8 默认-XX:+UseParallelGC，即Parallel Scavenge + Parallel Old
// 可以用jinfo pid或者java -XX:+PrintCommandLineFlags -version查看

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 44564480 (42.5MB)
   MaxNewSize               = 715653120 (682.5MB)
   OldSize                  = 89653248 (85.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)	// 启动了G1垃圾收集器后会大有不同

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 187695104 (179.0MB)
   used     = 43869272 (41.836997985839844MB)
   free     = 143825832 (137.16300201416016MB)
   23.372624573094885% used
From Space:
   capacity = 14680064 (14.0MB)
   used     = 14670504 (13.990882873535156MB)
   free     = 9560 (0.00911712646484375MB)
   99.93487766810826% used
To Space:
   capacity = 18350080 (17.5MB)
   used     = 0 (0.0MB)
   free     = 18350080 (17.5MB)
   0.0% used
PS Old Generation
   capacity = 90701824 (86.5MB)
   used     = 24615232 (23.47491455078125MB)
   free     = 66086592 (63.02508544921875MB)
   27.138629538475435% used

22509 interned Strings occupying 2756384 bytes.	// 字符串常量池占比
```

在启用了G1的JVM上这个输出结果有一定变化，如下：

```
> ~ jmap -heap 91204
Attaching to process ID 91204, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.131-b11

using thread-local object allocation.
Garbage-First (G1) GC with 4 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 40
   MaxHeapFreeRatio         = 70
   MaxHeapSize              = 2147483648 (2048.0MB)
   NewSize                  = 1363144 (1.2999954223632812MB)
   MaxNewSize               = 1287651328 (1228.0MB)
   OldSize                  = 5452592 (5.1999969482421875MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 1048576 (1.0MB)

Heap Usage:
G1 Heap:
   regions  = 2048
   capacity = 2147483648 (2048.0MB)
   used     = 40702448 (38.81687927246094MB)
   free     = 2106781200 (2009.183120727539MB)
   1.8953554332256317% used
G1 Young Generation:
Eden Space:
   regions  = 5		// 也可以看出Eden区是按需将region标记为eden的
   capacity = 73400320 (70.0MB)
   used     = 5242880 (5.0MB)
   free     = 68157440 (65.0MB)
   7.142857142857143% used
Survivor Space:
   regions  = 10	// 所以eden的region竟然比
   capacity = 10485760 (10.0MB)
   used     = 10485760 (10.0MB)
   free     = 0 (0.0MB)
   100.0% used
G1 Old Generation:
   regions  = 24
   capacity = 50331648 (48.0MB)
   used     = 24973808 (23.816879272460938MB)
   free     = 25357840 (24.183120727539062MB)
   49.61849848429362% used

22611 interned Strings occupying 2766112 bytes.
```

具体含义就不解释了。

第二个命令：

```
> ~ jmap -histo:live 91204 | less
 num     #instances         #bytes  class name
----------------------------------------------
   1:         59859        8873752  [C
   2:         18168        1598784  java.lang.reflect.Method
   3:         58970        1415280  java.lang.String
   4:         11160        1235200  java.lang.Class
   5:         26752         856064  java.util.concurrent.ConcurrentHashMap$Node
   6:          2739         628944  [B
   7:         10297         586104  [Ljava.lang.Object;
   8:         15701         502432  java.util.HashMap$Node
   9:          5064         436984  [Ljava.util.HashMap$Node;
  10:         19116         424272  [Ljava.lang.Class;
```

感觉很爽是不是，哈哈。不加live会把已死亡对象也输出出来，格式是一样的。就不赘述了。

然后来试一下保存内存镜像的命令。我们来写个非常简单的任务。

```java
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

public class SlowInfinityLoop {

    public static void main(String[] args) throws InterruptedException {
        List<ABC> list = new ArrayList<>();
        while (true) {
            list.add(new ABC());
            TimeUnit.MILLISECONDS.sleep(100);
        }
    }

}

class ABC {
    private String name = UUID.randomUUID().toString();
}
```

运行时加参数`-Xms20m -Xmx20m`，起来后这么操作一番：

```
> ~ jps
24402
24778 Launcher
24780 Jps

> ~ jmap -heap 24794
Attaching to process ID 24794, please wait...
Debugger attached successfully.
Server compiler detected.
JVM version is 25.131-b11

using thread-local object allocation.
Parallel GC with 8 thread(s)

Heap Configuration:
   MinHeapFreeRatio         = 0
   MaxHeapFreeRatio         = 100
   MaxHeapSize              = 20971520 (20.0MB)
   NewSize                  = 6815744 (6.5MB)
   MaxNewSize               = 6815744 (6.5MB)
   OldSize                  = 14155776 (13.5MB)
   NewRatio                 = 2
   SurvivorRatio            = 8
   MetaspaceSize            = 21807104 (20.796875MB)
   CompressedClassSpaceSize = 1073741824 (1024.0MB)
   MaxMetaspaceSize         = 17592186044415 MB
   G1HeapRegionSize         = 0 (0.0MB)

Heap Usage:
PS Young Generation
Eden Space:
   capacity = 5767168 (5.5MB)
   used     = 2119584 (2.021392822265625MB)
   free     = 3647584 (3.478607177734375MB)
   36.75259676846591% used
From Space:
   capacity = 524288 (0.5MB)
   used     = 0 (0.0MB)
   free     = 524288 (0.5MB)
   0.0% used
To Space:
   capacity = 524288 (0.5MB)
   used     = 0 (0.0MB)
   free     = 524288 (0.5MB)
   0.0% used
PS Old Generation
   capacity = 14155776 (13.5MB)
   used     = 0 (0.0MB)
   free     = 14155776 (13.5MB)
   0.0% used

2485 interned Strings occupying 181464 bytes.

> ~ jmap -dump:live,format=b,file=Desktop/heap.bin 24794	// 相对路径
Dumping heap to ~/Desktop/heap.bin ...
Heap dump file created

> ~ jhat ~/Desktop/heap.bin
Reading from ~/Desktop/heap.bin...
Dump file created Sat Aug 17 17:40:52 CST 2019
Snapshot read, resolving...
Resolving 15335 objects...
Chasing references, expect 3 dots...
Eliminating duplicate references...
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

#### jhat

dump内存我们已经玩得可以了，接下来就是如何分析内存里有什么。jhat是个非常方便的工具，能够用来分析内存dump文件里的对象信息。我们就以上例中的`heap.bin`文件为例进行分析。

```
> ~ jhat -h
Usage:  jhat [-stack <bool>] [-refs <bool>] [-port <port>] [-baseline <file>] [-debug <int>] [-version] [-h|-help] <file>

	-J<flag>          Pass <flag> directly to the runtime system. For
			  example, -J-mx512m to use a maximum heap size of 512MB
	-stack false:     Turn off tracking object allocation call stack.
	-refs false:      Turn off tracking of references to objects
	-port <port>:     Set the port for the HTTP server.  Defaults to 7000
	-exclude <file>:  Specify a file that lists data members that should
			  be excluded from the reachableFrom query.
	-baseline <file>: Specify a baseline object dump.  Objects in
			  both heap dumps with the same ID and same class will
			  be marked as not being "new".
	-debug <int>:     Set debug level.
			    0:  No debug output
			    1:  Debug hprof file parsing
			    2:  Debug hprof file parsing, no server
	-version          Report version number
	-h|-help          Print this help and exit
	<file>            The file to read

For a dump file that contains multiple heap dumps,
you may specify which dump in the file
by appending "#<number>" to the file name, i.e. "foo.hprof#3".

All boolean options default to "true"
```

设置项很多，感觉比较有用的是`-execlude`和`-bashline`，其他的暂时没看出有多少用。

随便跑一下

```
> ~ jhat ~/Desktop/heap.bin
Reading from /Users/mateng/Desktop/heap.bin...
Dump file created Sat Aug 17 17:40:52 CST 2019
Snapshot read, resolving...
Resolving 15335 objects...
Chasing references, expect 3 dots...
Eliminating duplicate references...
Snapshot resolved.
Started HTTP server on port 7000
Server is ready.
```

登陆`localhost:7000`，如下：

![JVM-jhat-20190818.jpg](https://www.history-of-my-life.com/imgs-for-md/JVM-jhat-20190818.jpg)

然后我们点击我们想观察的`ABC`这个类：

输出中大部分自己看一看含义就好，然后我们使劲往下翻。找到

> ## Other Queries
>
> Reference Chains from Rootset
>
> - [Exclude weak refs](http://localhost:7000/roots/0x7bec73328)
> - [Include weak refs](http://localhost:7000/allRoots/0x7bec73328)
>
> [Objects reachable from here](http://localhost:7000/reachableFrom/0x7bec73328)

点Exclude weak refs，然后再往下翻，找到下面这一块。

可以看到不同的线程对于这个类的实例的持有状况。

![JVM-jhat-local-references-20190818.png](https://www.history-of-my-life.com/imgs-for-md/JVM-jhat-local-references-20190818.png)

可以确认第一个线程是main线程（直接点线程的超链就能看到线程的详细信息，其中有name）。那我们现在就不管了，我们可以看清楚，就是一个ArrayList持有了ABC的实例。这个比较好的就是能查看引用链条，方便看是哪里发生了内存泄露。

然后我们再回到最开始的页面，点`Show instance counts for all classes (excluding platform)`这个链接，可以看到下面的显示。

> Instance Counts for All Classes (excluding platform)
>
> 752 `instances` of class `com.lifeStory.study.ABC`
> 0 `instances` of class `com.lifeStory.study.SlowInfinityLoop`

这个在排查OOM时还比较有用，比如说我们一个线上项目OOM后dump出来的文件显示结果如下：

![JVM-jhat-OOM-20190818.png](https://www.history-of-my-life.com/imgs-for-md/JVM-jhat-OOM-20190818.png)

其中实例最多的都是fuseki的库，实际上这些对象我们只用了一小会，之后都不用了，但没有释放，显然是代码中相关作用域出问题了。修改之就可以了。

另外提示一下，分析堆的dump文件很耗内存，而且很耗时。上面这个文件，16G的堆文件，jhat在一台服务器上执行，吃了顿饭回来还等了一小会才分析完毕。

#### VisualVM

照例来一发help

```
> ~ jvisualvm -h
Usage: /Library/Java/JavaVirtualMachines/jdk1.8.0_131.jdk/Contents/Home/lib/visualvm/platform/lib/nbexec {options} arguments

General options:
  --help                show this help
  --jdkhome <path>      path to Java(TM) 2 SDK, Standard Edition
  -J<jvm_option>        pass <jvm_option> to JVM

  --cp:p <classpath>    prepend <classpath> to classpath
  --cp:a <classpath>    append <classpath> to classpath
Module reload options:
  --reload /path/to/module.jar  install or reinstall a module JAR file

其他模块选项:
  --modules
  --refresh                 刷新所有目录
  --list                    打印所有模块, 模块版本和启用状态的列表
  --install <arg1>...<argN> 将提供的 JAR 文件作为模块安装
  --disable <arg1>...<argN> 禁用指定代码库名称的模块
  --enable <arg1>...<argN>  启用指定代码库名称的模块
  --update <arg1>...<argN>  更新所有模块或指定的模块
  --update-all              更新所有模块
  --extra-uc <arg>          添加额外的更新中心 (URL)
  --openfile <arg>          打开由 <arg> 指定的文件，该文件可能是应用程序快照、NetBeans Profiler 快照或 HPROF 堆 dump。
  --openjmx <arg>           打开 JMX 连接 (主机:端口) 指定的应用程序
  --openpid <arg>           打开进程 id 为 <arg> 的应用程序
  --openid <arg>            打开 id 为 <arg> 的应用程序

Core options:
  --laf <LaF classname> use given LookAndFeel class instead of the default
  --fontsize <size>     set the base font size of the user interface, in points
  --locale <language[:country[:variant]]> use specified locale
  --userdir <path>      use specified directory to store user settings
  --cachedir <path>     use specified directory to store user cache, must be different from userdir
  --nosplash            do not show the splash screen
```

因为visualVM是插件化的，装一堆插件上去会有很多功能。但暂时没有时间研究，就用最基本的就好了，最基本的用法在这里，注意这个应该是基于JDK1.8以下的版本：

[使用 VisualVM 进行性能分析及调优](https://www.ibm.com/developerworks/cn/java/j-lo-visualvm/index.html)，看起来也不不难，基础知识上面已经有了。

### 学会看GC日志

不同的垃圾收集器输出的日志不是完全一样的，但基本格式都差不太多，可以使用`-XX:+PrintGCDetails`参数来打开JVM的GC日志（堆内存上下限都设为20m），比如上面的`SlowInfinityLoop`，打开日志后输出如下：

```shell
[GC (Allocation Failure) [PSYoungGen: 5632K->496K(6144K)] 5632K->956K(19968K), 0.0029640 secs] [Times: user=0.01 sys=0.01, real=0.00 secs] 
# 省略28行基本一致的输出，Full GC是第29行
[Full GC (Ergonomics) [PSYoungGen: 480K->0K(4608K)] [ParOldGen: 13214K->11473K(13824K)] 13694K->11473K(18432K), [Metaspace: 8059K->8038K(1056768K)], 0.0417188 secs] [Times: user=0.23 sys=0.00, real=0.04 secs] 
[GC (Allocation Failure) [PSYoungGen: 3072K->384K(4608K)] 14545K->11857K(18432K), 0.0012428 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
# 6次类似的输出，下面的Full GC是第36行
[Full GC (Ergonomics) [PSYoungGen: 448K->0K(5632K)] [ParOldGen: 13297K->13533K(13824K)] 13745K->13533K(19456K), [Metaspace: 8038K->8038K(1056768K)], 0.0230733 secs] [Times: user=0.17 sys=0.00, real=0.02 secs] 
# 之后全部都是Full GC，总共到1864行，而且越到后来速度FullGC的间隔时间越短
# 下面这一次Full GC直接是eden区不够引起的，但是已经没可回收的内存了
[Full GC (Allocation Failure) [PSYoungGen: 4096K->4096K(5632K)] [ParOldGen: 13823K->13823K(13824K)] 17919K->17919K(19456K), [Metaspace: 8038K->8038K(1056768K)], 0.0250757 secs] [Times: user=0.16 sys=0.00, real=0.02 secs] 
# 此时已经OOM了，跳出了循环，再进行了一次Full GC，可以看到堆内存基本被清空了。
[Full GC (Ergonomics) [PSYoungGen: 4096K->0K(5632K)] [ParOldGen: 13823K->1364K(13824K)] 17919K->1364K(19456K), [Metaspace: 8038K->8038K(1056768K)], 0.0087641 secs] [Times: user=0.04 sys=0.00, real=0.01 secs] 
# 打出了堆信息
Heap
 PSYoungGen      total 5632K, used 185K [0x00000007bf980000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 4096K, 4% used [0x00000007bf980000,0x00000007bf9ae638,0x00000007bfd80000)
  from space 1536K, 0% used [0x00000007bfe80000,0x00000007bfe80000,0x00000007c0000000)
  to   space 1024K, 0% used [0x00000007bfd80000,0x00000007bfd80000,0x00000007bfe80000)
 ParOldGen       total 13824K, used 1364K [0x00000007bec00000, 0x00000007bf980000, 0x00000007bf980000)
  object space 13824K, 9% used [0x00000007bec00000,0x00000007bed551d0,0x00000007bf980000)
 Metaspace       used 8051K, capacity 8188K, committed 8448K, reserved 1056768K
  class space    used 939K, capacity 984K, committed 1024K, reserved 1048576K
# 打出了异常栈信息
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.util.Arrays.copyOf(Arrays.java:3332)
	at java.lang.AbstractStringBuilder.ensureCapacityInternal(AbstractStringBuilder.java:124)
	at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:448)
	at java.lang.StringBuilder.append(StringBuilder.java:136)
	at java.util.UUID.toString(UUID.java:380)
	at com.lifeStory.study.ABC.<init>(SlowInfinityLoop.java:21)
	at com.lifeStory.study.SlowInfinityLoop.main(SlowInfinityLoop.java:13)
```

以上面为例，GC的日志一般是这样的：

先输出这次是YGC还是FGC，之后输出GC发生的原因，之后输出此次GC所使用的垃圾收集器以及GC后内存的变化情况，之后输出这次GC消耗的时间。比如

```
[GC (Allocation Failure) [PSYoungGen: 5632K->496K(6144K)] 5632K->956K(19968K), 0.0029640 secs] [Times: user=0.01 sys=0.01, real=0.00 secs] 
```

GC指这次发生了`Young GC`，`Allocation Failure`指的是Eden区内存不够分配对象了，`PSYoungGen`是指使用的是`Parallel Svavenge`垃圾收集器，

`PSYoungGen: 5632K->496K(6144K)`是指这次被回收的区前后内存的变化，格式是：`回收前内存->回收后内存(该区域总内存)`

方括号外面的`5632K->956K(19968K)`是指，回收前堆内存、回收后堆内存，堆总内存。

之后消耗的时间。

我们这个进程运行起来后，用visualVM观察堆使用状况，如下图：

![JVM-visualVM-heapusage-loop-20190818.png](https://www.history-of-my-life.com/imgs-for-md/JVM-visualVM-heapusage-loop-20190818.png)

有意思的是，明明我们这个代码看起来没什么可回收的了，就是一直在add，从没释放过什么东西，为啥这个还回收了这么多东西呢？另外，最后曲线趋向于平缓，又是什么阶段？

首先我们来看一下内存的详细情况：

![JVM-visualVM-heapDetails-loop-20190818.png](https://www.history-of-my-life.com/imgs-for-md/JVM-visualVM-heapDetails-loop-20190818.png)

如果从进程一开始就盯着的话，会发现这个char[]一直在变来变去，每次GC他都少一大截。如果仔细追踪代码，会发现是UUID包内部使用的一个玩意。

至于最后的平稳阶段，是因为内存真的退无可退了，一直在疯狂FGC。

### 题外话：OOM可以被catch

只是没什么好的理由的话，为什么要catch Error？

## JVM与Container

### 内存到底怎么回事

之前在分析我们的一些业务代码时发现了一个小问题，那就是运行在docker容器里的JVM似乎并不能感知到K8S对于硬件资源的限制，因此在未手动指定堆大小的时候，默认会使用物理机的1/4作为堆的最大内存，1/64作为堆的初始化内存，这显然不是我们要的效果。

那么这个问题怎么解决，参考以下这两篇文章：

[JVM Memory Settings in a Container Environment](https://medium.com/adorsys/jvm-memory-settings-in-a-container-environment-64b0840e1d9e)

[JVM in a Container](https://merikan.com/2019/04/jvm-in-a-container/)

大概的解释就是，在JDK1.8_131以后，JVM增加了一个实验特性，可以感知container的硬件限制了，一般来说，需要做以下设置：`-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=2 `，三个参数同时上。但这仍然存在很大的资源浪费，会浪费一半的container内存，内存越大浪费越严重。

在JDK1.8_131之前，或者在JDK1.8_131之后但不愿意浪费一半内存的，可以使用Docker的EntryPoint特性，构造一个这样的EntryPoint

```shell
# set -XX:MaxRAM to 70% of the cgroup limit
docker run --rm -m 1g openjdk:8-jdk 
	sh -c 'exec java -XX:MaxRAM=$(( $(cat /sys/fs/cgroup/memory/memory.limit_in_bytes) * 100 / 70 )) -XX:+PrintFlagsFinal -version'
```

实际是手动读取了容器对硬件资源的限制，之后自己做了一个设置。

而在更高级别的JDK（实测JDK1.8_191即可，更低版本未测试），则可以使用以下参数：

`-XX:+UseContainerSupport -XX:MaxRAMPercentage=70.00`，就能设置JVM使用container70%的内存。

下面我们用docker测一下，首先为了控制版本，我们先吧openjdk的tag打出来，虽然docker本身没有提供搜索tag的功能，但是官方给出了一个小脚本，如下：

```shell
i=0

while [ $? == 0 ]
do
   i=$((i+1))
   curl https://registry.hub.docker.com/v2/repositories/library/$1/tags/?page=$i 2>/dev/null|jq '."results"[]["name"]'

done
```

运行一下，要等很久才能输出所有结果，总共有2482行，查看一下，然后运行其中一个选定的版本。

```shell
# 测试一下 131
# 不加相关参数，可见堆内存并没有受到300M的限制，至于为啥是444.5
# 是因为mac下的docker Desktop自己限制了自己用2G内存，而java默认用1/4
> ~  docker run --rm -it -m 300M openjdk:8u131-alpine java -XshowSettings:vm -version
VM settings:
    Max. Heap Size (Estimated): 444.50M
		// 之后还有部分输出省略，下同。

# 第一种方式
> ~ docker run --rm -it -m 300M openjdk:8u131-alpine java -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=2 -XshowSettings:vm -version
VM settings:
    Max. Heap Size (Estimated): 133.50M

# 131不支持UseContainerSupport参数
> ~ docker run --rm -it -m 300M openjdk:8u131-alpine java -XX:+UseContainerSupport -XX:MaxRAMPercentage=70.00 -XshowSettings:vm -version
Unrecognized VM option 'UseContainerSupport'
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.

# 测试一下181
# 什么都不加
> ~ docker run --rm -it openjdk:8u181-alpine java -XshowSettings:vm -version
VM settings:
    Max. Heap Size (Estimated): 444.50M

# 加上内存限制，可见似乎181已经可以自己感知container内存大小了，但并不是1/4，看起来是有最小值
> ~ docker run --rm -it -m 300M openjdk:8u181-alpine java -XshowSettings:vm -version
VM settings:
    Max. Heap Size (Estimated): 121.81M

# 加上实验特性，还是会接受设置，确实用了1/2
> ~ docker run --rm -it -m 300M openjdk:8u181-alpine java -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=2 -XshowSettings:vm -version
VM settings:
    Max. Heap Size (Estimated): 145.00M

# 确实用了70%
> ~ docker run --rm -it -m 300M openjdk:8u181-alpine java -XX:+UseContainerSupport -XX:MaxRAMPercentage=70.00 -XshowSettings:vm -version
VM settings:
    Max. Heap Size (Estimated): 203.00M
    
# 191效果与181完全相同
```

### 版本到底怎么回事

看docker openjdk的tag，茫茫多，总共2482个，而且很多tag看起来非常接近：

```
"8-alpine3.9"
"8-alpine"
"8-alpine3.8"
"8u181-jre"
"8u181"
"8u181-jdk"
"8u181-jdk-alpine"
"8u181-jre-alpine"
"8u181-jre-alpine3.8"
"8u181-jdk-alpine3.8"
```

以上只展示了8的一小部分，看起来其命名基本上是`大版本号[小版本号]-[jdk|jre]-baseos[版本]`

如果不指定小版本号，那么会使用最新的镜像，显然是会不断变化的，于是我们没法精确控制这个版本，虽然说相同的大版本应该向后兼容，但是保持开发环境和部署环境的一致还是非常重要的，因此最好明确指定版本号，比如`8u131`这样的。

而jdk和jre的区别，是jre只有java runtime环境，体积会比JDK小一点，但是jmap这样的调试工具就一概没有了。个人认为还是JDK+精确版本控制比较好。

至于baseOS，一般能用alpine就用他，因为非常精简，能控制镜像体积。