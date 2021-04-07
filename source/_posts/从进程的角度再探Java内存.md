---
title: 从进程的角度再探Java内存
date: 2020-04-27 14:00:13
tags: Java
---

# 从进程的角度再探Java内存

`Java`的内存模型大家都很熟悉了，比如运行时内存分为：线程栈、本地方法栈、程序计数器、方法区、运行时常量池、堆、本地内存。堆又进一步分为新生代、老年代，新生代又进一步分为eden区和两个survivor区，方法区在JDK1.7之前的hotspot虚拟机的实现中用的是永生代这个概念，但在JDK1.8以后挪到了本地内存空间中去，其他的虚拟机实现比如JRocket一直没有永生代这个概念。

但同样广为人知的是，Java虚拟机不过是操作系统的一个进程而已。在操作系统看来，这个进程和其他进程并没有什么本质的区别。假设这个虚拟机当前运行在Linux系统中，那么经典的内存分页、交换空间、虚拟内存空间模型，是不是应该还是适用于这个虚拟机。

于是引发了很多思考：

1. 经典的Linux进程的内存模型，也有堆和栈的概念，这里的堆和栈，和Java运行时内存空间的堆和栈是什么关系？Java运行时内存中的`本地内存`，到底又和Linux经典内存模型中的哪一部分对应？

2. 一个Java进程，能不能申请比物理内存更大的堆空间？如果能，他实际占用的物理内存是多大？

3. Java有OOM错误，操作系统也有OOM错误，这两者之间有什么关系？两者在导致进程死亡上到底有哪些不同？

4. 如果再引入一个变量：docker，则问题就更有意思一点。docker daemon只是操作系统的一个进程这个毫无疑问，但是每一个启动起来的container，在操作系统看来又是以什么形态存在的呢？在docker中运行的JVM到底能使用多大的内存，能不能利用经典Linux内存模型中的换页机制？如果超出了docker的内存限制，这时候JVM进程到底是被谁杀死？是docker daemon、操作系统，还是JVM主动停止。

   对这个问题我们可以设想一个场景，在一个物理内存为16G的机器上，启动了一个内存限制为2G的docker，JVM的堆大小能不能设置成3G？假如能，是不是真的能申请到3G内存？假如不能，这个docker里运行的JVM最大能使用多大的堆？超出了会触发谁的OOM机制？

往往是复杂的问题让人兴奋，抽丝剥茧，看到纷繁表象下奔腾在电路板上的二进制洪流。那一刻的光芒，是蔷薇园里最好的一滴晨露，是草原上金线勾勒的鬓角，是中庭下樱花一点的绛唇，是一个程序员的高光时刻。

## 经典的Linux内存模型简介

### 运行时内存的几个部分

翻开《程序员的自我修养》这本书第三章：可执行文件的装载与进程。其中讲到：

进程的**虚拟内存空间**中有部分留作操作系统使用，在32位的linux系统上，一般是虚拟内存空间中最高的那1G空间，而用户只能使用3G空间。而这3G空间中，又主要分为这么几块空间：

+ 代码段。就是我们的代码编译成本机可执行文件后的那个东西，也就是我们平时编译后得到的那个文件，在运行时装载到代码段。
+ 数据段。这就是我们代码里的一些常量数据。比如说在代码里写`std:string a = "hello world!"`，这个字符串就是个常量数据，不会保存到代码段，而是放到数据段。
+ 堆空间。这个堆是Linux进程内存模型中的堆，要注意区分和`Java`运行时内存模型中的堆不是一个概念，虽然他们的功能是非常接近的，就是为进程在运行时动态分配的对象提供内存空间。
+ 栈空间。这个栈也是Linux进程内存模型中的栈，要注意区分和`JVM`内存模型中的线程栈、本地线程栈也不是完全一样，虽然他们的功能也是非常接近的，就是为进程提供一个运行栈。

+ 其实细究起来还有很多其他的段，但和我们的讨论相关性很低，就先不管了。

### 换页机制与swap空间

只讲结果不探讨原理和为什么。

