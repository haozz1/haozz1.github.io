---
title: Java基础整理之线程
date: 2020-04-07 13:56:25
tags: Java
---



## 多线程问题

### 多线程基本概念

+ 线程

  线程是可以被CPU调度的最小运行单位。也可以认为是一种轻量级的进程。每个进程都有自己独立的地址空间，这保证了进程之间的隔离性，但也带来了进程之间相互访问的复杂性。而同进程的多个线程共享地址空间，因此相比进程来说要快的多，也更轻量一些。

+ 并行与并发

  并行：多个cpu实例或者多台机器同时执行一段处理逻辑，是真正的同时。

  并发：通过cpu调度算法，让用户看上去同时执行，实际上从cpu操作层面不是真正的同时。并发往往在场景中有公用的资源，那么针对这个公用的资源往往产生瓶颈，我们会用TPS或者QPS来反应这个系统的处理能力。

  无论是并行还是并发，CPU调度不同线程的代码块的顺序是随机的，在一个线程运行的任意时刻，都可能被剥夺CPU的使用权，加上不同的线程地址空间共享，经常会操作同一资源，这就引发了多线程环境下最让人头疼的线程安全问题。

+ 线程安全

  即一段代码在并发的情况之下，经过多线程使用，线程的调度顺序不影响任何结果。这个时候使用多线程，我们只需要关注系统的内存，cpu是不是够用即可。反过来，线程不安全就意味着线程的调度顺序会影响最终结果，如不加事务的转账代码：

+ 同步

  指的通过人为的控制和调度，保证共享资源的多线程访问成为线程安全，来保证结果的准确。线程安全的优先级高于性能。

+ 锁

  一个用来做线程同步的工具，一个锁保护了代码中的一段，这段代码称之为临界区。同一时间，只有持有锁的线程才能进入临界区代码。在java的世界里，每个Object实例都有一个内部锁，通过`synchronized`关键字和`wait()`,`notify()`,`notifyAll()`方法控制。

### 线程的状态

线程的内部状态有6个，以`enum`的形式定义在`Thread`里：

```java
public class Thread implements Runnable {
    public enum State {
        /**
         * Thread state for a thread which has not yet started.
         */
        NEW,

        /**
         * Thread state for a runnable thread.  A thread in the runnable
         * state is executing in the Java virtual machine but it may
         * be waiting for other resources from the operating system
         * such as processor.
         */
        RUNNABLE,

        /**
         * Thread state for a thread blocked waiting for a monitor lock.
         * A thread in the blocked state is waiting for a monitor lock
         * to enter a synchronized block/method or
         * reenter a synchronized block/method after calling
         * {@link Object#wait() Object.wait}.
         */
        BLOCKED,

        /**
         * Thread state for a waiting thread.
         * A thread is in the waiting state due to calling one of the
         * following methods:
         * <ul>
         *   <li>{@link Object#wait() Object.wait} with no timeout</li>
         *   <li>{@link #join() Thread.join} with no timeout</li>
         *   <li>{@link LockSupport#park() LockSupport.park}</li>
         * </ul>
         *
         * <p>A thread in the waiting state is waiting for another thread to
         * perform a particular action.
         *
         * For example, a thread that has called <tt>Object.wait()</tt>
         * on an object is waiting for another thread to call
         * <tt>Object.notify()</tt> or <tt>Object.notifyAll()</tt> on
         * that object. A thread that has called <tt>Thread.join()</tt>
         * is waiting for a specified thread to terminate.
         */
        WAITING,

        /**
         * Thread state for a waiting thread with a specified waiting time.
         * A thread is in the timed waiting state due to calling one of
         * the following methods with a specified positive waiting time:
         * <ul>
         *   <li>{@link #sleep Thread.sleep}</li>
         *   <li>{@link Object#wait(long) Object.wait} with timeout</li>
         *   <li>{@link #join(long) Thread.join} with timeout</li>
         *   <li>{@link LockSupport#parkNanos LockSupport.parkNanos}</li>
         *   <li>{@link LockSupport#parkUntil LockSupport.parkUntil}</li>
         * </ul>
         */
        TIMED_WAITING,

        /**
         * Thread state for a terminated thread.
         * The thread has completed execution.
         */
        TERMINATED;
    }
}
```

状态转换已经很清晰了，要注意的是

+ 所谓阻塞状态

  这里的`BLOCKED`与我们平时说的阻塞含义不完全一致，`BLOCKED`的仅指线程无法获得锁而无法继续时所处的状态，而我们日常所说的阻塞，还包括`TIMED_WAITING`状态，比如调用sleep后我们也说线程进入了阻塞状态，实际上在java内部，线程处于`TIMED_WAITING`状态。

+ 所谓RUNABLE

  java内部并不区分可运行的线程和正在运行的线程，一个可运行的线程，可能正在等待CPU的调度，也有可能正在使用CPU。总之它此时都是RUNABLE状态。

### 线程的终止和中断

线程的终止有两种方式：1. 其工作执行完毕；2. 出现了未捕获的异常。在java早期，还有一个stop方法让线程强制终止，并随之释放其持有的锁。但这种强制终结线程的方法显然不安全，比如转账，从一个账户转出后线程被强制终止，那么账户金额就出现了不一致。因此现在已被弃用。

除以上两种方式以外，没有方法能强制一个线程终止。线程中断提供了一种方式，请求线程终止自己。方式是，当对一个线程调用了`interrupt()`方法，线程的中断状态会被置位。线程**应该**时不时检测这个中断状态，决定如何处置。一些非常重要的线程，可能会忽略这个中断标志，而大部分线程都应该将中断标志视为自己退出的标志，即线程自己决定结束自己，比如直接跳到代码块尾部。如果该中断状态被置位时线程处于阻塞状态，即没有机会检测中断标志，则阻塞状态会结束，线程会抛出`InterruptedException`。

一般不建议将这个Exception的影响压制在过低的范围内，比如说：

```java
public void mySubMethod(){
  try{
  	sleep(10);
	} catch (InterruptedException e){
  	// 
	}
}
```

因为这样实际上会丢失中断请求，建议的处理方式是：要么直接在方法上声明抛出异常，要么在catch块中将中断标志重新置位：

```java
public void mySubMethod(){
  try{
  	sleep(10);
	} catch (InterruptedException e){
  	Thread.currentThread().interrupt();
	}
}
```

而检查线程是否中断也有两种方式，一个是Thread的静态方法`interrupted`，检查当前线程是否被中断，但该方法有副作用，会清除中断标志，一个是thread的方法`isInterrupted`，没有副作用。

### 线程的方法们

```java
static Thread currentThread() // 返回对当前正在执行的线程对象的引用。 
static Boolean interrupted() // 检测中断位，并将中断位直接清除
long getId()				// 返回该线程的标识符。 
String getName()		// 返回该线程的名称。 
int getPriority() 	// 返回线程的优先级。 
void interrupt() 		// 设置线程的中断位标志 
boolean isAlive()		// 测试线程是否处于活动状态。 
void join()					// 让调用线程等待该线程终止
void join(long millis)	// 等待该线程终止的时间最长为 millis 毫秒。 
void join(long millis, int nanos)	// 等待该线程终止的时间最长为 millis 毫秒 + nanos 纳秒。 
void setDaemon(boolean on)	// 将该线程标记为守护线程或用户线程。 
void setPriority(int newPriority)		// 更改线程的优先级。 
static void sleep(long millis)	// 在指定的毫秒数内让当前正在执行的线程休眠（暂停执行），此操作受到系统计时器和调度程序精度和准确性的影响。 
static void sleep(long millis, int nanos)	// 在指定的毫秒数加指定的纳秒数内让当前正在执行的线程休眠（暂停执行），此操作受到系统计时器和调度程序精度和准确性的影响。 
void start() 	// 使该线程开始执行；Java 虚拟机调用该线程的 run 方法。 
static void yield()	// 使当前的线程主动声明放弃CPU的使用权，重新参与CPU调度竞争
```

重点讲这么几个方法：

+ `join()`

  `thread.join()`的效果是，调用该方法的线程，等待该方法被调用的线程对象所在的线程完成工作后再继续。看下面一段代码：

  ```java
  public class TestJoin {
      public static void main(String[] args) {
          Thread thread = new JoinThread();
          thread.start();
          try {
              // 主线程等待thread的业务处理完了之后再向下运行  
              thread.join();
          } catch (InterruptedException e) {
              e.printStackTrace();
          }
      }
  }
  ```

  在这个例子里，主线程调用了thread线程的join()方法，效果是主线程等待thread线程结束才能继续。即无论在`thread.start()`之后是main被调度，还是thread被调度，效果都是thread先运行完。因为如果main先被调度，一定会立即执行到`join()`，从而放弃调度。

+ `sleep()`

  sleep()会**阻塞**当前进程，让出CPU使用权，准确地说，线程现在将进入`TIMED_WAITING`状态。虽然是Thread的静态方法，但实际上只是暂停当前的线程。在sleep()到期后，线程并不一定会立即被调度，只是进入了Runable状态，可以被调度。该操作不会放弃锁。

+ `yield()`

  当前线程主动放弃CPU的使用权，重新进入CPU的调度中，此时有可能其他线程被调用，也有可能该线程立即又被CPU调度。该操作和sleep()一样，不会放弃锁。
  
+ `setDaemon()`

  将线程设置为守护线程。一个守护线程能做的事情就是为其他线程做服务，比如说提供时钟打点、垃圾回收等等。当一个JVM里只有守护线程运行时，JVM也会退出。

### 线程和锁

线程之间地址空间共享，因此可以方便地操作同一个资源，经典的比如生产者、消费者模型，生产者线程向一个队列里一直写入数据，而消费者线程从队列中不断取出数据进行消费，因为线程的工作在任何时间都会被打断，其他线程的工作可能会插入，那么向队列中插入数据和取出数据的行为可能变得不安全，比如引起队列内部数据结构的混乱等。还比如说在多线程同时操作一个数字， 对其进行加减操作，因为加减操作不是原子的，就可能造成数据的不一致等。

为了避免这种**线程不安全情况**的发生，在多线程编程环境下，我们经常需要使用锁。

#### wait、notify和synchronized

##### monitor机制