现代计算机系统中，每个进程看到的内存都是从0开始的，完整、连续、平坦的一块空间。在32位的linux系统中，一般每个进程都会认为自己有4G内存可以使用。进程看到的内存称之为虚拟内存，而显然这样的虚拟内存和物理内存的地址不可能是一一对应的。所以，实际上无论是物理内存还是虚拟内存，都被分为一些固定大小的页面（一般4k一个页面），内核空间中保存一个虚拟内存页到物理内存页的映射关系。因此不同进程看到的同样地址的虚拟内存，可以被映射到不同地址的物理内存中去。

实际上，内存管理的方式更加聪明一些，比如说虽然进程能看到4G的空间，但是他并不一定真的会用那么多，实际上很多进程就使用一点点内存。于是当这些虚拟内存没用到的时候，操作系统连虚拟内存到物理内存的映射关系都不会建立，只有当实际用到时才建立。

那么假如真的有一些进程就是比较占用空间，一直在申请新的内存，确实用到了很大的内存空间，这样的两三个进程，实际用到的内存空间就会超过物理内存，这时候操作系统可以将一些暂时不用的物理内存页的内容写到文件系统里面去（windows下的虚拟内存，注意区分这个虚拟内存和我们刚才讨论的虚拟内存不是一个概念，linux下的交换空间），保证当前的进程仍然可以运行。这个**暂时不用**，我个人的理解是严格来讲只要当前CPU执行的指令用不到的内存地址，理论上都算是暂时不用，都可以交换到文件系统里面去。

这样做的好处是允许系统中运行的进程能够**实际使用**超过物理内存的内存。这里的**实际使用**是指，假如说从进程的角度观察，自己可用的内存空间一直是4G，那么N个进程看到的虚拟空间地址就是$$4*N$$，肯定远远超过物理内存大小了。但他看到的这些内存空间并不一定实际使用，只有它确实在堆空间中申请了一大堆数据，这样才叫**实际使用**，这样的话，一个进程可能使用100M，另一个使用10M，这样很多个进程，也**未必能实际使用超过物理内存的内存**，在这种情况下，操作系统**就不必须**把物理内存交换到文件系统里去了（实际上并不是只有物理内存耗尽才交换过去，在物理内存耗尽之前，就有可能交换一部分物理内存到交换空间，原因见：[Linux交换空间](https://blog.51cto.com/wutengfei/2163283)）。

因此，即使进程实际使用的内存超过物理内存，现代操作系统也能够使用交换空间来满足需求。唯一的问题是，在文件系统进行IO是很慢的操作，和在内存里做操作慢了好几个数量级，因此应当尽量避免发生这种事情。

操作系统设置的交换空间也不是无限的，虽然可以设置地大一点，但肯定不可能无限制地设置。而一旦交换空间  +  物理内存仍不足以满足多个进程的**实际使用**时，就会出现操作系统级别的OOM。

发生了这种OOM错误以后，操作系统就会开始对现有的进程进行扫描，计算出一个`badness`评分，然后将评分高的一个或几个进程直接杀掉。操作系统的这个机制称为`OOM Killer`。这个评分的计算大概参考了以下几个维度：

- 子进程内存消耗，越多越容易被选中
- CPU密集型以及老进程，比刚启动的进程更不容易被选中
- root启动的进程更不容易被选中
- 用户可以通过控制`oom_adj`来控制进程选中优先级（范围是-17到15，值越高越容易被杀掉，值为-17时禁止该进程被杀掉）

## Linux运行时内存和Java运行时内存的对应关系

### JVM进程的堆，与Java内存模型的堆

最常用的`JVM`之一：OpenJDK，本身是用`cpp`写的。当该虚拟机运行起来以后，就成为操作系统的一个进程，和其他的操作系统进程相比，这个进程并没有什么特别之处。因此，从整体上观察，一个`JVM`进程的内存模型就是下面这样子：

![JVM与内存模型](https://www.history-of-my-life.com/imgs-for-md/JVM-and-Linux-memory-20190901-01.jpg)

其中代码区，放的是`jvm`虚拟机的代码，数据区，放的是`jvm`虚拟机的常量数据，堆区，用来放`jvm`虚拟机申请的实例，这个装载过程是由操作系统进行的，当`JVM`虚拟机运行起来以后，虚拟机才开始进行我们熟悉的`JVM`内存的初始化。所以，当考虑到`JVM`的运行时内存状况时，一个`JVM`进程的内存是这样的：

![JVM与内存模型](https://www.history-of-my-life.com/imgs-for-md/JVM-and-Linux-memory-20190901-02.jpg)

这个图是网上找的图，出处见参考部分，只有`1.7`及以前版本的`Hotspot`才有永久代，其他的虚拟机实现，以及`1.8`及以后的`Hotspot`虚拟机实现，都不再有永久代这个概念。不过既然已经有了这张图，我们先以这个图为例来讨论`JVM`的内存模型。

首先，永久代装的是JVM的方法区和运行时常量区。看起来，方法区刚好对应了传统linux内存模型中的代码区，而运行时常量池则对应了数据区。

其次，新生代、老年代装的是JVM在运行时为用户代码新建的对象。那么新生代 + 老年代，就对应了linux内存模型中的堆区了。

然后，栈本来就是按需建立的，所以JVM中的栈和linux模型中的栈基本完全对应。

可见，我们所熟悉的JVM内存，除程序计数器和栈以外，都建立在`JVM`这个进程的`堆空间`，这是显而易见的事情，因为所有的这些东西，都是`JVM`这个进程在运行时新建出来的对象。而`Java`内存模型中的`堆`空间，则是`JVM`虚拟机所建立的一段逻辑空间。

就像操作系统为每个进程建立了一个逻辑堆空间一样，`JVM`虚拟机为运行在其上的`java`代码建立了一个逻辑堆空间，所谓的永久代、新生代、老年代，都是一些数据结构所对应的逻辑空间而已。

这个问题想明白了，简直是醍醐灌顶，豁然开朗。

以前的永久代，是`jvm`这个进程在它的堆空间上分配的一段内存，它维护了这段逻辑内存与自己的进程的虚拟内存对应的一个数据结构，（没看源码的我盲猜）这个数据结构可能包含了起始地址、结束地址、最大可用空间大小等等，在`JVM`装载`java`字节码时，会将`class`文件的内容、运行时常量池放在这个部分。以前的永久代最大空间在`JVM`运行时就确定了，默认是`64M`，可以通过`-XX:MaxPermSize`来指定，当其超过了大小时，就会发生永久代溢出错误，这也是（java）的OOM错误的一种，是`JVM`主动的行为。

那么用元空间替代了永久代，并不是说元空间不受`JVM`这个进程的管控了，只是说`JVM`对其管理的方式不再像以前的永久代、或者堆那样管理罢了。对堆的管理可能还维护了一个可用空间、起始虚拟内存地址、结束内存地址等，但对于元空间，这部分数据结构可以不用维护了，直接在`JVM`进程的堆里面申请就完了，要是申请大了，甚至超过了物理机的内存，也没关系啊，操作系统给换页，只要不超过`物理内存+swap`的总大小就完了，因为`swap`空间一般还挺大的，一般不至于发生（操作系统级别的）OOM，但一旦因为这个部分的使用发生OOM，那就是操作系统的OOM，会导致操作系统随机杀进程。

所以说，所谓的用元空间代替永久代，并不是说某一块内存发生了什么神奇的变化，只是在管理这一块内存的时候，采用了一些新的概念模型，如此而已。

### 第一个问题的答案

**经典的Linux进程的内存模型，也有堆和栈的概念，这里的堆和栈，和Java运行时内存空间的堆和栈是什么关系？Java运行时内存中的`本地内存`，到底又和Linux经典内存模型中的哪一部分对应？**

Java内存模型中的本地内存、堆，其实都是分配在`Linux`经典内存模型中的`堆`中。

什么是`native memory`，翻译成本地内存、直接内存都可以。所谓的本地内存，就是`JVM`进程所使用的内存。其实我个人理解本地内存就是操作系统分配给`JVM`进程的内存，那自然从地址空间来讲，包含了java内存模型中的堆在内。只是我们在讨论时，为了理解的方便，经常将java内存模型中的堆从本地内存这个概念中独立出来。那么本地内存的作用就是：

+ 管理java heap的状态数据（用于GC）;

+ JNI调用，也就是Native Stack；

+ JIT（即使编译器）编译时使用Native Memory，并且JIT的输入（Java字节码）和输出（可执行代码）也都是保存在Native Memory；

+ NIO direct buffer；

+ Threads；

+ 类加载器和类信息都是保存在Native Memory中的。

### 第二个问题的答案

**一个Java进程，能不能申请比物理内存更大的堆空间？如果能，他实际占用的物理内存是多大？**

那是自然可以的咯，申请大了没关系，大不了内存换页啊。到底占据多大的物理内存空间，这得看`JVM`到底是怎么初始化堆的，实际上还真有不同的初始化方式：

+ 在默认情况下，`JVM`也只是建立了一个有关于java堆的数据结构，java堆的那一块内存并不会去初始化。意思是，对于操作系统来说，`JVM`这个进程并没有**实际使用**那些内存，因此操作系统也不会给这部分内存真的分配物理地址。于是，实际上这个`JVM`耗费的物理内存只有一点点，就是`JVM`本身用到的一些内存，以及java代码使用的那些内存，这两个部分。

  这种方式在大部分情况下是很好的，因为毕竟不是所有人都会主动去设置java堆的大小，而是依赖于java的默认设置（堆最小是1/64物理内存，最大1/4物理内存），而且即使设置了java堆的大小，也未必真的会用那么多。如果每个JVM启动时都把java堆内存的部分初始化一下，占用物理内存，何止是没必要，简直就是浪费，甚至可能导致操作系统内存吃紧发生问题，此外，初始化那么大一块内存，也很费时间。

  但也有一些不好，考虑一个用java实现的数据库或搜索类服务器，比如说ES，这样的一个服务肯定是你给我多少内存，我恨不得都吃掉，至少可以做缓存啊对不对，最好是启动时我就占很大一块，启动速度可以慢一点，启动后运行速度一定是越快越好。而这样的一些服务一般也是独立部署在服务器上，整台服务器资源基本上都是给他用的，也不太需要顾及其他进程的感受。那这时候，在启动时直接让`jvm`初始化java堆，这样就能避免在运行时触发缺页异常，保证启动后的速度。

+ 于是，`JVM`实际上有个参数做这个事情，`-XX:+AlwaysPreTouch`，将会在java堆空间中写满0，这样就可以实现初始化的目的了。避免在实际使用到堆空间时再触发缺页异常，影响运行时的效率。

能申请比物理内存更大的堆空间，不代表这么做是对的。一般的进程里，即使用到了比物理内存更大的内存，但其热点内存区域仍然只有一小部分，也就是说，至少频繁用到的内存页仍然可以只有一小部分，因此可以长期装载在物理内存中，不会频繁引起缺页异常导致数据在物理内存和swap之间换入换出，所以运行效率还尚可。但是`JVM`由于内存回收机制的存在，在`gc`过程中，需要扫描很大一块内存，假如堆内存大于物理内存，将会导致疯狂换页，这效率一般就低到尘埃里面去了，甚至会拖垮整个服务，所以，我们尽量不要设置大于物理内存的堆内存，而且还要考虑到内存中还有部分是给内核空间用的、给其他关键进程比如`sshd`用的，因此一般应该使用比物理空间小一些的堆内存空间。

### 第三个问题的答案

**Java有OOM错误，操作系统也有OOM错误，这两者之间有什么关系？两者在导致进程死亡上到底有哪些不同？**

Java的OOM错误，是因为java堆空间已经满了，而且经过了GC仍不足以分配内存空间，这时候`JVM`会抛出一个`OutOfMemoryError`，这个错误属于java异常体系的一部分，继承自`Throwable`，可以用如下方式被捕获：

```java
public void oomMethod(){
    try {
        // do some stupid thing to cause OOM
    } catch (OutOfMemoryError e){
      	// do something to recover, but you really need a good reason to catch 
      	// an OOM error.
    }
}
```

Spring Boot在引入web依赖时，就会帮我们catch这个Error，从而我们的服务可以发生OOM，打出异常栈之后，该服务还能继续接受和处理其他请求。

但操作系统的OOM，是因为所有运行在该操作系统下的进程所**实际使用**的内存，超出了物理内存和交换空间之和，此时操作系统会根据一些参数计算出一个score，然后把score最大的一个或几个进程直接杀掉，一般就是占用内存最大的那个，（也就是我们的主服务了），这个过程可以人为干预，但仍然有一定的随机性。

所以，java的OOM是自己检测到无法运行进程了需要退出，是个主动的行为，因此也可以catch后中止这个退出行为；而操作系统的OOM杀进程对于JVM来说，是一个被动的行为，一旦被操作系统选中，作为一个普通进程的jvm将没有任何办法，只能被干掉。

举一个现实中可能会遇到的情况，我们的线上服务在发生OOM时，有时候会在日志中打出一个OOM的异常栈，然后服务还是启动状态，仍然能接受下一次请求；而在有些情况下，k8s的container会直接重启，查看重启原因会发现是`OOM Killed`，这就是操作系统的`OOM`杀了，而因为我们的容器里实际上只运行了`JVM`，其他进程并不占内存，所以我们可以大胆猜测，是运行时的`JVM`申请了太多的本地内存，最终导致实际使用的内存超过了物理内存和交换空间之和导致OOM杀，那本地内存在`JVM`中到底怎么申请的呢？一般会大量使用本地内存、而且在运行前不好估计使用量的，就是NIO框架，以及Spring大量使用的反射机制，所以，调低堆大小，给本地内存留出更大的空间，理论上可以解决这个问题。

## 终章：与Docker的战斗

### docker中的进程，也只是一个进程

想直观地观察到上面的结论，没法用windows或者mac下的docker desktop来观察，因为docker只能运行在linux内核上，所以windows或者mac下docker desktop本质上是个虚拟机。实际上可以理解为，在windows和mac下运行了一个虚拟机，又在虚拟机中运行着docker，因为多了这层虚拟机，所以会干扰我们理解问题。于是我首先在mac上安装一个centOS 7.2的虚拟机，然后在虚拟机上安装docker，在虚拟机上观察。过程不赘。

我们在虚拟机中运行一个mysql

```shell
> ~ docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=password mysql:5.7
4a679a362f02fe513bc66b10f1308e42ba99500f1a12ff18587c8eb1435baae0

> ~ docker ps
CONTAINER ID    IMAGE        COMMAND                  CREATED             STATUS              PORTS                               NAMES
4a679a362f02    mysql:5.7    "docker-entrypoint.s…"   5 seconds ago       Up 4 seconds        0.0.0.0:3306->3306/tcp, 33060/tcp   awesome_hugle

> ~ docker top 4a679a362f02
UID       PID     PPID     C  STIME   TTY    TIME        CMD
polkitd   12230   12213    0  11:55   ?      00:00:00    mysqld

> ~ ps aux | grep 12230		# 第一行是我专门粘贴出来的
USER     PID    %CPU %MEM    VSZ   RSS   TTY    STAT   START   TIME  COMMAND
polkitd  12230  0.3 10.3 1136228 193620  ?      Ssl    11:55   0:00  mysqld

# 下面是解释为什么USER是这么诡异的一个值
# 参考https://blog.dbi-services.com/how-uid-mapping-works-in-docker-containers/
> ~ docker exec -it 4a679a362f02 id mysql
uid=999(mysql) gid=999(mysql) groups=999(mysql)

> ~ id 999
uid=999(polkitd) gid=998(polkitd) 组=998(polkitd)
# 可见容器里uid为999的用户是mysql，对应着容器外的polkitd，哈哈，虽然知道
# 这个冷知识看起来没有多大用，但是有疑惑还是顺手查一查比较安心
```

所以，我们可以看到，运行在容器中的mysql对于操作系统来说也是一个进程而已。（在mac下观察不到这个进程，是因为mac的docker还运行了一层虚拟机）

### 真的没有特别之处吗？

从理论上讲是有一些。docker通过linux的nameSpace技术实现了容器之间的资源隔离，并且使用linux的Cgroup技术，限制了容器中的进程所能使用的硬件资源。所以对于每一个运行在容器中的进程来说，他只能看到容器允许他看到的文件系统、使用容器为他单独维护的网络环境，这是一个完整的、完全独立于宿主机和其他容器的文件系统、网络环境，并且，最多只能使用Cgroup限制的硬件资源，包括CPU资源和内存资源。

因为我们在这里讨论的是硬件资源的限制，所以我们主要来了解一下CGroups。这篇文章讲得非常透彻：[Docker 背后的内核知识——cgroups 资源限制](https://www.infoq.cn/article/docker-kernel-knowledge-cgroups-resource-isolation)，这篇文章我也没看完，但已经足够解答我们之前提到的问题了。在这里补充一点，查看当前容器的`Cgroup`限制，可以去以下目录看：

```shell
> ~ docker run --rm -it -m 20m ubuntu ls /sys/fs/cgroup/memory/
cgroup.clone_children		    memory.memsw.failcnt
cgroup.event_control		    memory.memsw.limit_in_bytes
cgroup.procs			    memory.memsw.max_usage_in_bytes
memory.failcnt			    memory.memsw.usage_in_bytes
memory.force_empty		    memory.move_charge_at_immigrate
memory.kmem.failcnt		    memory.numa_stat
memory.kmem.limit_in_bytes	    memory.oom_control
memory.kmem.max_usage_in_bytes	    memory.pressure_level
memory.kmem.slabinfo		    memory.soft_limit_in_bytes
memory.kmem.tcp.failcnt		    memory.stat
memory.kmem.tcp.limit_in_bytes	    memory.swappiness
memory.kmem.tcp.max_usage_in_bytes  memory.usage_in_bytes
memory.kmem.tcp.usage_in_bytes	    memory.use_hierarchy
memory.kmem.usage_in_bytes	    notify_on_release
memory.limit_in_bytes		    tasks
memory.max_usage_in_bytes
# 这里有好多熟悉的小伙伴啊
```

### 有关docker与JVM的几个问题

**每一个启动起来的container，在操作系统看来又是以什么形态存在的呢**

答案就是一个进程，但这个进程因为受到CGroup的限制，在申请硬件资源时会被操作系统内核直接限制。

**在docker中运行的JVM到底能使用多大的内存，能不能利用经典Linux内存模型中的换页机制**

能用多大的内存显然取决于CGroups的硬件资源限制。根据[Docker 运行时资源限制](https://blog.csdn.net/candcplusplus/article/details/53728507)中的介绍，限制容器所使用的硬件资源的参数分别如下：

+ CPU

  ```shell
  docker run --name "container1" -c 1024 image
  docker run --name "container2" -c 512 image
  ```

  参数值是一个整数类型，用于设置当前Docker容器使用cpu的权重值(默认为1024)。这是一个相对值，每个容器能够使用的cpu, 取决于它的cpu shares 占所有容器cpu share的比例: 例如:

  CPU是个稍微特殊一点的资源，假如说CPU资源不紧张的情况下，那我们也没有理由限制容器使用的CPU的数量，所以上述限制只有在CPU紧张的情况下才会发挥作用，如果二者都需要内存，那么 container1 分配到两倍于 container2 的CPU资源。

+ 内存

  内存分为物理内存和swap空间，所以docker的限制也包含这两个方面，主要包含六个参数：

  + 第一个是`--memory`，可以简写为`-m`

    ```shell
    # 此时该容器可以使用的最大的内存值为20M, 默认的swap最大值是物理内存的两倍
    > ~ docker run -m 20M image
    
    # 我们运行下面命令来验证这一点
    > ~ docker run --rm -m 20m ubuntu cat /sys/fs/cgroup/memory/memory.limit_in_bytes
    20971520
    
    > ~ docker run --rm -m 20m ubuntu cat /sys/fs/cgroup/memory/memory.memsw.limit_in_bytes
    41943040
    
    # 但是有意思的是，容器中的进程并不一定能感知到容器对该进程的硬件资源限制，比如
    > ~ docker run --rm -it -m 20m ubuntu free
                  total        used        free      shared  buff/cache   available
    Mem:        1877556      523812      414256        9128      939488     1186308
    Swap:       2064380       41728     2022652
    
    > ~ docker run --rm -it -m 20m ubuntu top
    top - 07:01:57 up  2:51,  0 users,  load average: 0.02, 0.03, 0.05
    Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
    %Cpu(s):  0.2 us,  0.2 sy,  0.0 ni, 99.5 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
    KiB Mem :  1877556 total,   416416 free,   521748 used,   939392 buff/cache
    KiB Swap:  2064380 total,  2022652 free,    41728 used.  1188244 avail Mem
    
      PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
        1 root      20   0   36616   1688   1272 R   0.0  0.1   0:00.04 top
        
    # 无论是top还是free，都并不会去读取cgroup的限制。
    # 其实JDK1.8_131之前也不会去读取CGroup的限制，导致其在计算默认堆大小时
    # 按照物理机的内存上限去申请。
    ```

  + `--memory-swap=300M`，这是指物理内存和swap加起来，总共可以用300M，必须和`-m`搭配使用，不然没意义。

    ```shell
    // 比如下面这个，实际就使用了200M的物理内存，和100M的swap空间
    docker run -m 200M --memory-swap=300M image
    ```

  + `--memory-reservation=300M`，这是一个软性限制，所谓的软性限制，是指并不是强制容器在任何时候都不超过这个内存使用值，而是说允许容器短时间内使用的内存超过这个限制，但如果系统内存紧张，会收回分配给容器中进程的资源，强迫其回到软性限制的范围内。软性限制是对物理内存的限制，因此一般来说应该比`-m`指定的值要小，比如：

    ```shell
    docker run -it -m 500M --memory-reservation 200M image
    ```

  + `--oom-kill-disable`，显然，因为docker的资源限制实际上还是使用操作系统内核的特性，所以OOM-killer也仍然由操作系统来进行，未设置这个参数时，当容器使用的内存达到了限制，操作系统内核就会杀掉这个进程，设置后，就不会杀掉这个进程。

    那么假如现在容器中进程使用的资源已经达到了上限，又不允许操作系统杀掉这个进程，此时进程再申请内存怎么办？根据[What does --oom-kill-disable do for a Docker container?](https://stackoverflow.com/questions/48618431/what-does-oom-kill-disable-do-for-a-docker-container/48618727)的介绍，似乎内核会在该进程申请内存时阻塞，直到人为干预为止。

    在未使用`-m`限制最大内存使用，又禁用了`oom-killer`的情况下，有可能该进程耗光操作系统内存，还杀不掉，导致内核随机杀掉其他进程，假如不小心杀掉了系统进程，系统就挂了。

  + `--oom-score-adj`这也是操作系统的一个参数，我们在上面介绍过，是操作系统在检测到OOM时计算`badness`分数时用的一个参数，值越大越容易被杀死。

  + `--memory-swappiness`这也是操作系统的一个参数。其实操作系统并不是只有当物理内存都耗尽的时候才启用swap空间，而是在物理内存使用到一定百分比时，就考虑启用swap空间了。将这个值设为0，意思是尽量使用物理内存，实在不行再用swap，设为100，意为积极使用swap，有机会就交换内存到swap中。

  + `--kernel-memory`，核心内存，不能被交换到swap空间的内存，就是枪支占着物理内存。

  一般来说，常用的就是`-m`和`--memory-reservation`这两个参数了。那么回到问题本身：在docker中运行的JVM，最多能用到分配给改容器的物理内存和swap空间之和的内存，当然可以把内存交换出去，虽然这不是个好主意。

**如果超出了docker的内存限制，JVM到底被谁杀死？**

操作系统啦，哈哈。到这里这个问题的答案简直就是呼之欲出。

**再来看最后那一段超长的问题假设：**

在一个物理内存为16G的机器上，启动了一个内存限制为2G的docker，JVM的堆大小能不能设置成3G？假如能，是不是真的能申请到3G内存？假如不能，这个docker里运行的JVM最大能使用多大的堆？超出了会导致触发谁的OOM机制？

这里就可以看出我们最初的假设还不够完善，我们再完善一下：

在一个物理内存为16G的机器上，启动了一个**物理内存和虚拟内存加起来**限制为2G的docker，JVM的堆大小能不能设置成3G？假如能，是不是真的能申请到3G内存？假如不能，这个docker里运行的JVM最大能使用多大的堆？超出了会导致触发谁的OOM机制？

答案是：JVM的堆大小何止能设置成3G，简直可以设置成1T，只要你不开`+AlwaysPreTouch`，这些堆又不会真的被使用，连申请都不会申请，只是一个数字，完全可以设置地非常非常大。但是并不能真的申请到超出2G的堆空间，别说2G，考虑到本地内存的占用，1.6G估计就很极限了，真申请到该JVM使用的内存（包括本地内存和Java运行时使用的内存）超过了2G，就会被操作系统OOM杀了。

## 尾声：Docker中的JVM

在`JDK1.8_131`之前，`JVM`并不会去读取容器对其使用资源的限制，而直接根据硬件资源来计算默认堆空间大小，综上所述，这样设置当然是可以启动起来的，但是在运行时随着申请的内存越来越多，超过了容器的资源限制，就会被操作系统OOM杀，也就是我们在K8S中看到的`OOM KILLED`，对于这个问题可以使用Docker的`EntryPoint`在启动时加上堆最大值和最小值。

`JDK1.8_131`时，JVM增加了两个参数用来控制这个行为，在这里我懒得找了，因为不怎么好用，至少在`JDK1.8_181`及以后，JVM提高了对容器的支持，这样就更好用了，但那几个参数我仍然没记住。具体可参见这么两个参考文章：

1. [JVM Memory Settings in a Container Environment](https://medium.com/adorsys/jvm-memory-settings-in-a-container-environment-64b0840e1d9e)
2. [JVM in a Container](https://merikan.com/2019/04/jvm-in-a-container/)

另外，通过上面的分析，我们可以得知，JVM虚拟机虽然能够使用交换内存，但是使用后很可能造成性能急剧下降，强烈不推荐这种用法。最好是合理设置小于物理内存的堆内存大小，并且令`memory-swappiness=0`，从而尽量要求操作系统不把JVM的内存交换到缓存空间中去。而本地内存中使用NIO造成的垃圾，虽然JVM也不是完全不管不顾，但是只有`FULL GC`时才能回收，而且实际上是调用了`System.gc()`，如果在虚拟机中参数中禁用了显式的gc，那么该部分内存就无法被回收，最终会触发操作系统的`OOM killer`，导致`JVM`进程被杀。

## 参考及待研究

+ Linux内存管理

  [Linux交换空间(swap space)](https://blog.51cto.com/wutengfei/2163283)

  [OOM_Killer](http://www.hulkdev.com/posts/oom_killer)

+ JVM内存管理

  [JVM 与 Linux 的内存关系详解](https://zhuanlan.zhihu.com/p/64737522)

  [JVM:能不能在16G机器上设置17G的堆？](https://www.jianshu.com/p/11ae309ab078)

+ Docker

  [Docker 背后的内核知识——cgroups 资源限制](https://www.infoq.cn/article/docker-kernel-knowledge-cgroups-resource-isolation)

  [Docker 运行时资源限制](https://blog.csdn.net/candcplusplus/article/details/53728507)

  [What does --oom-kill-disable do for a Docker container?](https://stackoverflow.com/questions/48618431/what-does-oom-kill-disable-do-for-a-docker-container/48618727)

  [JVM Memory Settings in a Container Environment](https://medium.com/adorsys/jvm-memory-settings-in-a-container-environment-64b0840e1d9e)

  [JVM in a Container](https://merikan.com/2019/04/jvm-in-a-container/)

+ JNI部分

  [在 JNI 编程中避免内存泄漏](https://www.ibm.com/developerworks/cn/java/j-lo-jnileak/index.html)

  [JNI内存管理](https://blog.csdn.net/pzysoft/article/details/79923121)

**声明：**

本文仅用于自学以及几位好友的内部交流，请勿转发给工作相关人员。

如果承蒙错爱想发博客，注明一下作者即可。

不保证文章中任何部分的正确性。