[Java 中的 Monitor 机制](https://segmentfault.com/a/1190000016417017)

操作系统在面对 进程/线程 间同步的时候，提供了的一些同步原语，其中semaphore信号量和mutex互斥量是最重要的同步原语。但直接使用这些原语进行同步必须十分小心，很容易出错，为此，为了更容易地编写出正确的并发程序，**一些编程语言**在mutex和semaphore的基础上，提出了更高层次的同步原语monitor，monitor的重要特点是，同一个时刻，只有一个线程能进入monitor中定义的**临界区**，这使得monitor能够达到互斥的效果。但仅仅有互斥的作用是不够的，无法进入monitor临界区的线程，它们应该被阻塞，并且在必要的时候会被唤醒。

使用monitor机制的目的主要是为了互斥进入临界区，为了做到能够阻塞无法进入临界区的线程，还需要一个monitor object来协助，这个monitor object内部会有相应的数据结构，例如列表，来保存被阻塞的线程；同时由 monitor机制本质上是基于mutex这种基本原语的，所以monitor object 还必须维护一个基于mutex的锁。

此外，为了在适当的时候能够阻塞和唤醒 进程/线程，还需要引入一个条件变量，这个条件变量用来决定什么时候是“适当的时候”，这个条件可以来自程序代码的逻辑，也可以是在monitor object的内部，总而言之，程序员对条件变量的定义有很大的自主性。不过，由于monitor object内部采用了数据结构来保存被阻塞的队列，因此它也必须对外提供两个 API 来让线程进入阻塞状态以及之后被唤醒，分别是wait和notify。

在java中，每一个object都有一个对应的内部锁，即一个monitor，object提供了wait、notify、notifyAll方法、java提供了`synchronized`关键字来操作这个锁。

##### 入口集和等待集(锁池与等待池)

**注意：这是重量级锁的实现，在后文锁的中高级扩展知识中有详细阐述**

[知乎：锁池和等待池相关提问](https://www.zhihu.com/question/64725629)

对于每一个monitor，java都会为其维护一个`Entry Set`(入口集，也有翻译为锁池的)和`Wait Set`(等待集，也有翻译为等待池)，其中，EntrySet用于保存等待获取该monitor对应的内部锁的所有线程，而其WaitSet则用于存储执行了在该对象上调用了`wait()`、`wait(long)`的线程。

假设objectX是任意一个对象，moniterX为这个对象对应的内部锁，假设有线程A、B、C同时申请monitorX，胜出的线程是B，那么AC线程会被暂停(即其生命周期被调整为BLOCKED)，同时AC被存入objectX对应的Entry Set中，当B释放monitorX时，JVM会决定唤醒EntrySet中的任意一个线程，将其生命周期状态调整为RUNABLE，这个被唤醒的线程会与其他活跃线程(不在EntrySet中，且状态为RUNABLE的线程)再次抢占monitorX，如果成功申请到monitorX，该线程从entrySet中移出，否则被唤醒的线程仍然停留在EntrySet中，并再次进入BLOCKED状态，等待下次有线程放弃锁。

如果有线程在获得锁的情况下，调用了wait()方法，则他将会主动放弃锁，同时被暂停（线程的生命周期状态被设置为WAITING或者TIMED_WAITING），并进入monitorX的WaitSet。此时，线程就被称为objectX的等待线程。当其他线程在objectX上调用了`notify()`或者`notifyAll()`后，WaitSet中的任意一个(`notify()`)或所有(`notifyAll()`)等待线程会被移入EntrySet，即线程生命周期调整为RUNABLE，这些被唤醒的线程会与EntrySet中被唤醒的线程以及其他活跃线程共同抢夺monitorX，如果其中一个被唤醒的等待线程成功申请到锁，那么该线程会从EntrySet中移除，否则其会继续停留在EntrySet中，并再次被暂停。

在一些情况下，使用`notify()`可能导致死锁，死锁的原因可以参见：[从一个死锁分析wait，notify，notifyAll](https://www.jianshu.com/p/45626f4e0fc1)

简版的解释是：假如我们用一个List作为对象作为锁，那么假如有一个生产者P向该List中添加数据，两个消费者C1，C2从List中取出数据，在某一时刻，P和C2都在等待池中，消费者C1消费完毕后，list为空，然后调用了list的notify，但C2被调度，此时C2因为条件不满足，获得锁之后会立即放弃锁，也进入等待池，此时三个线程全都在等待池，等待不可能到来的资源，就形成了死锁。

为了防止这类死锁的发生，一个是可以将notify改为notifyAll，一个是可以在调用wait时传入合适的超时时间，这样可以保证在超时后，线程仍然会进入锁池。**effective java**中推荐的标准写法如下：

```java
synchronized (obj) {
    while (<condition does not hold>){
        obj.wait(timeout);
    }
    // Perform action appropriate to condition
}
```

理解了等待集和入口集的概念以后，就可以画出线程的全生命周期图：

![线程全生命周期图](https://www.history-of-my-life.com/imgs-for-md/java%E7%BA%BF%E7%A8%8B%E5%85%A8%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E5%9B%BE2019-6-11.jpg)

###### todo 存疑

这里有个小疑问，在知乎的上述回答中，该大佬称

> 从java虚拟机性能的角度来说，java虚拟机没有必要在notifyAll之后将WaitSet中的线程移入EntrySet，因为从一个队列移动到另一个队列有开销。其次，notifyAll后WaitSet中多个线程会被唤醒，但在极端情况下这些线程没有一个能获得锁，或者获得了锁但也因为其他资源不满足而无法运行，那么这个时候，这些等待线程仍然需要调用wait()进入等待状态，此时将他移出WaitSet后需要马上再移回WaitSet，这就是一种浪费。

所以，该大佬表示：

> 当其他线程在objectX上调用了`notify()`或者`notifyAll()`后，WaitSet中的任意一个(`notify()`)或所有(`notifyAll()`)等待线程会被唤醒，即线程生命周期调整为RUNABLE，这些被唤醒的线程会与EntrySet中被唤醒的线程以及其他活跃线程共同抢夺monitorX，如果其中一个被唤醒的等待线程成功申请到锁，那么该线程会从**WaitSet**中移除，否则其会继续停留在**WaitSet**中，并再次被暂停。

但在这个理论中，我没搞清楚一点，即这样难道不会造成死锁？以上面notify()导致死锁的案例为例：

假如此时P和C1在等待集，C2获取锁后处理完毕，进行notifyAll()，此时P和C1、C2共同竞争锁，结果C2再次获胜，此时按大佬的说法，P和C1仍然在等待集中，但此时C2因为队列为空无法处理，调用wait()进入等待集。但按照大佬的说法，JVM会唤醒EntrySet中的线程，但不会唤醒WaitSet中的线程，所以此时所有的线程都在WaitSet里，没有任何线程可以唤醒他们了。除非，当P和C1在等待集中被唤醒后因为抢夺锁失败，重新进入BLOCKED状态，而JVM在唤醒时也不是只唤醒WaitSet中的线程，而是唤醒所有在BLOCKED状态的线程，这样才可以避免死锁。

但这与大佬的说法不符，而且每次都要扫描WaitSet和EntrySet中的所有线程，那么还要这两个Set干嘛？每次全部扫描不就OK了。所以这个问题还是比较没搞清楚，在这里权做记录。

##### synchronized、wait、notify与notifyAll的效果

一言以蔽之，**被 synchronized 关键字修饰的方法、代码块，就是monitor机制的临界区。**例如：

```java
public class Monitor {

    private Object ANOTHER_LOCK = new Object();

    private synchronized void fun1() {
    }

    public static synchronized void fun2() {
    }

    public void fun3() {
        synchronized (this) {
        }
    }

    public void fun4() {
        synchronized (ANOTHER_LOCK) {
        }
    }
}
```

可以发现，上述的 synchronized 关键字在使用的时候，往往需要指定一个对象与之关联，例如 synchronized(this)，或者 synchronized(ANOTHER_LOCK)，synchronized 如果修饰的是实例方法，那么其关联的对象实际上是this，如果修饰的是类方法，那么其关联的对象是this.class。总之，synchronzied需要关联一个对象，而这个对象就是monitor object。

只有在Entry Set中的线程才会去竞争锁，而哪个线程获得锁则由系统的调度程序决定。当调用某个object的wait方法时，当前线程必须已经获得了锁(所以要求wait必须在synchronized块中调用)，而调用该方法将导致当前线程直接放弃锁，进入Wait Set。进入Wait Set的锁将永远不会主动**竞争**锁，除非其他线程在该Object上调用了notify或者notifyAll方法，这两个方法也要求当前线程必须获得锁，因此也必须在synchronized块中调用，如果调用的是notifyAll方法，那么所有在该object的Wait Set中的线程都会进入Entry Set，重新竞争锁，而notify方法会则随机将一个线程从wait set移动到entry set。

#### sleep与wait

sleep和wait虽然都会导致阻塞，但是wait会释放锁，而sleep不会释放锁。必须要想明白的一个问题是，锁其实只和monitor对象有关，而wait正是操作monitor对象的方法，而sleep则和锁无关。

wait存在的原因是，只有获得锁才能判断资源是否就位(以防刚判断完资源就被修改了)，但如果资源没就位获得锁就没有意义，所以就搞出了一个synchronized -- wait的结构，即“获取锁-判断资源情况-判断不满足就释放锁-然后等通知重来”，用特定的代码形式来实现“条件同步”。所以wait必须存在于synchronized代码块中，如果在未获得锁时调用，就必然抛出`IllegalMonitorStateException`异常。而sleep既然与锁无关，也就不要求放在synchronized代码块里了。

因为wait()和sleep()时线程都处于阻塞状态，因此无法检查线程的中断状态，所以如果阻塞期间中断标志被置位，都会抛出`InterruptedException`。

#### 锁的可重入性

一般来说，锁上有一个计数器。已经获得monitorX锁的线程，可以重复获得该锁，每次进入临界区，锁的计数器会加一，退出一个临界区，会减一。从而实现可重入。

#### Lock and Condition

##### synchronized的不足之处

在java5之前，要想直接使用`thread and lock`机制，必须使用`synchronized wait notify`机制。但是synchronized关键字也有一些缺点：

+ 一个锁只能等待一个条件。在并发多的情况下，多个条件可以让代码的可读性更好，也更容易实现一些。
+ 无法控制获得锁的顺序，在一些倒霉的情况下，某些线程可能总是得不到调度。而接下来要介绍的Lock就会提供公平机制(会较大降低性能)，优先选择长期未得到调度的线程。
+ synchronized是悲观锁，造成的性能损失较大。而之后要介绍的Lock是乐观锁，采用CAS(compare and swap来)加锁，性能要好得多。
+ 在尝试获取锁时阻塞且无法中断。意思是说，假如一个线程尝试进入synchronized锁定的临界区，那他就必须一直等待，无法退出，如果别人永远不释放锁，那这个线程就永远等下去。接下来要介绍的Lock就要高端一些，有`tryLock()`方法，可以直接返回，也有方法可以在等待一段时间后返回，此时当前线程将目前持有锁的线程中断，或者决定做其他工作。
+ 缺少读写锁的支持。当多线程同时读一个文件时，读操作理应相互不阻塞，而写操作本身、读写操作才需要阻塞，但synchronized显然是一刀切，读操作也相互阻塞，这不合理。
+ 无法得知锁当前的状态，即是否被锁上，有多少线程在等待锁等。
+ 在Lock的注释里，还提到，synchronized必须按顺序加锁，逆序解锁，并且必须在同一个作用域里释放，这使得一些技术难以施展，比如说一些算法可能需要chain lock，即获得nodeA的锁，然后获得nodeB，然后释放nodeA，然后获得nodeC的锁，这样加锁顺序和释放顺序就不是严格逆序，而且也不一定在一个作用域，用synchronized的话就很难应用。

可以说，synchronized是在刚开始，JDK还不够成熟时诞生的通用解决方案，任何对象都可以作为它的监视器。任何对象都可以作为monitor这种设计是否合理还有待考证，而它确实存在的这些不足，也促进了JDK1.5推出更高级的锁机制。即`java.util.concurrent.locks`包里的一些类。

##### Lock接口简介

自从JDK1.5依赖，java新增了标准包：`java.util.concurrent.locks`，除了推出了比synchronized语义更加灵活的Lock and Condition以外，还推出了很多高级线程同步工具。其实根据《Java核心技术》一书的建议，普通用户：

> 1. 最好既不使用Lock/Condition，也不使用synchronized关键字，因为这些都是比较贴近底层的同步原语，用的不好非常容易出错。concurrent包带来了很多同步工具，他们都隐藏了这样的加锁与加锁行为，很大程度上降低了程序员的心智负担。在可行的情况下，应当使用更高级的同步工具。
>
> 2. 如果synchronized已经够用，那就尽量使用synchronized，因为这需要更少的代码量，减少了出错的几率。
> 3. 如果特别需要Lock/Condition带来的新特性，才使用Lock/Condition。

但既然已经开始了解并发问题了，那总是要了解一下Lock/Condition的。

提炼了一下Lock接口的注释内容：

> 1. 介绍了一下什么是锁。在绝大多数情况下，一个锁只能被一个线程获取，即确保同一时间只有一个线程能访问共享资源，但例如读写锁这样的锁，就允许并发访问同一个资源。
> 2. synchronized关键字解锁时只能在同一作用域，且解锁顺序必须与加锁顺序严格逆序。这限制了一些高端技巧的应用。
> 3. Great power comes with Great responsibility。在Lock带来更大灵活性的同时，也带来了释放锁的额外要求，要求程序员一定要记得在finally块中解锁。如果加锁和解锁不在同一个作用域，就需要加倍小心。
> 4. Lock提供了比synchronized更多的特性。包括请求锁时不阻塞：`trylock()`，无法获得锁时立刻返回false；包括因等待锁而阻塞时，获得锁的过程可以打断，`lockInterruptibly()`，如果在调用这个方法时，或者在这个方法阻塞的任意时刻，线程的中断位被置位，则该方法立即抛出`InterruptedException`；包括请求锁时可以设置最大等待时间：`tryLock(long,TimeUnit)`，在等待超时时会返回false。
>
> 5. 目前与Lock紧密相关的几个类或接口有：`ReentrantLock` `Condition` `ReadWriteLock`

##### Lock接口的方法

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

+ `lock()`

  请求获得锁。这个方法在无法获得锁时会阻塞，而且不能打断，所以稍微也有点危险。

+ `void lockInterruptibly() throws InterruptedException`

  请求获得锁，如果方法被调用时线程已经被中断了，直接抛出`InterruptedException`。该方法在无法获得锁时会阻塞，但可以打断，打断时会抛出`InterruptedException`异常。

+ `boolean tryLock()`

  尝试获得锁并立即返回。如果成功获得锁返回true，如果失败返回false。建议的用法如下：

  ```java
  Lock lock = ...;
  if (lock.tryLock()) {
      try {
          // manipulate protected state
      } finally {
          lock.unlock();
      }
  } else {
      // perform alternative actions
  }
  ```

+ `boolean tryLock(long time, TimeUnit unit) throws InterruptedException`

  效果和`void lockInterruptibly() throws InterruptedException`类似，阻塞时被打断会抛异常，时间到期后返回false。

+ `Condition newCondition()`

  获得一个condition，在线程获得锁的时候，可以在返回的condition上调用`await()`方法，放弃锁，该方法返回时会重新获得锁。

##### ReentrantLock

就是一个可重入锁，实现了`Lock`接口的功能而已。

该锁的构造函数中有一个可选参数fair，如果将其设为true，则会在锁分配上偏爱那些长期得不到调度的线程。但该参数会导致较大的性能损失，只有在确有必要的时候，才应该使用该参数。

##### ReadWriteLock

读写锁维护了两个相关的Lock。这个接口只有两个方法：

```java
public interface ReadWriteLock {

    Lock readLock();
  
    Lock writeLock();
  
}
```

读写锁的注释梗概：

> 1. 在读写锁的实现里，只要没有写锁，读锁可以被多个线程获得，而写锁排除其他写锁和读锁。
> 2. 读写锁需要保证内存的同步，即当我们获得一个读锁时，上一个写锁中进行的所有修改都要对这个读锁可见。
> 3. 从理论上讲，采取读写锁的实现会有更好的并发性能表现，因为普通的锁在同一时刻只能被一个线程持有，而读写锁允许多个线程同时持有读锁。但实践中，这样的并发性能提升仅能在多处理器上观察到，并且受共享数据访问模式影响较大。
> 4. 读写锁是否能够提高性能，很大程度上依赖于数据被读取与被修改的频率的差别、读写操作各自耗费的时间，以及数据上的竞争大小——即同时尝试获取读锁或者写锁的数量。比如，一个初始化后的集合，如果不经常修改，但经常被搜索，比如说字典，就是使用读写锁的完美场景。但是，如果更新频率比较高，那么大部分时间数据都会被排他的写锁锁定，这就只有有限的、甚至没有并发提升了。进一步讲，如果读操作本身耗时非常短，那么读写锁本身实现所带来的复杂度耗时，甚至消解了读并发所带来的性能提升。因此，必须提前针对场景进行性能测试，才能知道是否应当使用读写锁。
> 5. 即使读写锁的基本概念十分直白，但仍然有很多细节上的决策需要不同的实现来决定。而实现上的不同也会影响不同的读写锁在一个程序中的表现。比如说：当一个写锁释放时，此时读锁和写锁同时请求，大多数实现会先把锁给写锁，因为写被认为是不频繁的操作，而个别实现会把锁给读锁，或者说有的实现就是按请求锁的顺序给；当有线程获得读锁、且此时有写锁在等待时，是否允许其他读锁加入锁争用，如果允许读锁竞争，可能会导致等待写锁的线程饿死，但只允许写锁，会导致读锁的并发性能下降；锁的可重入性，比如在线程获得写锁时，是否允许再获得读锁，或者一个读锁直接升级成写锁。总之这些实现的差别都有可能导致性能的差异，所以**必须提前针对场景进行性能测试，才能知道是否应当使用读写锁。**

##### ReentrantReadWriteLock

根据构造函数中的fair参数不同，这个锁的表现有较大的差别。其注释梗概是：

> 1. 当锁为非公平锁(默认情况)时，读写锁获得锁的顺序都是未定义的。但这个模式可以提供更高的吞吐量。
>
> 2. 当锁为公平锁时。当一个锁释放时，等待时间最长的写锁会获得锁，除非此时有一批读锁等待的时间都超过了等待时间最长的写锁，那这些读锁会一起获得读锁。
>
>    当锁为公平锁时，假如一个写锁正在等待或者写锁已被某个线程获得，那么读锁都会被阻塞，尝试在这种情况下获取读锁的线程都必须等待下一个写锁获得并释放后，假如获得该写锁的线程放弃了锁，且一个或多个等待读锁的线程等待时间都超过了等待时间最长的写锁，那么这个或这些线程会获得读锁。
>
>    当锁为公平锁时，一个尝试获得写锁的线程会一直阻塞，直到读写锁都处于释放状态。
>
> 3. 该锁允许读写锁重入，即获得读锁的线程可以继续获得读锁，获得写锁的线程继续获得写锁，同时，获得写锁的线程还可以继续获得读锁，但获得读锁的线程无法获得获得写锁。
>
> 4. 只有写锁支持condition，而读锁不支持。
>
> 5. 最多支持65535个递归写锁重入和65535个读锁。试图超过这个限制都将导致抛出一个Error。(这啥样的程序能超啊……)

同时，注释中给了两个很有意思的示例：

```java
class CachedData {
  Object data;
  volatile boolean cacheValid;
  final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
 *
  void processCachedData() {
    rwl.readLock().lock();
    if (!cacheValid) {
      // Must release read lock before acquiring write lock
      rwl.readLock().unlock();
      rwl.writeLock().lock();
      try {
        // Recheck state because another thread might have
        // acquired write lock and changed state before we did.
        if (!cacheValid) {
          data = ...
          cacheValid = true;
        }
        // 在获得写锁时可以获得读锁
        rwl.readLock().lock();
      } finally {
        rwl.writeLock().unlock(); // 释放写锁，但仍然持有读锁。
      }
    }
 *
    try {
      use(data);
    } finally {
      rwl.readLock().unlock();
    }
  }
}
```

> 读写锁最经典的使用场景是某种类型的collection上，特别是该集合需要足够大，读操作的频率远超写操作，而且读操作的时间超过了同步的负担。比如，下面这个例子：

```java
class RWDictionary {
  private final Map<String, Data> m = new TreeMap<String, Data>();
  private final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();
  private final Lock r = rwl.readLock();
  private final Lock w = rwl.writeLock();

  public Data get(String key) {
    r.lock();
    try { return m.get(key); }
    finally { r.unlock(); }
  }
  public String[] allKeys() {
    r.lock();
    try { return m.keySet().toArray(); }
    finally { r.unlock(); }
  }
  public Data put(String key, Data value) {
    w.lock();
    try { return m.put(key, value); }
    finally { w.unlock(); }
  }
  public void clear() {
    w.lock();
    try { m.clear(); }
    finally { w.unlock(); }
  }
}
```

##### Condition

条件有时候会被称为条件队列（但其实它没有什么先进先出的特性）或者条件变量。可以另一个线程在等待另一个线程执行时阻塞，直到被另一个线程通知可以继续运行。其方法如下：

```java
public interface Condition {
		// 对应Object的wait()
    void await() throws InterruptedException;
		
  	// 一个无法被打断的wait(),即使中断标志位被置位仍然会等待，直到获得锁时才返回。
  	// 返回时如果已经设置了中断标志位，则中断标志位不会清除。
    void awaitUninterruptibly();

  	// 下面这三个方法类似于wait(long timeout)，只是参数上更加灵活了
    long awaitNanos(long nanosTimeout) throws InterruptedException;
  	boolean await(long time, TimeUnit unit) throws InterruptedException;
    boolean awaitUntil(Date deadline) throws InterruptedException;
  
		// 对应Object的notify() 和 notifyAll() 方法
    void signal();
    void signalAll();
}
```

### 原子性、有序性与可见性

[三大性质总结：原子性，有序性，可见性](https://www.jianshu.com/p/cf57726e77f2)

+ 原子性

  **一个操作是不可中断的，要么全部执行成功要么全部执行失败**，不能再分了。一个原子操作总是线程安全的，因为其不可能被打断。虽然原子操作不需要加锁，但绝大部分操作都不是原子的，JAVA编程思想这本书建议：**没有能力手写JVM的**的程序员都不要尝试依赖原子性做到线程安全，该加锁加锁。

+ 有序性

  出于性能优化的原因，编译器和处理器会将指令尽兴重排序，也就会说，java程序的有序性是指：如果在本线程内观察，可以认为所有操作是有序的，但从另一个线程观察，则未必如此。有序性会影响线程安全，考虑下面这段代码：

  ```java
  public class Singleton {
      private Singleton() { }
      private static Singleton instance;
      public Singleton getInstance(){
          if(instance == null){
              synchronized (Singleton.class){
                  if(instance == null){
                      instance = new Singleton();
                  }
              }
          }
          return instance;
      }
  }
  ```

  其中`instance = new Singleton()`的行为并不是原子的，它至少可以拆为三个部分：1. 分配对象的内存空间；2.初始化对象；3.设置instance指向刚分配的内存地址。从调用new的线程观察，当new返回时，这三步都已经完成了，但实际上在JVM内部，可能会调换2与3的顺序，即先将instance指向内存空间，再初始化对象。此时这段代码有可能出现问题，即当线程A正在执行new操作时，刚好完成了new的第三步(第二步初始化还没完成甚至没有开始)，线程B进入了第一个判断，发现instance已经被赋值，不再是null，此时B将会得到一个未构造完成的instance，实际上不可用，这就是顺序性导致的线程安全问题。

+ 可见性

  指一个线程修改了共享变量以后，其他线程是不是可以立刻得知这个修改。

为解决非原子的操作不被中途打断，因此有了锁机制，锁将保证其他线程无法进入需要不被中断的代码区，所以一个获得锁的线程总是能连续完成自己的工作不被打断。

而**volatile**关键词则能：

1. 禁止涉及到相关变量的指令重排，从而使一段代码更加有序。
2. 强制处理器每次在读取数据时从主存中读取，并将结果写入主存。

所以如果上述代码改成下面这样子，就能够禁止重排，避免错误。

```java
private volatile static Singleton instance;
```

要注意的是，volatile关键词并不能保证原子性，所以多线程操作同一个变量，仍然需要加锁。

实际上，synchronized不仅可以保证方法或者代码块在运行，同一时刻只有一个方法进入临界区，同时还可以保证共享变量的内存可见性。

**上面的例子举得并不好，实际上sychronized关键字本身就能保证有序性，但网上都是这个例子，我也找不出更好的说明有序性的例子，所以权且以此举例。**

### 锁的中高阶扩展知识

[知乎：Synchronized 实现原理](https://zhuanlan.zhihu.com/p/67372870)

#### 自旋锁

线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒对CPU来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力。同时我们发现在许多应用上面，对象锁的锁状态只会持续很短一段时间，为了这一段很短的时间频繁地阻塞和唤醒线程是非常不值得的。

因此，引入了一种新的锁获得机制。在一般锁中，如果无法获得锁，线程就会进入阻塞状态，而在自旋锁中，如果请求锁的线程发现锁被占用，则（在一定时间内）一直循环尝试试图获得锁，进入一种**忙循环**状态，直到获取该锁为止。

曾经有个经典的例子来比喻自旋锁：A，B两个人合租一套房子，共用一个厕所，那么这个厕所就是共享资源，且在任一时刻最多只能有一个人在使用。当厕所闲置时，谁来了都可以使用，当A使用时，就会关上厕所门，而B也要使用，就得在门外焦急地等待，急得原地转圈，是为“自旋”。

虽然我们一般认为，忙循环是对CPU资源的一种浪费，但因为自旋锁避免了上下文调度开销，因此对于线程只会阻塞很短时间的场合时是有效的。

#### 适应自旋锁

JDK 1.6引入了更加聪明的自旋锁，即自适应自旋锁。所谓自适应就意味着自旋的时间不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。

线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。**有了自适应自旋锁，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测会越来越准确，虚拟机会变得越来越聪明。**

#### 锁消除

有些java的内置API使用了锁机制，比如StringBuffer、Vector（弃用）、HashTable（弃用），这时候虽然用户没有显式加锁，但是存在隐式的加锁操作。锁消除是一种编译器优化措施，即通过扫描，发现有些代码完全不可能存在并发问题，那就将这段代码中的加锁解锁操作全部去除以提高性能。比如说：

```java
public class TestLockEliminate {
    public static String getString(String s1, String s2) {
        StringBuffer sb = new StringBuffer();
        sb.append(s1);
        sb.append(s2);
        return sb.toString();
    }
}
```

在这段代码里，StringBuffer是函数内部的局部变量，因此不可能出现多线程同步访问，也就没有资源竞争，但StringBuffer的append操作却在内部加了锁，这显然是一种性能浪费。所以在编译期就可以大胆把加锁操作全部取消以提高性能。

#### 锁粗化

在大部分情况下，我们都希望同步块的作用范围越小越好，仅在确实需要操作共享数据时才尽兴同步，这样做的目的是尽可能缩小阻塞带来的性能损失，如果存在锁竞争，等待锁的线程也能尽快拿到锁。但在个别情况下，如果出现一系列的加锁、解锁操作，可能会导致不必要的性能损耗，所以引入了锁粗化的概念：

即将多个连续的加锁、解锁操作连接在一起，扩展成一个更大范围的锁，如一个vector在一个for循环中不断进行add操作，那么编译器就可以把加锁操作移到for循环以外。

####java对象头

这是一个准备知识。

因为在java中任意对象都可以作为锁，那么必然要有某种映射关系，来存储该对象和与他相关的锁、线程信息，比如哪个线程目前持有锁，哪些线程在等待等。java选择把这样的映射关系存储在对象头中(有时候是存储指向这样的映射关系的指针，总之通过对象头就可以找到锁信息)。

在JVM中，对象在内存中除了本身的数据外还会有个对象头，对于普通对象而言，其对象头中有两类信息：`mark word`和类型指针，另外对于数组而言还会有一份记录数组长度的数据。其中每一部分数据的长度根据虚拟机位数不同，分别是32位或者64位。

其中，对象头的`mark word`部分在锁机制中被复用，在标志位不同的时候代表不同的含义。下面是32位虚拟机的对象头在不同的锁状态下，分别代表以下含义：

![java对象头](https://www.history-of-my-life.com/imgs-for-md/java对象头与锁20190613.jpeg)

其中是否偏向锁标志位和锁标志位共同指示当前对象代表的锁处于何种状态。如处于无锁状态，即该对象没有被用作锁，则该标志位分别是0，01，处于偏向锁状态则为1，01，处于轻量级锁和重量级锁时是否偏向锁标志被占用，靠锁标志位是00还是10来指示是轻量锁还是重量锁。当锁标志位是11时该对象头与锁状态无关，此处不讨论。至于不同锁状态下对象头的具体含义，在下文会有阐述。

#### 悲观锁、乐观锁与CAS操作

锁其实只是为了保证多线程操作下的数据安全，所以只要能做到这一点，从理论上都可以称为某种锁。但对于如何保证数据安全这一点上，也有两种经典的处理模式。

+ 悲观锁

  悲观锁假设线程之间的竞争非常激烈，因此每一次操作都有可能产生冲突，所以它采取的策略是，每一次操作都先获得锁，操作完毕后释放锁，这样对一个数据同一时间都只有一个线程在操作，保证了安全性。悲观锁是个广义的概念，不仅应用于线程安全，还用于数据库安全等，假如一个数据库认为某些数据上的竞争非常激烈，那么他就会在每一次操作上加上事务，不允许其他数据操作(实际中事务还有很多级别，这是题外话)，这就可以认为是一种悲观锁。显然悲观锁的性能不会很好，在线程同步问题上，阻塞和调度线程需要内核态和用户态切换，而且还会导致CPU的上下文切换；在数据库领域，就是降低了并发数量。

+ 乐观锁

  为了缓解这个问题，推出了乐观锁。乐观锁的思想是，假设每次操作都没有竞争，我先做完我的工作，然后在写回的时候把我持有的数据oldValue和现在数据的主副本做对比，如果两个数据相等，那我认为这个数据没有其他线程操作过，如果不相等，我就再把数据从主副本载入，再做一次运算，再做一次对比。这是一个死循环，直到设置成功。写出代码大概是这样：

  ```java
  public final int getAndAddInt(Object atmoticObject, long valueOffset, int addValue) {
      int atmoticObjectValue;
      do {
          atmoticObjectValue = this.getIntVolatile(atmoticObject, valueOffset);
      } while(!this.compareAndSwapInt(atmoticObject, valueOffset, 
                           atmoticObjectValue, atmoticObjectValue + addValue));
  
      return atmoticObjectValue;
  }
  ```

  其中`compareAndSwapInt`在CPU层面是个原子操作，其中offset是atmoticObject中value相对atmoticObject对象内存地址的偏移量，如果做完运算后，对比发现atmoticObjectValue的oldValue还等于内存中的数据，我们就假设这个数据从来没被修改过，然后就将该内存的值设为新值，如果这个值被修改了，那我们就必须回来重新取值、重新做一次比较。而这个`比较-如果相等就设置`的操作，就叫做**CAS**。**CAS**就是乐观锁的核心实现。

  在数据库方面也有乐观锁的实现，即每条数据都带有一个version字段，写入时会检查并version，如果与持有的old version相同，就写入并同时自增version的值。

  可见，乐观锁的思想和自旋锁是相同的，就是认为线程阻塞的切换消耗太大，超过了让线程忙循环一段时间的消耗，因此采取了让CPU忙循环的方式处理竞争问题。但这种方式显然都有一个问题，即当线程之间的竞争真的很激烈的时候，忙循环可能抢不到锁，最后还得走线程阻塞那条路，而CAS操作可能每次都发现值不一样，不得不不断循环，还不如加锁来的快。

  因此，乐观锁适用于线程竞争不激烈的场合，性能表现较好，而悲观锁在线程竞争激烈的场合，其实效果更好。

+ CAS操作和ABA问题。

  [深入浅出CAS](https://www.jianshu.com/p/fb6e91b013cc)

  CAS操作有个重大的问题，就是它假设内存里的value与自己持有的oldValue相等，那么值就没有被修改过，但实际上有可能这个值原本为A，被某个线程修改为B后再修改为A，此时值其实已经修改，但会被认为没有修改过。

  在数字计算中，这并不是问题，因为只要相等我就可以认为这个值没有被修改过，即使被修改过，说明其他线程计算的结果刚好是他，那和没有变没有区别，数据依然是安全的。

  但维基百科中列出了一种情况，可能因为ABA问题导致错误，详见[维基：CAS与ABA问题]([https://zh.wikipedia.org/wiki/%E6%AF%94%E8%BE%83%E5%B9%B6%E4%BA%A4%E6%8D%A2#ABA%E9%97%AE%E9%A2%98](https://zh.wikipedia.org/wiki/比较并交换#ABA问题))

  为解决ABA问题，可以在数据上加上版本号，于是即使value相同，但问题变成了1A2B3A，就不会被误认为值没有被修改过了。

#### 重量级锁

[死磕synchronized关键字](https://github.com/farmerjohngit/myblog/issues/12)

即传统意义上的锁，这就是JDK1.6以前`synchronized`关键字操作的锁实现方式。利用操作系统底层的同步机制实现java中的线程同步。此时对象头中的`mark word`为指向堆中monitor对象的指针。

其实现包括入口集、等待集、owner等。这个上文已经阐述过了。与对象头的关系绘图如下：

![java重量级锁-对象头-20190613](https://www.history-of-my-life.com/imgs-for-md/java重量级锁-对象头-20190613.png)

#### 轻量级锁

[从jvm源码看synchronized](https://www.cnblogs.com/kundeg/p/8422557.html)

[轻量级锁加锁&解锁过程](https://gorden5566.com/post/1019.html)

JVM的开发者发现在很多情况下，在Java程序运行时，同步块中的代码都是不存在竞争的，不同的线程交替执行同步块中的代码。这种情况下，用重量级锁没有必要。因此JVM引入了轻量级锁的概念。线程在执行同步块之前，JVM会先在当前的线程的栈帧中创建一个`Lock Record`，其包括一个用于存储对象头中的 `mark word`（官方称之为`Displaced Mark Word`）以及一个指向对象的指针。下图右边的部分就是一个`Lock Record`。

![java-轻量级锁与对象头](https://www.history-of-my-life.com/imgs-for-md/java轻量级锁-对象头1-20190613.png)

在轻量锁的实现下，一个线程获得了该对象的轻量级锁的标志就是，该锁对象对象头中的锁标志位为00，且前30位记录了线程栈中的Lock Record的地址。

+ 加锁过程

  1. 在线程栈中创建一个`Lock Record`，将其`obj`（即上图的Object reference，也有教程称之为owner字段）字段指向锁对象。

  2. 直接通过`CAS`指令将`Lock Record`的地址存储在对象头的`mark word`中，如果对象处于无锁状态则修改成功，代表该线程获得了轻量级锁。如果失败，则进入步骤3

  3. 检查线程是否已经持有该锁（锁对象的markWord指向的是不是当前现成的栈帧范围内），如果已经持有了，那就代表这是一次锁重入。则把此次重入时新建的`Lock Record`中的`Displaced Mark Word`设为null。之后结束。

  4. 如果发现并不是当前线程占有了锁，即出现了锁争用，此时将出现**锁膨胀**，即该锁升级为重量级锁，object对象头中的mark word被修改为重量级锁的格式，同时在堆中建立monitor数据结构。

     值得注意的是，其实最新的实现中，即使发现了锁竞争，也会先自旋等待一下，如果等待不到再升级为重量级锁。

  这里比较难理解的一点是，其实线程每次尝试获得锁时都会在线程私有栈上创建一个新的`Lock Record`。

+ 解锁过程

  1. 找到最新的一个obj字段等于当前锁对象的`Lock Record`
  2. 如果`Lock Record`的`Displaced Mark Word`为null，代表这是一次重入，将`obj`设置为null后什么也不做，退栈了事。
  3. 如果`Lock Record`的`Displaced Mark Word`不为null，说明这正是当前线程第一次获得该锁时的`Lock Record`，则利用CAS指令将对象头的`mark word`恢复成为`Displaced Mark Word`。如果成功就继续，否则膨胀为重量级锁后退出。(没搞明白为什么会失败，能想到的就是这个线程在运行时其他线程尝试获得锁失败，锁已经膨胀为重量级锁所以才失败？)

#### 偏向锁

在更极端的情况下，虽然一个线程看起来有加锁的必要，但实际上这段代码在运行时只有一个线程在执行。从来没有第二个线程执行。

在这种情况下，其实连轻量级锁都用不到，因为轻量级锁在加锁和解锁过程中要用到多次CAS，这个操作也稍微有些耗时，因此丧心病狂的工程师创造了偏向锁这个概念。以应对这种虽然加了锁，但运行时只有单线程调用代码的情况。

当JVM启用了偏向锁模式（1.6以上默认开启），当新创建一个对象的时候，如果该对象所属的class没有关闭偏向锁模式，那新创建对象的`mark word`将是可偏向状态，此时`mark word中`的thread id（参见上文偏向状态下的`mark word`格式）为0，表示未偏向任何线程，也叫做匿名偏向(anonymously biased)。

+ 偏向锁的加锁

  + 当该对象第一次被线程获得锁的时候，发现是匿名偏向状态，则会用CAS指令，将`mark word`中的thread id由0改成当前线程Id。如果成功，则代表获得了偏向锁，继续执行同步块中的代码。否则说明其他线程也在操作这个对象的偏向锁并且那个线程成功了，则将偏向锁撤销，升级为轻量级锁。
  + 当被偏向的线程再次进入同步块时，发现锁对象偏向的就是当前线程，会往当前线程的栈中添加一条`Displaced Mark Word`为空的`Lock Record`中，然后继续执行同步块的代码，因为操纵的是线程私有的栈，因此不需要用到CAS指令；由此可见偏向锁模式下，当被偏向的线程再次尝试获得锁时，仅仅进行几个简单的操作就可以了，在这种情况下，`synchronized`关键字带来的性能开销基本可以忽略。
  + 当其他线程进入同步块时，发现已经有偏向的线程了，则会进入到**撤销偏向锁**的逻辑里，一般来说，会在`safepoint`中去查看偏向的线程是否还存活，如果存活且还在同步块中则将锁升级为轻量级锁，原偏向的线程继续拥有锁，当前线程则走入到锁升级的逻辑里；如果偏向的线程已经不存活或者不在同步块中，则将对象头的`mark word`改为无锁状态（unlocked），之后再升级为轻量级锁。

  所以偏向锁的逻辑是：当锁已经发生偏向后，只要有另一个线程尝试获得偏向锁，则该偏向锁就会升级成轻量级锁。

+ 偏向锁的解锁

  当有其他线程尝试获得锁时，是根据遍历偏向线程的`lock record`来确定该线程是否还在执行同步块中的代码。因此偏向锁的解锁很简单，仅仅将栈中的最近一条`lock record`的`obj`字段设置为null。需要注意的是，偏向锁的解锁步骤中**并不会修改对象头中的thread id。**

  因此如果锁已经偏向后另一个线程试图获取锁，就会立即发现此时其实并不是只有一个线程在操作锁，锁直接升为轻量级锁。

偏向锁还存在批量重偏向与撤销的问题，太过复杂，不在这里描述，可以参考[死磕synchronized关键字](https://github.com/farmerjohngit/myblog/issues/12)的最后一节。

#### 锁的升级（锁膨胀）与降级

一般来讲，一个synchronized所操作的对象上的锁会经历从无锁状态到偏向锁状态、轻量级锁到重量级锁的状态。在早期，锁只能膨胀而不能降级，但据那些拆OpenJDK源码的大神说，其实jdk8的实现中就有了锁降级机制。这个更复杂，还是不研究了。

### 高级同步工具之阻塞队列

虽然我们在前面把这些锁的知识总结了七七八八，搞得很复杂。但实际上，负责人的java参考书都会说：无论是synchronized关键字，还是Lock&Condition，都属于很贴近底层的原语，用的不好很容易挂自己。而使用由并发处理的专家实现的高级同步工具显然要方便和安全地多。

显然java也照顾到了菜鸡们的能力和感受，在jdk1.5中就推出了`java.util.concurrent`包，包含了很多高级线程同步工具。将底层程序员解放出来了。

这其中比较强大的一种工具就叫做阻塞队列。阻塞队列可以解决多线程下经典的生产者/消费者问题。通过队列作为线程之间传输数据的工具，生产者线程只管往队列中添加产品，而消费者只管取。阻塞队列本身保证存取过程是线程安全的，而因为产品的产生和消费都是在各自的线程里，一些变量不再需要共享，编写出线程安全的程序就容易一些。

BlockingQueue的接口方法如下：

```java
public interface BlockingQueue<E> extends Queue<E> {

    boolean add(E e);
    boolean offer(E e);
    void put(E e) throws InterruptedException;
    boolean offer(E e, long timeout, TimeUnit unit) throws InterruptedException;
    E take() throws InterruptedException;
    E poll(long timeout, TimeUnit unit) throws InterruptedException;
    int remainingCapacity();
    boolean remove(Object o);
    public boolean contains(Object o);
    int drainTo(Collection<? super E> c);
    int drainTo(Collection<? super E> c, int maxElements);
}
```

其中`put`和`take`方法分别在队列已满和队列为空时会阻塞。其他方法就不介绍了。因为这些队列会在一些非阻塞的取接口中返回null表示队列中没有值，因此这些阻塞队列都不允许插入null值。

阻塞队列的具体实现很多，包括：

+ ArrayBlockintQueue

  构造时指定容量，并且可以设置是否需要公平性，公平性会偏爱长期得不到调度的线程，但也会降低性能。

+ LinkedBlockingQueue

  构造时可以指定容量，但默认是没有边界，允许无限个对象放进来。这个不限制上限的同步阻塞队列，是java默认的线程池`ExecutorService`在一些消费者能力不足时引发OOM问题的罪魁祸首。以后提到线程池的时候会再说这个问题。

+ PriorityBlockingQueue

  弹出时将优先级最高的数据弹出，放入的元素必须实现了Comparable接口，如果没有，则需要在构造函数中提供一个比较器。这个队列也是无界的。

+ BlockingDeque

  接口。提供一个双向队列，相应的，它提供的方法里也有双向存取的方法。

+ DelayQueue

  放在这个队列里的元素必须实现`Delayed`接口，在一个元素自带的延迟时间超时之前，这个元素无法通过各种取方式从这个队列中取出。(还没搞明白有啥用)

+ SynchronousQueue

  [SynchronousQueue使用实例](https://www.jianshu.com/p/b7f7eb2bc778)

  这是种奇怪的队列，即该队列上必须有一个consumer在等待的时候，put操作才能成功，否则一直阻塞，(如果使用offer，则直接返回false)。具体用途目前看了一些文章，都是在说在

  ```java
  public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
      return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                    60L, TimeUnit.SECONDS,
                                    new SynchronousQueue<Runnable>(),
                                    threadFactory);
  }
  ```

  这里用到了，这就可以在线城池没有线程在等待时offer操作直接返回false，以便线程池管理工具创建新的线程来完成工作，而如果线程池已经有了空闲线程，即`SynchronousQueue`队列有人在等着消费，那就offer成功。先不深究。

+ TransferQueue

  这个有点像更灵活的`SynchronousQueue`，即当使用`transfer`方法时，必须有线程等到消费才能放入，而使用put时就更表现地像正常的`BlockintQueue`。

总之，使用阻塞队列可以简化生产者/消费者模型，使线程不必自己处理复杂的同步问题，无论是生产者还是消费者线程，都只要简单put和take，就可以完成线程安全的存取操作，并且其自带的阻塞功能可以用于线程同步。是生产者/消费者模型很好的选择。

### 高级同步工具之同步器

#### 计数门闩(CountDownLatch)

在某些场景下，主线程开启多个线程处理多个子任务，当各个子任务处理完毕后，主线程还要做一些合并工作。比如说使用了分治法的一些算法。

此时，主线程需要等待子线程任务完毕，这就是计数门闩使用的绝佳场景。

代码如下：

```java
public void process() throw InterruptedException {

    ExecutorService executorService = Executors.newCachedThreadPool();
    CountDownLatch countDownLatch = new CountDownLatch(THREAD_NUM);

    for (int i = 0; i != THREAD_NUM; ++i) {
        executorService.execute(new SomeRunable(countDownLatch));
    }
   
    countDownLatch.await();
    executorService.shutdownNow();
}
```

SomeRunable实现如下：

```java
public class SomeRunable implements Runnable {
  
    private final CountDownLatch countDownLatch;
  
    public SomeRunable(CountDownLatch countDownLatch) {
        this.countDownLatch = countDownLatch;
    }

    public void run() {
        // do some real work
        countDownLatch.countDown();
    }
}
```

#### 栅栏(Barrier)

像是一个可循环的计数门闩。考虑一个算法分为好几个阶段，每一个阶段都需要上一个阶段的所有线程都完成后才能进行，那么Barrier就比计数门闩要好用。更进一步的是，创建barrier时，可以传入一个Runable的对象作为栅栏所有线程都到达栅栏之后要执行的操作。

即：

```java
Runable barrierAction = …;
CyclicBarrier barrier = new CycliBarrier(threadNums, barrierAction);
```

这个动作执行完毕后，所有到达栅栏的线程才会继续执行。

#### Phaser

更高级的栅栏，允许改变不同阶段参与线程的个数。

#### 交换器(Exchanger)

当两个线程在同一个数据缓冲区的两个实例上工作时可以使用交换器，典型的比如一个线程往缓冲区写数据，另一个在使用，采用交换器直接交换两个缓冲区，则效率更高一些，并且不必创建新的缓冲区。

#### 同步队列(SynchronousQueue)

这是一种将生产者线程和消费者线程配对的机制，当一个线程在该队列上调用put方法时，他会阻塞一直到有消费者线程来取。反之亦然。与Exchanger不同的是，这个同步队列中数据传递方向是单向的。因为可以保证生产者线程总是要等到有消费者才返回，也可以作为一种同步工具。

### 高级同步工具之原子类

`java.util.concurrent.atomic`包中提供了一些类，他提供的很多方法采用了很高效的机器级指令来保证操作的原子性，其实就是CAS。这些方法都可以安全地在多线程中进行自增和自减操作，但是在更高级的适用场合，仍然无法保证原子性，可能需要手动CAS。

假设我们要将`AtomicLong`的一个实例加10，则可能写出这样的代码：

```java
public AtomicLong atomicLong = new AtomicLong();
// in some thread
atomicLong.set(atomicLong.get() + 10);
```

但是因为这个set和get操作并不是原子的，因此并不安全。

要想线程安全地执行这个操作，应该像下面这样写：

```java
// in some thread
long oldValue, newValue;
do {
    oldValue = atomicLong.get();
    newValue = oldValue + 10;
} while (!atomicLong.compareAndSet(oldValue, newValue));
```

看起来很眼熟，实际上这就是手动CAS了。依赖于compareAndSet的原子性，这个增加操作可以多线程安全地执行。然而手写这个while还是有点繁琐，所以在java 8中，Atomic增加了新的方法用于这种场景：

```java
atomicLong.updateAndGet(x -> x + 10);
// 或者
atomicLong.accumulateAndGet(10,Math::addExact);
```

但是要记住，Atomic系列类的底层实现是CAS，这种乐观锁机制在竞争激烈的情况下需要经常忙循环，性能急剧下降。所以Java8提供了`LongAdder`和`LongAccumulator`，`DoubleAdder`和`DoubleAccumulator`类，他在内部保存了多个变量（加数），其总和为当前值，这样就可以有多个线程更新不同的加数，而最后用sum返回结果。因此，在预估到竞争会很激烈时，用LongAdder的效果明显要好，LongAccumulator则可以提供一个运算函数，做加法以外的操作。

### 线程安全的集合

如果多线程要并发的操作一个集合，可能会破坏集合的内部结构。比如说，HashMap在桶内红黑树重排时，如果遇到线程切换，其他线程的操作可能会破坏红黑树结构，导致发生发生异常或者指针出现循环。

线程安全的集合是指，这些集合在多线程下操作绝对不会破坏其内部结构。而且他们内部使用了很复杂的算法，允许对集合的部分进行加锁，从而允许并发地访问，甚至写入集合的不同部分，使竞争最小化，以提高并行率。比如并发的哈希表，默认情况下允许16个写线程同时执行。

线程安全的集合包括：ConcurrentHashMap、ConcurrentSkipListMap、ConcurrentSkipListSet、ConcurrentLinkedQueue等。

但要注意的是，就如同上面说的原子类也不能保证set之类函数的原子性，需要手写CAS，线程安全的集合只是保证了多线程操作不会破坏集合的内部结构，而绝不是说保证了原子性，比如这样的操作：

```java
Long oldValue = map.get(word);
Long newValue = oldValue == null ? 1: oldValue + 1;
map.put(word, newValue); // Error-might not replace oldValue
```

多线程执行下不能确定结果是什么，因此安全的方法还是手写CAS：

```java
do{
    oldValue = map.get(word);
    newValue = oldValue = null ? 1 : oldValue + 1;
} while (!nap.replace(word, oldValue, newValue));
```

也可以通过java8的新接口来替代这个手工CAS：

```java
map.compute(word, (k, v) -> v = null ? 1: v + 1);
```

或者更进一步：

```java
map.computelfAbsent(word , k -> new LongAdderO) _increment() ;
```

### Runnable、Callable、Future、FutureTask

上文已经基本把线程以及线程安全的问题挖的差不多了，但搞到现在，才发现从来没有介绍怎样让Java跑起来一个多线程的程序……

在比较古老的时期，多线程需要继承Thread对象，直接覆盖Thread的run方法，把业务代码写在run里面，但这种把任务和任务调度混在一起的方法并不是一个好的实现，所以很快这种做法就被抛弃了，具体的业务代码被包装到一个Runnable对象中去，所以要写一个多线程的程序，基本上类似这种写法(在不使用线程池时)：

```java
public class TestThread {

    static class SomeRunable implements Runnable {
        @Override
        public void run() {
            System.out.println("我是线程" + Thread.currentThread().getName());
        }
    }

    public static void main(String[] args) {

        for (int i = 0; i != 10; ++i) {
            Thread thread = new Thread(new SomeRunable());
            thread.start();
        }
        
    }

}
```

在lambda表达式的加持下，代码可以写成下面这样：

```java
public class TestThread {

    public static void main(String[] args) {

        for (int i = 0; i != 10; ++i) {
            Thread thread = new Thread(() -> System.out.println("我是线程" + Thread.currentThread().getName()));
            thread.start();
        }

    }

}
```

可见，首先是实现`Runnable`接口，在其`run()`方法中写上业务代码，然后再将该对象传入Thread的构造函数中，再调用`thread.start()`方法，线程就可以启动。

但是，Runable对象有个小缺憾，就是没有返回值，如果没有返回值的话，线程想向外传递数据就只能通过其他手段，比如在Runnable构造方法中传入一个集合之类的。这有时候不太方便。因此在java1.5中出现了新的类，允许线程的业务代码返回结果，这就是Callable接口，与Runnable只有一个run方法类似，这个接口只有一个call方法。

但这里就出现了两个问题，一个是Thread的构造函数只支持传入Runable，并且没有提供返回值的接口，call方法的返回值存到哪里去，又从哪里获得呢？而且，因为线程是异步的，我怎么知道什么时候返回值准备好了呢？

这时候就出现了一个适配器，名为`FutureTask`，它同时实现了Runable和Future接口，接受一个Callable作为构造函数的参数，于是问题得到了解决，代码可以写成这样：

```java
public class TestThread {

    static class SomeCallable implements Callable<Long> {

        @Override
        public Long call() {
            return Thread.currentThread().getId();
        }

    }

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        Long sth = 0L;
        for (int i = 0; i != 10; ++i) {
            FutureTask<Long> task = new FutureTask<>(new SomeCallable());
            Thread thread = new Thread(task);
            thread.start();
            sth += task.get();
        }
        System.out.println(sth);

    }

}
```

其中Future保存了异步计算的结果，Future对象的所有者在计算结果准备好之后就可以获得它。Future的方法如下：

```java
public interface Future<V>{
	V get() throws throws InterruptedException, ExecutionException;
	V get(long timeout, TimeUnit unit) throws throws InterruptedException, ExecutionException;
	void cancel (boolean maylnterrupt);
	boolean isCancelled();
	boolean isDone();
}
```

其中get()会阻塞一直到计算完成，第二个方法如果超时就会抛出`TimeoutException`。isDone可以检测计算是否完成，而cancel可以请求取消计算，如果计算还没开始则不再开始，如果计算正在运行且mayInterrupt参数为true，则线程被中断(记住线程的中断只是个置位，具体线程如何处理还看具体线程)

### Fork/Join框架

[应用 fork-join 框架](https://www.ibm.com/developerworks/cn/java/j-jtp11137.html)

[聊聊并发：Fork/Join框架介绍](https://www.infoq.cn/article/fork-join-introduction)

#### Fork/Join的设计思路

上面第一篇文章，从编程语言要适应硬件发展的相适应的角度，详细阐述了`Fork/Join`框架诞生的历史和原因。文中说到，现代的CPU已经能在硬件层面提供更多的内核，但更多的内核如果没有相应的更细粒度的任务来驱动，那么处理器将出现空闲的风险，即使还有很多工作要处理。而`Fock/Join`就是用于表示更细粒度算法的框架。

所谓的`Fork/Join`，其实是一种分治法。即将一些更大的任务划分为相互隔离的细粒度的问题，分别交由不同的线程完成，之后再将这些线程的结果合并在一起。写出伪代码类似：

```
public T doTask(task) {
    if (scale of task is small enough) {
        slove it;
    } else {
        split into small tasks
        doTask(childTask1);
        doTask(childTask2);
        doTask...
    }
}
```

这个框架绘图出来类似这样：

![fork/join框架](https://www.history-of-my-life.com/imgs-for-md/java-fork-join1-20190614.png)

之所以叫fork/join模型，是因为先fork出一些子任务来，然后再将这些子任务的结果join在一起。

这样的一个任务模型，如果要用原生的java线程来完成也不是不可以，其实就可以用上面的那套Callable/Future工具来做，利用Future的`get()`方法阻塞来进行线程同步，不断拆分任务，交给一个callable对象完成，等待future的结果，再合并起来。

这样折腾要耗费不少精力，容易出错，而且仔细思考上面的模型，如果一个线程提前完成了任务，那么因为必须等待其他线程到达join点，即使它没工作了，也只能无所事事，所以这显然是一种浪费。另外，其实在这个模型里，线程在到达join之前相互之间并不会阻塞，使用传统的加锁方法需要考虑线程阻塞的问题，也有额外的开销。针对这两个问题，那帮高级工程师采用了两周手段。

+ **工作窃取（work stealing）**

  每个工作线程都有自己的工作队列，一般使用双端队列来实现。当一个任务派生出新的线程时，它将自己放到deque的头部，当一个任务执行与另一个未完成任务的合并操作时，它将另一个任务推到队列头部并执行，当线程的任务队列为空时，它将尝试从另一个线程的尾部窃取一个任务。

  之所以用双端队列，是因为双端队列可以减少争用，一个线程总是从队首取任务，而窃取任务的另一个线程总是从队尾窃取任务，这样就极少发生争用。而且使用deque暗含着后进先出，即当前线程总是处理刚压进去的，更琐碎一些的任务，而窃取线程总是能拿到更早压进去的，更大块的任务，这样就可以在窃取到以后再拆分任务，避免了多次窃取导致的成本。

+ **阻塞优化**

  因为线程之间除了工作窃取并不发生争用，因此线程在绝大部分情况下不会发生阻塞，于是在类库里一些线程同步操作被精心重新设计了。最大限度减少了争用。

#### Fork/Join的使用步骤

1. **分割任务**

   要求子任务之间是独立的，相互不干扰的。有时候经过分割子任务还是很大，那就递归划分，直到任务足够小，再实际处理之。

2. **执行任务并合并结果**

   分割的子任务分别放在双端队列里，然后几个启动线程分别从双端队列里获取任务执行。子任务执行完的结果都统一放在一个队列里，启动一个线程从队列里拿数据，然后合并这些数据。

   为此，我们需要创建Fork/Join任务的类，一般来说，我们继承下面两个子类。

   + RecursiveAction。用于没有返回结果的任务。
   + RecursiveTask。用于有返回结果的任务。

   同时，我们需要一个池来维护这些任务。

   + ForkJoinPool ：ForkJoinTask 需要通过 ForkJoinPool 来执行，任务分割出的子任务会添加到当前工作线程所维护的双端队列中，进入队列的头部。当一个工作线程的队列里暂时没有任务时，它会随机从其他工作线程的队列的尾部获取一个任务。

     他有三个主要方法：

     1. execute：异步执行，没有返回结果；
     2. invoke、invokeAll：异步执行，阻塞直到任务完成后返回。
     3. submit：异步执行，并立即返回一个Future对象。

来段代码。比如求一大堆数相加、求一群数字的最大值等等，以求和为例。

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ForkJoinTask extends RecursiveTask<Long> {

    private static final long MAX = 1000000000L;
    private static final long THRESHOLD = 1000L;
    private long start;
    private long end;

    public ForkJoinTask(long start, long end) {
        this.start = start;
        this.end = end;
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        Long sum = forkJoinPool.invoke(new ForkJoinTask(1, MAX));
        System.out.println(sum);
        System.out.println(System.currentTimeMillis() - start + "ms");
    }

    @Override
    protected Long compute() {
        long sum = 0;
        if (end - start <= THRESHOLD) {
            for (long i = start; i <= end; i++) {
                sum += i;
            }
            return sum;
        } else {
            long mid = (start + end) / 2;

            ForkJoinTask task1 = new ForkJoinTask(start, mid);
            task1.fork();

            ForkJoinTask task2 = new ForkJoinTask(mid + 1, end);
            task2.fork();

            return task1.join() + task2.join();
        }
    }
  
}
```

#### Fork/Join的异常处理

ForkJoinTask 在执行的时候可能会抛出异常，但是我们没办法在主线程里直接捕获异常，所以 ForkJoinTask 提供了 isCompletedAbnormally() 方法来检查任务是否已经抛出异常或已经被取消了，并且可以通过 ForkJoinTask 的 getException 方法获取异常。使用如下代码：

```java
if(task.isCompletedAbnormally())
{
    System.out.println(task.getException());
}
```

getException 方法返回 Throwable 对象，如果任务被取消了则返回 CancellationException。如果任务没有完成或者没有抛出异常则返回 null。

#### Fork/Join的适用场景

首先要任务能够被切分为子任务，而且数据集要足够大，并且每个元素处理的成本都要比较高，才能补偿建立fork/join框架所消耗的成本，而且要合理设置阈值，不能太大也不能太小，阈值太小则线程建立太多，抵消了并行运算带来的好处，太大则不能充分利用硬件和并行带来的好处。

### Future、CompletionStage与CompleteableFuture

Future提供了一种延迟获得结果的方法，但是Future提供的方法比较单一，而且如果我们要用程序语言表达：当一个线程结束以后，我们用它的结果再做一些工作这样的场景时，可能需要先用一个Future等待结果，然后用get等待结果，再将结果传入新的线程。这样的操作会异步、阻塞、再异步、再阻塞，显然很不优雅。如果之后的操作也是异步的，这样就有非常麻烦的线程同步开销。

因此，在java8中引入了新的异步执行接口，`CompletionStage`，以及该接口的实现类`CompleteableFuture`，实际上，`CompleteableFuture`同时实现了`CompletionStage`和`Future`接口，因此在一系列操作之后，依然可以用Future的get()等方法来获得结果。

#### CompletionStage

注解摘录：

> 一个Stage可能就是一个异步的计算，它在等待另一个stage的action完成后，执行某个动作或者执行某个结果。一个stage在它的运算完毕时就认为结束了，但是它的结束可能会触发依赖它的其他stage开始运行。这个接口定义的方法其实只采用了一些基本的形式，但是这些基本形式的展开和组合有可能构成一些非常有用的编程模式。
>
> 一个Stage所执行的计算，可以是一个函数、一个消费者，或者一个Runnable对象。这取决于他们是否接收参数作为输入，以及是否有返回值。比如说，可能会写出这样的代码：
>
> ```java
> stage.thenApply(x -> square(x))
>   .thenAccept(x -> System.out.print(x))
>   .thenRun(() -> System.out.println());
> ```
>
> 大部分的方法都是对一个stage的运算结果起作用，但是只有`compose()`方法是作用于stage本身。即大部分方法可能都返回一个result，但compose()方法返回一个`CompleteableFuture`
>
> 在一些方法中，允许传入自定义的Executor(执行器，线程池的一个抽象)，如果传入了自定义的Executor，则stage的线程行为可能会被Executor改变，甚至完全不允许并行。在CompleteableFuture的实现中，这个Executor是`ForkJoinPool.commonPool()`
>
> 一个stage的开始可能会被一个stage的完成触发，或者需要两个stage都完成，或两个stage中的一个完成即可触发。依赖于一个stage的新stage可以用`then`开头的方法来触发，而依赖于两个stage的新stage，则根据需要使用用`runAfterBoth()`或者`runAfterEither()`方法，以及这些方法的变种来触发。这些函数都接受另一个Stage作为一参数，一个函数、一个消费者，或者一个Runnable对象作为二参数。
>
> 有几种途径为stage注册最终的回调函数，即当整个执行链执行完毕后应该做哪些工作。
>
> + `whenComplete()`
>
>   它接受一个回调函数action作为参数，当前stage完成时，执行这个action，返回一个新的Stage，这个新的stage中持有之前stage调用链的结果(和异常)。这个action接受上个stage的结果输出和异常。
>
>   如果action本身抛出了异常，那么假如之前的stage已有异常，那么最终的future持有的异常仍然是之前stage的异常，如果之前的stage没有异常，那么最终future持有的异常就是action抛出的异常。
>
>   它的声明如下：
>
>   ```java
>   public CompletionStage<T> whenComplete
>           (BiConsumer<? super T, ? super Throwable> action);
>   ```
>
>   action是一个consumer，没有返回值。
>
> + `handle()`
>
>   handle()就更加高大上一点，它接受的是一个有返回值的回调函数作为参数，返回一个新的stage，这个回调函数也接受上个stage的结果输出和异常。它可以返回一个新的result。
>
>   它的声明如下：
>
>   ```java
>   public <U> CompletionStage<U> handle
>           (BiFunction<? super T, Throwable, ? extends U> fn);
>   ```
>
>   注意其参数是一个BiFunction，可以有返回值，而`whenComplete()`的参数是一个BiConsumer，没有返回值。
>
> + `exceptionally()`
>
>   注册一个回调函数，其接受一个Throwable作为参数，返回一个stage，假如之前的运算出现异常，则该回调函数被调用，该回调函数可以返回一个与之前的stage同类型的result，如果之前的stage正常完成，那么这个回调函数不会被调用。
>
>   这个方法可以用于以下场景：假如某个工作正常完成，那么返回一个正常完成的值，如果出现异常，返回一个其他值。比如说：假如不出错，返回1，出异常了就调用回调函数，可能返回一个0。以使接下来的stage无论当前stage是否异常都可以正常运行。
>
> 对于上面这两个方法来说，无论前面的操作正常进行了还是出异常了，回调函数都会被调用，如果出异常了，那么result为null，如果正常结束，那么exception参数为null。
>
> 如果调用链中某一环发生异常，那么依赖于该环节结果的所有stage都会以异常状态结束，并且会返回一个CompletionException，将最先产生的异常作为cause，如果一个环节的完成是用both方法组合起来的，并且both的两个环节都异常了，那么将随机有一个cause，如果是用either组合起来的，且其中有一个环节异常了，那么不能保证最后有没有cause。
>
> 在运算完成之前，都可以调用`complete(T value)`提供一个值，使当前的stage立即结束，并返回提供的值。但如果stage已经正常结束，则不会使用这个值。

CompletionStage的每一个方法都有异步的重载版本，以及异步+自提供Executor的重载版本。每一个stage都有两种结束状态，正如线程有两种状态一样：正常结束、未捕获的异常结束。

`CompletionStage`有一系列重要的方法，如果忽略各种异步、提供Executor的重载版本，大概可以梳理出以下几个系列，在这些方法的注解中，我们称调用这些方法的stage为**上一个stage**。

+ `public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);`

  返回一个stage，该stage将在上个stage正常结束后运行，执行提供的fn，并且将上个stage的结果作为参数。

+ `public CompletionStage<Void> thenAccept(Consumer<? super T> action)`

  返回一个stage，当上一个stage正常结束后，执行提供的action，并且将上个stage的结果作为参数。

+ `public CompletionStage<Void> thenRun(Runnable action)`

  返回一个stage，当上一个stage正常结束后，执行提供的action。

+ `public <U,V> CompletionStage<V> thenCombine`

  ```java
  public <U,V> CompletionStage<V> thenCombine
          (CompletionStage<? extends U> other,
           BiFunction<? super T,? super U,? extends V> fn);
  ```

  返回一个stage，当上一个stage和other都正常结束时，把两个stage的结果作为参数传入fn中，调用fn。

+ `public <U> CompletionStage<Void> thenAcceptBoth`

  ```java
  public <U> CompletionStage<Void> thenAcceptBoth
          (CompletionStage<? extends U> other,
           BiConsumer<? super T, ? super U> action);
  ```

  返回一个stage，当上一个stage和other都正常结束时，把两个stage的结果作为参数传入action中。

+ `public CompletionStage<Void> runAfterBoth(CompletionStage<?> other,Runnable action);`

  返回一个stage，当上一个stage和other都正常结束时，运行action

+ `public <U> CompletionStage<U> applyToEither`

  ```java
  public <U> CompletionStage<U> applyToEither
          (CompletionStage<? extends T> other,
           Function<? super T, U> fn);
  ```

  返回一个stage，当上一个stage或other中任意一个正常结束时，以他的结果调用fn。要注意的是，这里的other stage返回值必须和上一个stage返回值一致。

+ `public CompletionStage<Void> acceptEither (CompletionStage<? extends T> other, consumer<? super T> action);`

  返回一个stage，当上一个stage或者other中任意一个正常结束时，以他的结果调用action。

+ `public CompletionStage<Void> runAfterEither(CompletionStage<?> other, Runnable action);`

  返回一个stage，当上一个stage或other中任意一个正常结束时，调用action。

+ `public <U> CompletionStage<U> thenCompose`

  ```java
  public <U> CompletionStage<U> thenCompose
          (Function<? super T, ? extends CompletionStage<U>> fn);
  ```

  其功能非常类似于`thenApply`，也是返回一个stage，当上一个stage完成时，以上一个stage的返回值作为参数调用fn，但该fn返回一个`CompletionStage`，也就是该方法的返回值。

  它和`thenApply`之间的区别，就好像stream的`map()`和`flatMap()`一样。

+ `public CompletionStage<T> exceptionally(Function<Throwable, ? extends T> fn);`

  已如上述。

+ `public CompletionStage<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);`

  已如上述。

+ `public <U> CompletionStage<U> handle((BiFunction<? super T, Throwable, ? extends U> fn))`

  已如上述。

+ `public CompletableFuture<T> toCompletableFuture();`

  就是返回和上一个stage一样的结果，如果上一个stage已经完成，可能就返回上一个stage，如果没完成，可能返回一个新的stage，但其持有的值还是一样的。其用途没搞清楚。

#### CompletableFuture

这是jdk包中对`CompletionStage`的唯一实现。正如stage一样，它可以组合各种同步或者异步任务，总之我们可以用它来串一串任务，并在最后直接用`whenComplete()`完成消费。或者把它像Future一样用，使用`get()`阻塞地获得最终结果。

关于`CompleteableFuture`的用法，可以见[CompletableFuture practical guide](https://medium.com/@Mumuksia/completablefuture-practical-guide-e4564f332f83)

简单地讲，一般都是用它的几个静态方法来构建一个`CompleteableFuture`，之后链式调用其方法来做后续操作。

```java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                       Executor executor);
public static CompletableFuture<Void> runAsync(Runnable runnable);
public static CompletableFuture<Void> runAsync(Runnable runnable,
                                                   Executor executor);
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs);
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs);
public static <U> CompletableFuture<U> completedFuture(U value)
```

这些方法是比较正常的构造一个CompleteFuture的办法，之后就可以愉快地调用链了。

`CompletableFuture`中有一些`CompletionStage`中没有的方法，包括：

+ `allOf(CompletableFuture<?>... cfs)`

  返回一个stage，该stage当所有参数中的cfs都完成后才返回。

+ `anyOf(CompletableFuture<?>... cfs)`

  返回一个stage，该stage当参数中提供的cfs有一个完成后就返回。

但有一些**Bad practice**，如下：

1. 不要搞一长串的调用链，写时一时爽，debug火葬场。
2. `get()`and`join()`方法最好不要调用，这些能够阻塞当前线程的调用，直到当前的`CompletableFuture`完成。

`CompletableFuture`因为实现了两个接口，而且有一些新的方法，因此在实例化时也经常直接实例化一个`CompletableFuture`，而不是`CompletionStage`

### 线程池

一个线程的生命周期包括创建线程、执行线程任务、销毁线程。记他们所需要的时间分别为T1、T2、T3。在很多时候，线程需要频繁处理一些生命周期非常短的任务，也就是说，T2的时间很短。在这种情况下，创建和销毁线程所需要的时间可能远远超过任务执行的时间。在这种情况下，使用线程池就更加合适。

线程池在Java中的抽象是`ExecutorService`，其顶级接口是`Executor`，但是真正的线程池接口还是`ExecutorService`，所以在代码里一般都直接用`ExecutorService`，写出这样的代码来：

```java
ExecutorService service = Executors.newFixedThreadPool(int nThreads);
```

一般来说，我们使用线程池调度算法的具体场景类似下面这段code

```java
class NetworkService implements Runnable {
  private final ServerSocket serverSocket;
  private final ExecutorService pool;

  public NetworkService(int port, int poolSize)
      throws IOException {
    serverSocket = new ServerSocket(port);
    pool = Executors.newFixedThreadPool(poolSize);
  }

  public void run() { // run the service
    try {
      for (;;) {
        pool.execute(new Handler(serverSocket.accept()));
      }
    } catch (IOException ex) {
      pool.shutdown();
    }
  }
}

class Handler implements Runnable {
  private final Socket socket;
  Handler(Socket socket) { this.socket = socket; }
  public void run() {
    // read and service request on socket
  }
}
```

这样就开始使用线程池了。那接下来详细观察一下线程池

#### ExecutorService

还是先来注解摘抄：

> 这是一个Eexcutor，提供专门管理Runable和Callable的方法。
>
> ExecutorService可以被`shutdown`，shutdown之后该executor就不在接受新的任务。有两种shutdown方法可以关闭一个executor，其中`shutdown()`就是简单地不再接受新的任务。而`shutdownNow()`则不仅不接受新的任务，而且正在等待的所有任务都会取消执行，正在执行中的任务都会尝试停止。当终止时，一个executor保证没有正在执行的任务，没有正在等待执行的任务，并且没有新的任务可以提交。
>
> 一个不用的executor必须被shutdown，这样它占用的资源才有可能被回收。
>
> Executor提供两个主要的方法来管理任务。一个是`void execute(Runnable runable)`，一个是`Future<T> submit(Callable<T> callable)`。
>
> 方法`invokeAny()`和`invokeAll()`提供了一种通用的管理任务的方法，即一次提交一大批任务，等待他们中有一个完成，或者都完成。`ExecutorCompletionService`可以用来自定义这两个方法。
>
> `Executors`是一个工厂类，用来产生各种合适的Executor。

一个ExectorService有以下方法：

```java
void shutdown();
List<Runnable> shutdownNow();			// 返回正在等待中的任务(该方法会取消正在等待任务的执行)
boolean isShutdown();
boolean isTerminated();
boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
void execute(Runnable command);		//该方法扩展于Executor接口
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);	// Runable结束后返回result
Future<?> submit(Runnable task);	// future的get()会阻塞到runnable执行完后，返回null

<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

<T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

<T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

<T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
```

具体不赘述。

#### Executors

这就是JDK提供的，方便生产线程池给我们用的工厂类。可以生产`Executor`， `ExecutorService`， `ScheduledExecutorService`、`ThreadFactory`和一些定义在`Executors`内部的`Callable`

其方法即注解如下(以下方法声明都忽略了`public static`修饰符)：

```java
/**
* 产生一个固定线程数量的线程池，当线程数量达到上限以后，新提交的任务将被放到一个无界阻塞队列中
* 当一个线程因为异常而死亡时，一个新的线程会创建来接替他的工作。
* 因为使用了无界队列，在消费者速度赶不上生产者速度时，任务会在队列中一直堆积，直到OOM
* 生产的Executor需要调用shutdown()显式关闭
* 实际产生一个ThreadPoolExecutor实例，使用的无界阻塞队列是LinkedBlockingQueue
*/
ExecutorService newFixedThreadPool(int nThreads);

/**
* 使用给定ThreadFactory生产线程。
*/
ExecutorService newFixedThreadPool(int nThreads, 
                                                 ThreadFactory threadFactory);

/**
* 产生一个维护足够数量的线程以支持参数指定并行级别的线程池，而且可能会使用多个队列来降低竞争。
* 该线程池的实际线程数量可能会动态调整，并且不保证任务执行顺序。
* 实际返回一个ForkJoinPool实例，使用的无界阻塞队列是LinkedBlockingQueue
*/
ExecutorService newWorkStealingPool(int parallelism);

/**
* 使用运行时可用处理器数量作为并行级别。
*/
ExecutorService newWorkStealingPool();

/**
* 生产一个只有一个线程的线程池，以串行执行任务。
* 返回一个ThreadPoolExecutor的装饰器
*/
ExecutorService newSingleThreadExecutor();
ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory);

/**
* 生产一个线程池，其线程数量上限是Integer.MAX_VALUE。一般没事别用这个线程池。
* 一个线程至少占用1M资源，无限创建线程将会OOM
* 返回一个ThreadPoolExecutor
*/
ExecutorService newCachedThreadPool();
ExecutorService newCachedThreadPool(ThreadFactory threadFactory);

/**
* 产生一个执行定期任务的线程池。以执行定期任务或者在一定delay后执行任务
* corePoolSize是指线程池中最小线程数量，即使他们处于空闲状态
* 返回一个ScheduledThreadPoolExecutor
*/
ScheduledExecutorService newScheduledThreadPool(int corePoolSize);
ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory)

/**
* 产生只有一个线程的定期任务线程池。
* 返回一个ScheduledThreadPoolExecutor的包装器。
*/
ScheduledExecutorService newSingleThreadScheduledExecutor();
ScheduledExecutorService 
						newSingleThreadScheduledExecutor(ThreadFactory threadFactory);

/**
* 返回一个Executor的装饰器，以防止用户通过将ExecutorService Cast成为具体类型
* 之后再修改线程池的配置
*/
ExecutorService unconfigurableExecutorService(ExecutorService executor);
ScheduledExecutorService unconfigurableScheduledExecutorService(
  					ScheduledExecutorService executor);

/**
* 返回一个默认的线程工厂。
* 实际上该接口只有一个方法，Thread newThread(Runnable r);
*/
ThreadFactory defaultThreadFactory();

/**
* 返回一个线程工厂，该线程工厂除了做defalutThreadFactory的工作以外，还额外地将新线程
* 的访问控制上下文和contextClassLoader上下文设为和调用该privilegedThreadFactory的
* 线程的相关内容一致。
*/
ThreadFactory privilegedThreadFactory();
```

#### ThreadPoolExecutor

一个ExecutorService，它使用自己维护的多个线称中的一个来完成提交到它的任务。

线程池解决了两个问题，一个是它通常能够通过降低单任务的施法前摇和后摇来缩短一个任务的执行时间，另一个是它可以协助管理执行一组任务所需要的资源，包括线程本身占用的资源等等。

在大部分时间里，我们都可以使用`Executors`这个工厂类来创建各种各样的`ThreadPoolExecutor`，但如这篇文章所述：[一个线程池误用引发的血案](https://zhuanlan.zhihu.com/p/32867181)，该工厂产生的线程池使用的都是无界队列来存放一时无法处理的任务，当任务的生产速度超过消费速度就会引起任务积压，最终导致OOM崩溃，而不限制线程数量的线程池，其默认线程数量上限是`Integer.MAX_VALUE`，在上述消费者能力不够的情况下会引起无限开线程，线程本身需要的资源导致OOM。

因此，包括阿里巴巴JAVA开发手册在内，一些JAVA语言的指导都要求不要使用`Executors`工厂类创建的线程池，而自己去创建线程池。ThreadPoolExecutor的注释表明，该类确实提供了极大的灵活性以自定义线程池的方方面面，如果确实需要自定义这些配置的话，需要遵守以下指导规范：

+ 核心线程数量和最大线程数量(Core and maximum pool sizes)

  一个`ThreadPoolExecutor`线程池将根据核心线程数量和最大线程数量自动调整线程池大小。当一个新任务提交时，如果此时正在运行的线程比corePoolSize数量小，那么线程池将会创建一个新线程来处理这个任务，即使现在还有线程处于空闲状态。如果正在运行的线程数量大于corePoolSize但是小于maximumPoolSize，任务将会先暂存到任务队列里，除非任务队列也满了，这时候才会新建一个线程。

  因此，通过调整核心线程数量和最大线程数量就可以控制线程的数量。一般来说，这两个参数实在新建时就创建好的，但仍然可以在运行时进行修改。(Executors中提供了方法可以禁止这种创建完毕后的修改。)

+ 创建时构造线程池

  一般来说，一个线程池创建以后并不新建线程，直到任务到来。但可以调用方法`prestartCoreThread`或者`prestartAllCoreThreads`要求线程池在初始化时就新建足够数量的线程。如果你在执行一个线程时已经有一个非空的任务队列，可以使用这个特性。

+ 创建新的线程

  新线程都是通过`ThreadFactory`创建的，如果没有特别指定，那么`ThreadFactory`将使用`Executors.DefaultThreadFactory`，该`ThreadFactory`创建的线程都属于一个线程组(一个已经不建议使用的概念)，并且都有相同的优先级，也就是默认优先级(Normal_Priority)，并且是非守护线程。通过提供自定义的`ThreadFactory`，用户可以自定义线程名、线程组、优先级、守护状态等等。如果`ThreadFactory`创建线程失败，返回了null，线程池并不会崩溃，但可能无法处理任何任务了。

  线程必须拥有modifyThread这个权限，如果没有该权限，那么executor可能会降级，配置更改可能不会生效，或者线程池能停留在一个终止了但没有完全清除的状态。

+ 存活时间Keep-alive times

  如果一个线程池当前的线程数超过了corePoolSize，超出的线程将会在空闲超过存活时间后被终止。这个参数也可以在运行时动态设置，如果设置成一个非常长的时间，实际效果就是这些线程将永远不会被终止。一般来说，存活时间仅对超过了corePoolSize的线程起作用，但可以通过`allowCoreThreadTimeOut(Boolean)`方法使超时策略也应用于核心线程。

+ 阻塞队列

  阻塞队列处理的问题是任务如何被提交给线程。阻塞队列的使用与线程池大小有一定关系。

  + 如果当前运行的线程数量少于corePoolSize，那么Executor将总是新建线程来完成任务，此时阻塞队列并没有用处。
  + 如果当前运行的线程数量大于或者等于corePoolSize，那么Executor将把线程放到队列中去，而不是新建线程。即使线程数量还没有达到maximumPoolSize。
  + 如果一个请求无法放入队列，即队列已满，且线程数量还没有达到maximumPoolSize，新线程才会被创建。如果队列已满、线程数量已到最大，那么新任务就会被拒绝。至于拒绝策略，在之后需要提供。

  有三种普遍的队列策略：

  + 直接hand to hand交接。

    比如使用SynchronousQueue，必须有可用线程在等待时任务交付才能够完成，否则就会新建一个线程来这个任务。这个方式避免了任务集中的任务有内部依赖时可能产生的锁定(This policy avoids lockups when handling sets of requests that might have internal dependencies.)，这种队列一般需要无上限的maximumPoolSize，因此可能造成线程池膨胀造成的OOM。

  + 无界阻塞队列。

    比如使用LinkedBlockingQueue，这种策略在corePoolSize数量的线程都在忙时，新任务会一直往队列里提交，也就是说，maximumPoolSize设置会实际上无效，因为永远不会有超过corePoolSize的线程创建。这种策略一般适用于任务之间独立、没有相互依赖的情况，比如一个web服务器。但是这种队列因为其没有容量上限，可能导致队列膨胀造成的OOM。

  + 有界队列

    这种队列能够防止占用资源无限膨胀，但可能更难控制。队列大小和最大线程数量将会相互影响，使用大队列和小线程池，可能会节省CPU和系统资源，但降低了吞吐量。如果使用小队列一般需要更大的线程池，这可能保持CPU的高使用率，但可能遭遇无法接受的上下文切换消耗，最终也降低了吞吐量。

+ 拒绝策略

  当线程池被`shutdown()`或者上述线程池处理能力满了的时候，新提交的任务就会被拒绝。无论是哪种情况引发的拒绝，`RejectedExecutionHandler.rejectedExecution(Runnable, ThreadPoolExecutor)`都将被调用，JDK提供了以下四个预定义的拒绝策略：

  + `ThreadPoolExecutor.AbortPolicy`

    抛出一个运行时异常：RejectedExecutionException

  + `ThreadPoolExecutor.CallerRunsPolicy`

    这个策略将直接在调用execute的线程下执行提交的task。因为生产者线程会因为执行这个task被阻塞，不能提交新的任务，可以延缓新task的产生。

  + `ThreadPoolExecutor.DiscardPolicy`

    直接丢弃

  + `ThreadPoolExecutor.DiscardOldestPolicy`

    丢弃队列首位的任务。如果使用了优先级队列，将导致优先级最高的任务被丢弃，不建议搭配优先级队列使用。

  也可以定义自己的拒绝策略，如果你要这么玩儿，就需要格外小心，特别是使用有界队列的时候。

+ 钩子方法

  可以继承并覆写该类的`beforeExecute(Thread, Runnable)`方法和`afterExecute(Runnable, Throwable)`，这两个方法，在任务执行前或者执行后执行某些制定操作，比如说重新初始化线程变量、收集统计数据、或者增加日志等（策略模式，aop思想），也可以覆写`terminated`方法，要求executor在完全终止时执行任何自定义的操作。

  如果钩子或者回调参数发生了异常，那么该线程可能连续失败。

+ 队列维护

  `getQueue()`方法允许用户访问工作中的队列，为监控或者debug提供支持。因为其他原因使用该方法都是强烈不建议的。`remove(Runnable)`和`purge()`方法可以用来在大量队列任务取消的时候，协助处理内存回收。

+ Finalization

  一个不再被引用且不再保有线程的线程池将会被自动shutdown，如果要确保不被引用的线程池锁占用的资源在用户忘记调用shutdown的情况下仍然能够被回收，那就必须设置合理的超时时间，并且设置核心线程数量为0或者打开allowCoreThreadTimeOut。

#### ScheduledThreadPoolExecutor

一个`ScheduledThreadPoolExecutor`支持定时执行任务，包括在给定延迟之后执行任务，以及周期性执行任务。

延迟任务（delayed tasks）将在正式启动之后运行，但启动后具体执行的时间将没有保证。在同一时间被启动的任务将会严格按照FIFO原则执行。当一个任务在执行前被取消，那么执行将会停止，但是该任务并不会被自动移出队列，直到超时，这么做是为了方便做监控。但这样就可能造成被取消的任务积压(如果设置的时间太长)导致OOM，为了避免这种情况，可以将`setRemoveOnCancelPolicy`设为`true`，这样他们被取消时就会立即移出队列。

虽然该类继承于`ThreadPoolExecutor`，但一些参数调整对该类并没有什么效果。一方面，这个类总是一个固定线程数量(corePoolSize)的线程池，并且使用无界队列，因此设置`maximumPoolSize`对该类无用。并且，在任何时候都不要设置corePoolSize为0，或者`allowCoreThreadTimeOut`，因为这可能会导致该线程池在真正需要执行任务时没有线程可用。

该类的主要方法继承自接口`ScheduledExecutorService`，对于该接口，有一些重要注解摘抄如下：

`schedule`方法允许创建带有多样化延迟的任务，并且返回一个task的object，以便用来取消任务执行以及检查执行情况。`scheduleAtFixedRate`和`scheduleWithFixedDelay`方法创建并执行一个定期执行的任务，直到被取消。

而`execute()`和`submit()`会被当做是一个延迟为0或者负的定时任务，也就是他们会立即执行。

所有的`schedule`方法都接受相对时间或者时间段作为参数，而不是绝对时间或者日期。

一个简单的代码示例，在接下来一小时里每十秒输出一个beep，直到一小时后取消。

```java
import static java.util.concurrent.TimeUnit.*;
class BeeperControl {
  private final ScheduledExecutorService scheduler =
    Executors.newScheduledThreadPool(1);

  public void beepForAnHour() {
    final Runnable beeper = new Runnable() {
      public void run() { System.out.println("beep"); }
    };
    final ScheduledFuture<?> beeperHandle =
      scheduler.scheduleAtFixedRate(beeper, 10, 10, SECONDS);
    scheduler.schedule(new Runnable() {
      public void run() { beeperHandle.cancel(true); }
    }, 60 * 60, SECONDS);
  }
}
```

接口的主要方法有：

```java
public ScheduledFuture<?> schedule(Runnable command,
                                       long delay, TimeUnit unit);

public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay, TimeUnit unit);

public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit);

public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit);
```

非常简单。需要注意的是。`ScheduledThreadPoolExecutor`实际上覆写了`submit`和`execute`方法，将他们作为延迟为0的任务提交给`schedule()`执行了，因此返回值实际上是`ScheduledFuture`。在覆写该类时应该注意到这一点。

#### ForkJoinPool

顾名思义，用来执行`ForkJoinTask`的线程池。但也提供了执行非`ForkJoinTask`的接口。

ForkJoinPool与其他线程池最大的不同在于，它应用了**任务窃取**技术。任务窃取技术的细节在上面的`Fork/Join框架`那个部分已经讲过了。不过这里提到了一句话，即

>ForkJoinPools may also be appropriate for use with event-style tasks that are never joined.
>
>对于那些不需要join的事件型的任务也非常合适。即并不关心结果，只求把任务分解后跑完。只fork而并不join。

在大部分场合下，使用静态方法`commonPool()`就足够了。任何没有被显式提交到特定线程池的ForkJoinTask都使用common pool。使用该pool通常会节省资源，因为它会缓慢回收无用线程，并且再使用时再恢复。

如果程序真的需要自定义线程池，那么可以在构造`ForkJoinPool`时传入指定的并行级别(parallelism level)，默认情况下等于可用处理器数量。该线程池会动态维护足够的线程来完成任务。

除了执行任务和生命周期管理，该类还提供了状态检查方法，比如`getStealCount()`，旨在帮助调优和监控fork/join框架的运行情况。同时，`toString()`方法以特定的格式输出监控信息。

对于其他的`ExecutorService`来说，主要通过`execute()`、`submit()`、`invoke()`来提交任务，`ScheduleExecutorService`主要通过`sheduleXXX()`系列方法来提交定时任务。对于ForkJoinPool来说，仍然可以通过以上三个主要方法来提交`ForkJoinTask`，但除非你的任务是那种事件型不需要或很少需要join的任务，不然还是推荐使用`ForkJoinStyle`的写法。即通过`ForkJoinTask.fork()`和`ForkJoinTask.invoke()`来替代，演示虽然上面已经有例子了，还是再来一次：

```java
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ForkJoinTask extends RecursiveTask<Long> {

    private static final long MAX = 1000000000L;
    private static final long THRESHOLD = 1000L;
    private long start;
    private long end;

    public ForkJoinTask(long start, long end) {
        this.start = start;
        this.end = end;
    }

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        ForkJoinPool forkJoinPool = new ForkJoinPool();
        Long sum = forkJoinPool.invoke(new ForkJoinTask(1, MAX));
        System.out.println(sum);
        System.out.println(System.currentTimeMillis() - start + "ms");
    }

    @Override
  	// ForkJoinTask主要是覆盖这个方法
    protected Long compute() {
        long sum = 0;
        if (end - start <= THRESHOLD) {
            for (long i = start; i <= end; i++) {
                sum += i;
            }
            return sum;
        } else {
            long mid = (start + end) / 2;

            ForkJoinTask task1 = new ForkJoinTask(start, mid);
            task1.fork();

            ForkJoinTask task2 = new ForkJoinTask(mid + 1, end);
            task2.fork();

            return task1.join() + task2.join();
        }
    }
  
}
```

commonPool一般是通过默认参数构造的，但是如果确有需要调整，也可以通过以下几个参数来调整：

```java
public ForkJoinPool(int parallelism,
                    ForkJoinPool.ForkJoinWorkerThreadFactory factory,
                    Thread.UncaughtExceptionHandler handler,
                    boolean asyncMode);
// 最后一个参数设为true，可能更方便执行那些事件型不需要或极少需要join的任务。
```

该线程池的主要特定方法包括：

```java
/**
* 执行指定任务直到完成，如果执行出现异常，异常会被抛出。
*/
public <T> T invoke(ForkJoinTask<T> task);
```

##### 进一步看看ForkJoinTask

为执行在`ForkJoinPool`中的任务设计的抽象基类。一个`ForkJoinTask`是一个类似线程(Thread-like)的实体，但比普通的线程要轻量地多。大量的任务和子任务能够被少量的线程处理，只是应用上稍微受限一点。

一个`ForkJoinTask`从显式提交给`ForkJoinPool`时正式运行，或者还没有与某个`ForkJoinPool`关联时，调用`ForkJoinTask.fork()`、`invoke()`方法，会把任务提交给`ForkJoinPool.Commonpool`。一旦开始，一个`Task`通常会产生一些subtask。虽然实际上大部分用户使用`ForkJoinTask`，都只会应用`fork()`，`join()`这两个方法，不过该类还是为高级玩家预留了一下方法。

一个`ForkJoinTask`是一个轻量级的`Future`，`ForkJoinTask`的高效来源于这样的假设：即他们的主要用途是执行纯粹的函数或者执行一些相互之间没有联系的任务。`ForkJoinTask`的主要的同步工具是`fork()`方法，这将触发一次异步执行，而`join()`动作将阻塞直到具体的计算任务完成。计算任务最好**避免**使用`synchronized`方法或`synchronized`块和大部分同步原语，除了`join`其他任务，或者`Phasers`这样推荐和`Fork/Join`框架合作的同步原语。**子任务应当不要执行阻塞IO**，并且最好只访问与其他线程完全隔离的变量。这些指导原则靠不允许抛出受检查异常如IOException来松散地保证。但用户任务仍然可能抛出非检查异常，那么该异常将会抛到尝试join这些发生异常的任务的线程里。除此之外，当资源耗尽的时候，比如无法为任务分配队列空间，有可能会抛出`RejectedExecutionException`。重新抛出的异常和普通的异常是一样的，只是在可能的时候会包含两个线程的异常栈信息：初始化这个任务的线程和真正遇到这个异常的线程，最少有后者的异常栈信息。

除非满足下列条件，否则ForkJoinTask不应该执行阻塞任务：

+ 其他任务应该不依赖(如果非要依赖，也尽量少)那些需要外部synchronization同步控制或者IO的任务。那些事件型、从来不join的异步任务就很符合这个要求；
+ 最小化资源争用。任务应当足够小，最好这个阻塞任务只执行这个阻塞任务。
+ 除非用过了`ForkJoinPool.ManagedBlocker`接口，或者有可能阻塞的任务小于ForkJoinPool的并行级别，不然线程池无法保证有足够的线程完成工作，或者保证性能。

让当前线程等待另一个任务完成并得到结果的主要方法是`join()`，但因为`ForkJoinTask()`本身实现了`Future`接口，因此也可以用`Future.get()`这种Future风格的方法来等待和获取结果。`invoke()`方法在语义上和`fork()`方法是一致的，`join()`方法则总是尝试在当前线程执行。这些方法的`quiet`变体不会产生结果，也不会报告异常，这些变体在一系列任务需要执行，但你需要在所有任务都完成后再处理结果或异常的场景下比较有用。`invokeAll()`方法就更牛逼了，它一口气fork出一系列任务，然后等待他们全部完成(join them).

在大多数情况下，一个ForkJoin任务执行就像一个递归函数中的调用(fork)/返回(join)。在大部分情况下，fork/join的顺序也像递归那样安排，比如`a.fork(); b.fork(); b.join(); a.join();`这种形式，就比a先join效率可能高一点。

有多种方式可以查看任务的执行状态，`isDone()`在任务完成，包括任务被取消的情况下返回true，`isCompleteNormally()`在任务正常完成且没有被取消、没有抛异常的情况下返回true，`isCancelled()`在任务取消时返回true，而`isCompletedAbnormally()`在任务取消或任务遇到异常时返回true。在任务取消时，`getException()`方法将会得到`CancellationException`，在任务遇到异常时，会得到具体异常。

该接口通常不会直接被实现，如果用户需要实现该接口，一般是实现它的一些抽象子类，这些抽箱子类被设计为执行不同风格的`Fork/Join`任务，特别是`RecursiveAction`，适合大多数不返回结果的任务，而`RecursiveTask`适合返回结果的任务，而`CountedCompleter`是为那些链式调用，即一个完成的任务trigger另一个完成的任务这种操作准备的。一般情况下，就是扩展其中一个，然后覆盖相应类中的`compute()`方法即可。

`ForkJoinTask`应当执行相对较小的任务，较大的任务应当被分割为更小的任务后再执行，一个粗糙的参考标准是，一个子任务应当进行大于100并且小于10000次运算，并且应当避免模糊的循环次数。如果一个任务太大了，那并行就无法提高吞吐量，如果太小了，那么调度的开销可能会超过拆分带来的好处。

`ForkJoinTasks`实现了`Serializable`接口，因此允许类似远程调用的技术。但应当在任务执行之前或之后进行序列化，而非正在执行时。

### 小结

多线程估计是程序员进阶路上绕不过的一个绊子。多线程本身的复杂性，再加上现代编译器和CPU的很多透明优化，将会造成很多很难排查、很难复现的问题。

使用线程，首先需要了解线程的一些基本概念，包括线程的六种状态、线程的终止和中断。特别是了解到线程的中断实际上只是在线程的中断标志位置位，至于如何处理这个中断标志，由线程自己决定。因此，线程应该总是在合适的位置检查中断标志位，以决定如何处理中断。在线程阻塞的情况下，比如`sleep()`或者`wait()`时，因为线程阻塞无法到达检查中断标志位的代码，此时将该线程中断标志置位将导致该线程抛出`InterruptedException`。

java从早期版本就支持`synchronized`同步关键字，并将`wait()`，`notify()`和`notifyAll()`设计到

每一个object中，从而使每一个object都可以作为锁使用，最早的`synchronized`关键字将会直接使用一个重量级锁，即monitor锁，与此相关的概念主要是入口集、等待集和锁owner。明白这些概念，将很容易理解`wait()`和`notify()`以及`notifyAll()`的具体效果，并且明白为什么`notify()`可能导致死锁。因为早期的`synchronized`使用了重量级锁，这也造成了大家只要使用该关键字就会造成性能较大损失的印象。

早期的`synchronized`提供的功能在当时的硬件条件下还是可以的，我个人理解，可能当时的硬件并没有普遍的多核，因此该关键字的主要功能实际上是在服务因为IO等阻塞后不影响其他任务的调度，即多线程主要用来处理异步，而并非并发，对性能的要求也没有这么极端。但随着硬件的发展，真正的多核时代到来了，更多的应用程序需要用到多核的硬件，此时使用java提供的`sychronized`低级同步原语十分困难且容易出错，java顺应程序员的呼声，推出了一系列高级同步工具，并且大幅度优化了`sychronized`的锁机制。

+ 现在的`synchronized`锁并不是一开始就是重量级锁，而是综合运用了偏向锁、轻量级锁、重量级锁逐步升级的技术，尽量避免锁性能浪费，同时采取了自旋锁，在部分情况下使用CAS乐观锁，在CPU忙循环消耗和线程上下文切换消耗中试图做出更优化的选择。并且使用了编译期的锁消除、锁粗化机制，尽量避免无谓的加锁解锁消耗。

+ 提供了`Lock and Condition`这种更高级的锁机制，他可以支持不阻塞地尝试加锁(失败时立即返回false)，还可以在其他代码块进行解锁，也不像`synchronized`关键字这样必须严格按照加锁倒序进行解锁。这就使得一些链式加锁（先获得A的锁，再获得B的锁，再释放A的锁，再获得C的锁，再释放B的锁）成为可能。

  同时，还提供了读写锁这种高级锁机制，以提高在大容量容器上、执行读操作远远多于写操作场景下锁的性能。该锁允许多个线程同时获取读锁，但当有线程获得写锁时，其他任何线程都不能获得读锁或者写锁。

  同时，这种锁还提供了Condition机制，以允许线程在获得锁后，因为其他资源未就绪(执行条件不满足)而放弃锁，给其他线程以准备资源的机会。

+ 为了方便程序员写出线程安全的代码，提供了很多内部使用了锁的工具、对外暴露简单接口的阻塞队列，以简化`生产者/消费者`模型的开发难度。这些队列根据是否有界、是否先进先出、是否双端可取分为很多种。并作为基础工具大量应用于更高级的线程工具中，比如Fork/Join框架。
+ 为了使程序员能够更灵活地控制线程同步，还推出了更高级的同步器，可以支持让主线程等待N个子线程完成工作（CountDownLatch），或者允许线程阶段性地在某处同步(Barrier、Phaser)。
+ 在Runnable对象之外，还提供了Callable与Future，来简化需要有返回值的任务多线程执行。为了能够让Callable跑在原来为Runnable设计的Thread类中，为Callable设计了一个适配器，将Callable转化为Future和Runnable，这就是FutureTask类。

+ JDK同时提供了一系列线程安全的集合，保证在多线程环境下操作这些集合不会破坏他们的内部结构，导致循环或者其他问题。但是要注意的是，这些集合本身只能保证在多线程环境下内部结构是安全的，而不能保证操作是原子的。即这样的代码`map.put(key,map.get(key)+1);`在多线程下操作会造成不可预知的结果，为保证类似操作的安全，应该手写CAS，或者利用这些线程安全集合的一些高级方法，以保证这类操作原子性。
+ JDK提供了一些原子类，即`AtomicXXX`系列类。这些类在底层使用了CAS机制，保证多线程下对他们的操作是安全的，结果是可预期的。同时，为了避免这些类在高度竞争的情况下出现性能问题，JDK还提供了`LongAdder`，`DoubleAdder`、`LongAccumulator`、`DoubleAccumulator`类，通过将数字的存储在内部分散到多个原子类上，而其总和表示最终数字的方式，减少了竞争带来的性能损耗。在多线程环境下，应该根据合适的环境选择合适的原子类，避免自己加锁。

同时，为了减少线程创建和销毁的开销，JDK提供了线程池的概念，线程池的顶级接口是`Executor`，但是实际上都会用`ExecutorService`、`ScheduledExecutorService`、`ForkJoinPool`作为真正是用的接口，他们具体的实现类包括`ThreadPoolExecutor`、 `ScheduledThreadPoolExecutor` 、 `ForkJoinPool`。这些线程都有自己的调度策略，有自己的专属方法，也有自己的一些特殊要求，比如`ForkJoinPool`执行的任务最好是完全独立，且不阻塞的。但这些线程池都需要解决这样的一些问题：

+ 线程池是否在创建时就创建一定数量的线程，还是在任务到来后动态创建。动态创建的策略是什么，线程池里的线程是否允许回收，回收策略是什么。这些策略与`corePoolSize`， `maximumPoolSize`，`keepAliveTime`以及 `allowCoreThreadTimeOut`这几个参数息息相关。
+ 线程池在任务不断到来，但可用线程都在工作时，到底采取什么策略来新建线程，或者暂存来不及执行的任务。这根据线程池用来保存任务的阻塞队列是什么，以及maximumPoolSize的选择息息相关。在生产者速度超过消费者速度的情况下，因为这个策略的不同，会使线程池的行为走向两个极端，一个是使用无界队列一直保存来不及执行的任务，导致任务OOM，一个是不断新建线程试图同时运行所有任务，导致线程资源OOM。因此，在生产环境下监控消费者能力不足、任务积压的情况十分必要。

这些线程池一般不用自己手动创建，JDK提供了`Executors`这个工厂类来产生线程，在大部分情况下，工厂生产的默认线程池就够用了。但主要需要提防线程生产者能力超过消费者能力这种情况，在这种情况下可能需要提供有界队列，并提供拒绝接受新任务时的处理器（handler）。

除了这些仍然需要和线程这个概念直接打交道的机制以外，JDK还提供了更高级别的线程抽象机制，即将线程的概念都隐藏起来，只暴露出执行任务的抽象接口，进一步剥离线程管理的复杂度。这些工具包括`CompleteableFuture`、`Fork/Join`框架。

+ `CompletableFuture`适合用来处理一系列有先后关系的任务，即特别适合描述这样的编程场景：一个异步任务完成后，我们执行另外一个异步任务，再执行另外一个异步任务，或者等待两个异步任务都完成后，用他们的结果执行一些任务等等，并且能妥善处理任意一步任务出现的异常。`CompletableFuture`同时实现了`CompletionStage`和`Future`接口，提供多达50多个方法，支持各种形式的异步任务调用链。可以极大简化上述场景的编程任务。
+ `Fork/Join`框架则适合处理那些可以分拆成相互独立的子任务的大型任务，以充分利用多核处理器的效能，同时它还通过**任务窃取**技术，保证所有线程都能保持较高负载率，而避免出现一个线程阻塞后无所事事的情况。为了充分利用Fork/Join框架的能力，应当为其提供合理的并行级别，而且应当尽力保证提交给它的任务是不阻塞的，而且相互之间没有依赖关系。

但是，万剑归宗。无论是使用底层的低级同步原语，还是使用更加高级的线程同步工具，都必须时刻记住线程安全这个大话题。要确保线程安全，需要对原子性、有序性与可见性有基本的了解，对现代CPU调度、多级缓存有一定的概念。无论如何，要记得无锁区的任意代码都有可能随时被中断，使用类似set方法时要格外留意。

本文求全而不求深，只求对Java多线程下的大部分工具有一个基本的了解，以便在使用时知道要去哪里寻找。

谢谢耐心看到这里。