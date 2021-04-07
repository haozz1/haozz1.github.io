---
title: Java的IO进化简史
date: 2020-02-07 12:32:25
tags: Java
---



-- 你有《时间简史》吗？

-- 你有病吧，我有时间也不去捡屎啊。

好吧，上面是个冷笑话。

<!--more-->

## 问题的引出

前一段时间看Java的内存模型，里面提到了本地内存，以及经常会使用本地内存的一种由`JDK1.4`引入的新IO方式，`NIO`，虽然官方称之为`New IO`，但是实际上因为其最重要的特性是非阻塞，因此常被称为`Non-Blocking IO`，即非阻塞式IO，它的出现解决了传统的`Blocking IO`带来的一些性能问题。其具体的优化细节和解决的痛点我们之后再详细讨论。

然后我就陷入`NIO`的坑里了，经过一段时间的整理，我发现`JDK1.7`以后，又引入了一种新的IO类型，`NIO2.0`，又称为`AIO`，意思是`Asynchronous IO`，即异步IO。能推出异步IO肯定有他的道理，但出现了这个概念以后，我就想认真梳理一下这些IO有什么不同，以及到底解决了什么问题。

于是首先我陷入了一个新手可能都会绕进去的概念里面：

## 同步、异步 | 阻塞、非阻塞

这两个可以排列组合的，所以将可能排列出同步阻塞、同步非阻塞、异步阻塞、异步非阻塞，这四种方式，特别是其中的异步阻塞最为诡异和难以理解。我一开始确实绕进去了，但后来还是根据上面知乎回答搞明白了这几个概念。

其中最核心的区别是：

同步、异步是针对调用者而言的，一个同步调用在**被调用者完成其任务前**不会返回，而一个异步调用，在**向被调用者提交完任务后**就返回了，被调用者通过通知来告知调用者其任务已经执行完毕，或者直接执行调用者提供的回调函数。

而阻塞、非阻塞关注的是程序在等待返回值时在干什么。阻塞是指当前的任务会被挂起，一直到结果返回，而非阻塞是指**即使调用没有立即完成**，程序在等待完成的过程中可以干点其他的事情。

这个还是不好理解的话，知乎上举的例子还是很简单易懂的。

> 老张爱喝茶，废话不说，煮开水。
> 出场人物：老张，水壶两把（普通水壶，简称水壶；会响的水壶，简称响水壶）。
> 1 老张把水壶放到火上，立等水开。（同步阻塞）
> 老张觉得自己有点傻
> 2 老张把水壶放到火上，去客厅看电视，时不时去厨房看看水开没有。（同步非阻塞）
> 老张还是觉得自己有点傻，于是变高端了，买了把会响笛的那种水壶。水开之后，能大声发出嘀~~~~的噪音。
> 3 老张把响水壶放到火上，立等水开。（异步阻塞）
> 老张觉得这样傻等意义不大
> 4 老张把响水壶放到火上，去客厅看电视，水壶响之前不再去看它了，响了再去拿壶。（异步非阻塞）
> 老张觉得自己聪明了。

其中最诡异的是异步阻塞，其行为很反逻辑，所以会让人想不明白，因为我都异步了，为什么要阻塞在这里等待？但其实这个在某些场合下也有用处。比如说：在数据库主从同步时，如果要求强一致性，那么在返回之前就必须保证所有的从数据库的消息都返回，那我是不是一个个调用从数据的阻塞接口，写完一个再写下一个？或者提交任务后一遍遍去问他们都做完了没有？显然更好的方式是我一口气给他们每一个都发送一个异步任务，然后阻塞着等他们都完成后通知我，我再进行下一步。其他我实在想不到应用场景……总之比较诡异。

上面老张的例子是个好理解的例子而已，而且上面所说的异步非阻塞的模型是**异步+通知**的模型，实际上还有个**异步+回调**的模型，如果非要用烧水煮茶的例子类比，就是这个水壶很智能，你可以告诉他水烧好后自己泡茶，然后你就彻底不用管了。

以上的四种模型，如果用代码语言去表达，大概会是什么样子呢？我试着举出下面的例子，如有错误，请自行甄别：

+ 同步阻塞

  ```java
  int a = sum(new int[]{1,2,3,4});		// 这就是个典型的同步阻塞调用
  public int sum(int[] input){
    	// 忽略判null
     	int ret = 0;
    	for(int i: input){
        	ret += i;
      }
    	return ret;
  }
  ```

+ 同步非阻塞

  ```java
  public class SyncBlock {
  
      static class SumTask implements Callable<Integer> {
  
          private int[] input;
  
          public SumTask(int[] input) {
              this.input = input;
          }
  
          @Override
          public Integer call() throws Exception {
              int ret = 0;
              for (int i : input) {
                  ret += i;
              }
              return ret;
          }
      }
  
      public static void main(String[] args) throws InterruptedException, ExecutionException {
  
          Callable<Integer> sumTask = new SumTask(new int[]{1, 2, 3, 4});
          FutureTask<Integer> futureTask = new FutureTask<>(sumTask);
          new Thread(futureTask).start();
          while (!futureTask.isDone()) {
              // 下面这行输出是模拟我在等待结果的过程中做其他事情
              // 同步的意思是，futureTask.isDone()在返回时该方法的任务已经完成了
              // 毕竟他的任务只是判断任务是不是已经结束。
              // 又因为调用者在等待结果时可以做其他事，所以称之为非阻塞
              System.out.println("我买了好破一台电脑，四个数都加不出来");
          }
          System.out.println(futureTask.get());
      }
  }
  ```
  
+ 异步非阻塞

  先来个回调方式，这样实际上调用线程什么都不用做了，只要提供一个合理的回调方法就行。

  ```java
  public class AsyncNonBlockingWithCallback {
  
      static class SumTask implements Runnable {
  
          private int[] input;
          Consumer<Integer> callback;
  
          SumTask(int[] input, Consumer<Integer> callBack) {
              this.input = input;
              this.callback = callBack;
          }
  
          @Override
          public void run() {
              int ret = 0;
              for (int i : input) {
                  ret += i;
              }
              callback.accept(ret);
          }
  
      }
  
      public static void main(String[] args) throws IOException {
  
          Consumer<Integer> callback = (result) -> System.out.println(result);
          Runnable sumTask = new SumTask(new int[]{1, 2, 3, 4}, callback);
          new Thread(sumTask).start();
          System.out.println("OH YEAH，任务甩给那个线程啦，我现在想干啥都行了，干完以后他自己处理回调就好了");
  
      }
      
  }
  ```
  
通知方式，即当异步任务完成后，会去通知调用线程。这是上面老王煮茶那个例子了。
  
```java
  public class AsyncNonBlockingWithNotify {
  
      static class SumTask implements Callable<Integer> {
  
          private int[] input;
          private Thread caller;
  
          SumTask(int[] input, Thread caller) {
              this.input = input;
              this.caller = caller;
          }
  
          @Override
          public Integer call() {
              int ret = 0;
              for (int i : input) {
                  ret += i;
              }
              caller.interrupt();
              // 这里隐藏了一个很微妙的线程同步问题，即这个interrupt和后面的返回并不是原子的
            	// 万一打断后，该线程失去了CPU使用权，没返回，那边会在get上阻塞一下，虽然不是大问题
              return ret;
          }
  
      }
  
      public static void main(String[] args) throws InterruptedException, ExecutionException {
  
          SumTask sumTask = new SumTask(new int[]{1, 2, 3, 4}, Thread.currentThread());
          FutureTask<Integer> futureTask = new FutureTask<>(sumTask);
          new Thread(futureTask).start();
          while (true) {
              // 我愉快地干自己的事情啦
            	// 区别于上一次我一直去轮询Future是不是完成了
              // 这里其实是异步任务(通过打断线程)来通知我任务完成了
            	// 我在这里只需要在合适的位置去检查这个中断标志就可以了
            	// 相当于老王在看电视的同时等待水壶响起来，但是在程序里总得有个时间点去检查
            	// 通知的状态。
              System.out.println("等待等待再等待，心儿已等碎，我和你在河两岸，共饮一江水");
              if (Thread.currentThread().isInterrupted()) {
                  break;
              }
          }
          System.out.println(String.format("好吧，这哥们加完了，结果是%d", futureTask.get()));
      }
      
  }
  
  /** 实际运行效果
  * 等待等待再等待，心儿已等碎，我和你在河两岸，共饮一江水
  * 等待等待再等待，心儿已等碎，我和你在河两岸，共饮一江水
  * 等待等待再等待，心儿已等碎，我和你在河两岸，共饮一江水
  * 等待等待再等待，心儿已等碎，我和你在河两岸，共饮一江水
  * 等待等待再等待，心儿已等碎，我和你在河两岸，共饮一江水
  * 等待等待再等待，心儿已等碎，我和你在河两岸，共饮一江水
  * 好吧，这哥们加完了，结果是10
  **/
  ```

+ 异步阻塞

  终于到了这个奇怪的组合了，稍微修改一下上面的例子即可。

  ```java
  public static void main(String[] args) throws InterruptedException, ExecutionException {
  
      SumTask sumTask = new SumTask(new int[]{1, 2, 3, 4}, Thread.currentThread());
      FutureTask<Integer> futureTask = new FutureTask<>(sumTask);
      new Thread(futureTask).start();
      while (true) {
          try {
              // 在sleep调用时如果interrupt标志已经置位，会直接抛出InterruptedException
              TimeUnit.SECONDS.sleep(1);
          } catch (InterruptedException e) {
              System.out.println("哎呀被打断了");
              break;
          }
      }
      System.out.println(String.format("好吧，这哥们加完了，结果是%d", futureTask.get()));
  
  }
  ```


在另外一个参考文件上获得了一段对理解阻塞非常有意义的文字，摘抄如下：

> 正在执行的进程，由于期待的某些事件未发生，如请求系统资源失败、等待某种操作的完成、新数据尚未到达或无新工作做等，**则由系统自动执行阻塞原语(Block)，使自己由运行状态变为阻塞状态。**可见，进程的阻塞是进程自身的一种**主动行为**，也因此只有处于运行态的进程（获得CPU），才可能将其转为阻塞状态。当进程进入阻塞状态，是不占用CPU资源的。

## BIO（Blocking IO）

同步阻塞IO模式，即数据的读取和写入必须阻塞在一个线程内等待其完成。如果想在等待时干点其他的事情，就需要手动实现类似上面举的例子中的一些多线程的东西。比如一个线程去读，另外一个线程干其他事情。但是，IO操作本身是同步的，这就导致如果不想阻塞在IO上，必须要多开一个线程。

于是，最传统的、基于BIO（一请求一应答）的通信模型图如下：

![BIO-Web-Service](https://camo.githubusercontent.com/5ef6de9824ae82bb0c403522a647953d1193a362/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f322e706e67)

一般由一个Acceptor去接受请求，经典的代码就是在`while(true)`里面去调用accept接口，accept接口会阻塞一直到有新的请求到来，每当来一个新的请求，`Acceptor`就开个新的线程去处理这个请求。

一请求一应答的模式有很大的缺陷：

1. 线程的新建和销毁的成本很高。要新建栈空间，初始化一堆东西，在 Linux 这样的操作系统中，线程本质上就是一个进程，创建和销毁线程都是重量级的系统函数。而且由于栈空间也会消耗内存空间，过多的线程会导致OOM错误。
2. 线程的切换成本很高。线程切换涉及到CPU寄存器上下文保存等，一直是一个成本很高的操作。

针对上面的问题，一般会考虑用线程池来缓解问题，线程池维护一些工作线程，这样就节省了线程新建和销毁的成本，一个线程池维护M个线程，可以应对远大于M的N个请求。实际操作中，一般是将socket包装成一个Runable，交给线程池去做，于是CS模型会变成下面这样

![](https://camo.githubusercontent.com/04b258a50ca7f9762f43d64e70f4489440bae4eb/68747470733a2f2f6d792d626c6f672d746f2d7573652e6f73732d636e2d6265696a696e672e616c6979756e63732e636f6d2f332e706e67)

但这仍然有一些问题：

1. 有一些HTTP长连接需要长期保持，比如说聊天软件，不可能创建一个无限大的线程池来保持每一个连接的通信。
2. 即使线程池大小不是问题，BIO的这种阻塞模型，会要求当连接建立时对应的处理线程就要分配出来，但这时候很有可能真正的数据还在传输中，没有到达，这个等待因为是IO等待，一般都会比较长，那么工作线程在这个过程中占据了线程资源，却没有真的干活。这受限于BlockingIO的同步阻塞特性，不好消除。

比如一个经典的基于BIO的Server可能如下：

```java
/**
 * @author 闪电侠
 * @date 2018年10月14日
 * @Description: 服务端
 * 原文自：https://www.jianshu.com/p/a4e03835921a
 * 本文作者有小改动
 */
public class IOServer {

  public static void main(String[] args) throws IOException {
        // TODO 服务端处理客户端连接请求
        ServerSocket serverSocket = new ServerSocket(3333);

        // 接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理
        while (true) {
            try {
                // 阻塞方法获取新的连接
                Socket socket = serverSocket.accept();

                // 每一个新的连接都创建一个线程，负责读取数据
                new Thread(() -> {
                    try {
                        int len;
                        byte[] data = new byte[1024];
                        InputStream inputStream = socket.getInputStream();
                        // 按字节流方式读取数据
                        while ((len = inputStream.read(data)) != -1) {
                            System.out.println(new String(data, 0, len));
                        }
                    } catch (IOException e) {
                    }
                }).start();

            } catch (IOException e) {
            }

        }
    }
  
}
```

如果用连接池改写，就是这个样子：

```java
public class IOServer {

    static class SocketConsumer implements Runnable {

        private Socket socket;

        public SocketConsumer(Socket socket) {
            this.socket = socket;
        }

        @Override
        public void run() {
            try {
                int len;
                byte[] data = new byte[1024];
                InputStream inputStream = socket.getInputStream();
                // 按字节流方式读取数据
                while ((len = inputStream.read(data)) != -1) {
                    System.out.println(new String(data, 0, len));
                }
            } catch (IOException e) {
            }
        }
    }

    public static void main(String[] args) throws IOException {
        // TODO 服务端处理客户端连接请求
        ServerSocket serverSocket = new ServerSocket(3333);
        ExecutorService executorService = Executors.newFixedThreadPool(50);

        // 接收到客户端连接请求之后为每个客户端创建一个新的线程进行链路处理
        while (true) {
            try {
                // 阻塞方法获取新的连接
                Socket socket = serverSocket.accept();
                executorService.execute(new SocketConsumer(socket));

            } catch (IOException e) {
            }
        }
        
    }

}
```

BIO的好处在于，当连接数不多（单机1000连接以下）的时候，用这种编程模型非常简单，不容易出错，而且工作的也还不错。但是应对十万甚至百万级的连接时，就会非常困难。

到了JDK1.4，java引入了新的IO模型，即NIO

## NIO（Non-Blocking IO）

### NIO要解决什么问题

知道了上面BIO的缺点，那我们大概就可以猜测NIO到底想要解决什么问题：

1. 能不能用尽量少的线程对应尽可能多的连接；
2. 能不能在需要的数据都到达以后，才把数据交由真正的工作线程去处理，避免工作线程空等数据到来。

答案是可以。我们先来考虑一下一次网络IO大概要经历哪些步骤（自己的理解，有错误请自行甄别）：

1. 网卡收到了一些数据，之后将这些数据放到物理内存的某个部分。
2. 这部分物理内存只有内核态的代码有访问权限，即所谓的特权代码。熟悉内存布局的同志应该了解，一个进程中有一部分内存是操作系统使用的，只有这部分特权代码有权限访问硬件资源。而用户代码要访问这一部分内存，必须通过系统调用。Java中的IO类，实际上也是对系统调用的一种封装。
3. 现在用户代码通过Java的IO类进行了一次系统调用，此时就需要把一些数据从内核态内存复制到用户内存中（至少从模型上理解是这样的，至于内存映射之类的骚操作这里先不考虑）。那么传统的BIO就是我们一直监听着这一块内存，没有数据就等着，有数据就从内核态的内存传过来给用户用。

那么这个读取的过程，其实要包含两个部分，一个部分是等待数据到来，一个部分是复制数据，其中等待数据到来的这个过程，就是阻塞，其示意图如下：

![BIO](https://www.history-of-my-life.com/imgs-for-md/BIO-CopyFromKernel-20190828.png)

那么内核能不能修改一下这个阻塞行为，当还在等待数据到来的时候直接返回一个数据还未准备好的状态，这样就不用阻塞等待了。那当然是可以的，这就是非阻塞IO了，其模型如下：

![NIO](https://www.history-of-my-life.com/imgs-for-md/Non-BlockingIO-CopyFromKernel-20190828.png)

这个模型显然是同步非阻塞的，需要我们一直去轮询。那我们是不是需要自己手写代码去轮询这个呢？答案是你可以这么做，就像我上面的代码中的：

```java
while (true) {
    if (!futureTask.isDone()) {
      	// 我还没准备好，技能冷却中，我还不能那么做，太远了
    } else {
      	// 好了
    }
}
```

这样的话，其实我就可以用**一个线程**去监视多个socket通道了，只要我们这个线程里开一个**轮询器**，把那些Socket全都注册到这个轮询器上，我轮询一圈，看数据好了没，好了我就提示用户，没好我就继续轮询。（显然我这里说的这种初级的轮询方式会造成严重的CPU空转问题，实际的设计应该更精巧一些，比如说socket准备好了以后主动通知轮询器，但这就是异步了，我们目前先不管这个，所以了解轮询器就是这么概念模型就好）

### NIO的核心组件

好了，这就是NIO的思想了，通过我上面描述的这个简易模型，他解决了上面提出的两个问题：

1. 用尽量少的线程对应尽量多的连接；
2. 在没有真正数据准备好的时候不要来烦我。

了解了NIO要解决的问题和它的思想，我们再来正式介绍NIO的核心组件。

NIO有很多组件，但核心的组件主要是下面这三个：

+ 轮询器，NIO称为selector；

+ 需要监听的连接，NIO称为channel；

+ 接收数据的缓冲区，NIO称为Buffer。

NIO将一个连接分为好几个步骤，并且将这些步骤抽象为几个可以监听的事件，不同的事件发生后，可以通知不同的监听器。

+ Accept：有可以接受的连接到来了；
+ Connect：成功建立了连接；
+ Read：有数据可以读
+ Write：准备好接收写入数据了

显然，当我们遇到合适的事件发生时再去做合适的事情就好了，再也不用一个线程对应一个请求，在阻塞时傻傻等待了。接下来我们详细介绍一下这几个组件。

#### Channel

一个channel就代表了和某一实体的链接，这个实体可以是文件、网络套接字等，官网上是这么解读的：

> A channel represents an open connection to an entity such as a
> hardware device, a file, a network socket, or a program component that
> is capable of performing one or more distinct I/O operations, for
> example reading or writing.

这个组件的地位，有点类似于BIO中的Stream，其功能也有点类似Stream，就是从一个Buffer中读取数据，或者把数据写入Buffer。

Java中NIO中最常用的通道实现有以下几个：

- FileChannel：读写文件，只有阻塞模式。这是因为UNIX当时不支持文件的非阻塞读取，而Java为了跨平台的表现一致，也就这样设置了。参考：[Why FileChannel in Java is not non-blocking?](https://stackoverflow.com/questions/3955250/why-filechannel-in-java-is-not-non-blocking)

  这意味着它也没法注册到Selector上面去。

- DatagramChannel: UDP协议网络通信

- SocketChannel：TCP协议网络通信

- ServerSocketChannel：监听TCP连接，TCP连接的服务端。

事件也是在Channel上发生的，所以我们需要在注册事件时，告知Selector我到底关心这个Channel上发生了哪些事件。

要注意的是，FileChannel只有阻塞模式。具体原因待调差。

#### Buffer

这就是NIO中使用的缓冲区，他不是一个byte[]，而是封装过的一个Buffer类，其继承层次如下：

![NIO-Buffer](https://www.history-of-my-life.com/imgs-for-md/NonBlockingIO-channel-20190828-01.png)

看来还帮我们封装了一些基本的数据类型，这样读起来可能方便一些，分配内存的时候，也是按相应数据类型来计算size的，比如`IntBuffer.allocate(1024)`就是申请一个能放1024个Int的缓冲区，而`Double.allocate(1024)`就是申请一个能放1024个Double的缓冲区。

Buffer有几个重要的变量，就是上面标注的那四个。

+ capacity。容量，allocate后就不会变了；
+ limit。此次可以操作的最后一个元素的位置。显然当Buffer被用于写的时候，就是Buffer的capacity，当Buffer用于读的时候，就是Buffer中已写入的元素的数量。
+ position。当前操作到的那个元素。
+ mark。一个可以随时指定的标记位，可以调用`Buffer#mark()`标记，调用`Buffer#reset()`使position指针回到上一次mark的位置。

Buffer即可以用来读，也可以用来写，只是不能同时用来读和写，从写状态切换为读状态使用`filp()`，从读切换到写状态使用`compact()`。

读写状态调用`Buffer#filp()`方法来进行转换。一般来说，都是先写进去一些数据，然后再读出来，而这个状态的切换实际上也挺精妙的。我们在写的过程中，position一直在增长，指向最后一个写入的元素，此时limit表示可以写的最后一个位置的下一个位置，当然是和capacity的值保持一致。那当要从这个Buffer里读取元素时，只需要将limit置为写的position，然后将position置为0即可，这样就可以从0开始读，最多读到position。再由读翻转到写，只要吧position置为0，limit再置为capacity就可以了。

#### Selector

根据上文描述的简易模型，这个选择器就对应着轮询器，用于监听各个通道中的事件。我们先将通道注册到选择器，并设置好关心的事件，然后就可以通过调用select()方法，等待我们关心的事件发生。

关心的事件上面已经介绍过了。

Selector本身是阻塞的，当没有任何关心的事件发生时，他不会占据CPU，在内存消耗上，他也只占据了一个线程的资源。当发生了感兴趣的事情以后，它再进行实际的工作。而且不像非阻塞IO那样，需要我们自己不断去查询连接状态，这样会空耗CPU。所以确实是一个比较理想的方案。

### 一个例子

```java
// 这是个单线程的Server
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.Iterator;

public class NioServer {

    public static void main(String[] args) throws IOException {
        // 创建一个selector
        Selector selector = Selector.open();

        // 初始化TCP连接监听通道
        ServerSocketChannel listenChannel = ServerSocketChannel.open();
        listenChannel.bind(new InetSocketAddress(8080));
      	// NIO的阻塞与否是可选的，在网络环境下我们将其设置为非阻塞
        listenChannel.configureBlocking(false);
        // 注册到selector（监听其ACCEPT事件）
        listenChannel.register(selector, SelectionKey.OP_ACCEPT);

        // 创建一个缓冲区
        ByteBuffer buffer = ByteBuffer.allocate(100);

        while (true) {
            selector.select(); //阻塞，直到有监听的事件发生
          	// 可能不止有一个连接状态发生了变化
            Iterator<SelectionKey> keyIter = selector.selectedKeys().iterator();

            // 通过迭代器依次访问select出来的Channel事件
            while (keyIter.hasNext()) {
                SelectionKey key = keyIter.next();
                if (key.isAcceptable()) { // 有连接可以接受
                    SocketChannel channel = ((ServerSocketChannel) key.channel()).accept();
                    channel.configureBlocking(false);
                  	// 再次注册，这次指明关心的事件是可读
                    channel.register(selector, SelectionKey.OP_READ);

                    System.out.println("与【" + channel.getRemoteAddress() + "】建立了连接！");

                } else if (key.isReadable()) { // 有数据可以读取
                    buffer.clear();

                    // 读取-1说明到了流末尾，说明TCP连接已断开，
                    // 因此需要关闭通道或者取消监听READ事件
                    // 否则会无限循环
                    if (((SocketChannel) key.channel()).read(buffer) == -1) {
                        key.channel().close();
                      	// 已经处理的事件一定要手动移除
                				keyIter.remove();
                        continue;
                    }

                    // 按字节遍历数据
                    buffer.flip();
                    while (buffer.hasRemaining()) {
                        byte b = buffer.get();

                        if (b == 0) { // 客户端消息末尾的\0
                            System.out.println();

                            // 响应客户端
                            buffer.clear();
                            buffer.put("Hello, Client!\0".getBytes());
                            buffer.flip();
                            while (buffer.hasRemaining()) {
                                ((SocketChannel) key.channel()).write(buffer);
                            }
                        } else {
                            System.out.print((char) b);
                        }
                    }
                }

                // 已经处理的事件一定要手动移除
                keyIter.remove();
            }
        }
    }
}
```

再来一个简单的Client

```java
public class NioClient {

    public static void main(String[] args) throws Exception {
        Socket socket = new Socket("localhost", 8080);
        InputStream is = socket.getInputStream();
        OutputStream os = socket.getOutputStream();

        // 先向服务端发送数据
        os.write("Hello, Server!\0".getBytes());

        // 读取服务端发来的数据
        int b;
        while ((b = is.read()) != 0) {
            System.out.print((char) b);
        }
        System.out.println();

        socket.close();
    }

}
```

运行一下是这样的：

```
# server端
与【/127.0.0.1:59974】建立了连接！
Hello, Server!

# client端
Hello, Client!
```

### NIO并不完美

这个例子展示了挺多东西。首先，我们可以看到Server用单线程处理了多个连接。但也暴露出一些问题。

1. 如果我们的服务器里用上面的单线程模型去处理，那么多CPU的性能优势就体现不出来。当然了，我们可以改善这个模型，使用一个线程来监听Accept事件，用多个线程来监听Read事件，代码就会变成这个样子。

   ```java
   import java.io.IOException;
   import java.net.InetSocketAddress;
   import java.nio.ByteBuffer;
   import java.nio.channels.SelectionKey;
   import java.nio.channels.Selector;
   import java.nio.channels.ServerSocketChannel;
   import java.nio.channels.SocketChannel;
   import java.util.Iterator;
   
   public class NioServerMultiThread {
   
       static class AcceptListener implements Runnable {
   
           private Selector acceptSelector;
           private Selector[] readSelectors;
           private int index = 0;
   
           AcceptListener(Selector acceptSelector, Selector[] readSelectors) {
               this.acceptSelector = acceptSelector;
               this.readSelectors = readSelectors;
           }
   
           @Override
           public void run() {
               try {
                   ServerSocketChannel listenChannel = ServerSocketChannel.open();
                   listenChannel.bind(new InetSocketAddress(8081));
                   listenChannel.configureBlocking(false);
                   // 注册到selector（监听其ACCEPT事件）
                   listenChannel.register(acceptSelector, SelectionKey.OP_ACCEPT);
   
                   while (true) {
                       if (acceptSelector.select(1) > 0) {
                           Iterator<SelectionKey> keyIter = acceptSelector.selectedKeys().iterator();
   
                           // 通过迭代器依次访问select出来的Channel事件
                           while (keyIter.hasNext()) {
                               SelectionKey key = keyIter.next();
   
                               if (key.isAcceptable()) { // 有连接可以接受
                                   try {
                                       SocketChannel channel = ((ServerSocketChannel) key.channel()).accept();
                                       channel.configureBlocking(false);
                                     	// 这个register方法有可能出现问题
                                       channel.register(readSelectors[index], SelectionKey.OP_READ);
                                       index = (index + 1) % readSelectors.length;
   
                                       System.out.println("与【" + channel.getRemoteAddress() + "】建立了连接！");
                                   } finally {
                                       keyIter.remove();
                                   }
                               }
                           }
                       }
                   }
               } catch (IOException ex) {
                   // do nothing
               }
           }
   
       }
   
       static class ReadListener implements Runnable {
   
           private Selector readSelector;
           private ByteBuffer buffer = ByteBuffer.allocate(100);
   
           ReadListener(Selector readSelector) {
               this.readSelector = readSelector;
           }
   
           @Override
           public void run() {
               try {
                   while (true) {
                     	// 如果用上个例子的select()方法会导致acceptListener中的register阻塞
                     	// 原因是readSelector锁定了内部key等待，而这个key在register中也需要先获取才能register，就死锁了
                     	// 所以只好让他定期来释放锁，给register以机会
                       if (readSelector.select(1) > 0) {
                           Iterator<SelectionKey> keyIter = readSelector.selectedKeys().iterator();
                           while (keyIter.hasNext()) {
                               SelectionKey key = keyIter.next();
   
                               if (key.isReadable()) {
                                   try {
                                       buffer.clear();
   
                                       // 读取到流末尾说明TCP连接已断开，
                                       // 因此需要关闭通道或者取消监听READ事件
                                       // 否则会无限循环
                                       if (((SocketChannel) key.channel()).read(buffer) == -1) {
                                           key.channel().close();
                                           continue;
                                       }
                                       buffer.flip();
                                       while (buffer.hasRemaining()) {
                                           byte b = buffer.get();
   
                                           if (b == 0) { // 客户端消息末尾的\0
                                               System.out.println();
   
                                               // 响应客户端
                                               buffer.clear();
                                               buffer.put("Hello, Client!\0".getBytes());
                                               buffer.flip();
                                               while (buffer.hasRemaining()) {
                                                   ((SocketChannel) key.channel()).write(buffer);
                                               }
                                           } else {
                                               System.out.print((char) b);
                                           }
                                       }
                                   } finally {
                                       keyIter.remove();
                                   }
                               }
                           }
                       }
                   }
               } catch (IOException ex) {
                   // do nothing
               }
           }
   
       }
   
       public static void main(String[] args) throws IOException {
   
           Selector acceptSelector = Selector.open();
           Selector[] readSelectors = new Selector[]{
                   Selector.open(),
                   Selector.open(),
                   Selector.open()
           };  // 假设我们是4核机器，于是开一个线程监听连接，开三个线程处理业务
   
           new Thread(new AcceptListener(acceptSelector, readSelectors)).start();
           new Thread(new ReadListener(readSelectors[0])).start();
           new Thread(new ReadListener(readSelectors[1])).start();
           new Thread(new ReadListener(readSelectors[2])).start();
   
       }
       
   }
   ```

2. 在这里我们对数据的处理的耗时非常短暂，实际上只是读出数据，并把它输出出来而已。但还是要知道，这个处理的过程对于每个readSelector所在的线程来说，都是串行的。既然是串行的，自然会导致其他请求处理不了，假如我们的这个处理的耗时比较长，比如说再有个数据库操作、文件读写、$$n^3$$以上复杂度的计算等，那其他注册在该selector上的请求都必须等待这个处理完了再说。这也是个灾难。我们当然可以启动N个readSelector来缓解这个问题，但是如果你启动了非常多的Selector，那么线程太多导致的内存和CPU切换耗资源的问题不就又出现了吗？

   这个问题的解决方案，老实说我暂时没想出来。是不是要走BIO的老路，再建一个连接池，每一个请求read以后把read的结果包装成一个Runnable，再让线程池去处理，这不就又回到了一个请求一个线程的老路上去了，唯一的优化就在于至少在创建线程时工作线程都是有事可干的。

   我只能猜想，这种耗时巨大的一些线上运算，或许用一个请求一个连接的BIO模型更优。这样至少不会出现前请求完成不了，后请求必须排队的情况。要解决并发不够的问题的话，那就部署集群好一点，反正加硬件几乎总是能解千愁。而NIO这种一个线程处理很多连接的方式，似乎更适合做一些不怎么占CPU的处理。

3. 上面的例子还有一些问题，比如说buffer是有capacity的，因此一条信息很有可能不能在一次读完，同一条连接会发过来很多个信息，我们肯定要对这些信息进行拼接、定界。在响应的时候，我们也不可能用一个while循环真的把数据写完才做其他工作，不然写入的话限于缓冲区和网络IO的慢速度，又是一种浪费CPU的阻塞了。

4. 这里我们的客户端和服务器之间的通信简直过于简单了，没有用任何协议，直接字节流扔过去，手动写了一个`\0`作为结束标志。现实中的客户端和服务器的通信哪有这样的，各种通信不都得有个协议，HTTP、FTP等，而我们已经看到了，用这一套原生的NIO接口，我们处理的是一些基础类型，比如char、int之类的，你得自己去处理各种协议。这还不玩死自己。所以，用这套源生API真的处理业务问题，会导致非常难写、非常容易出错。

5. 另外，epoll的某种实现似乎还有bug，可能导致cpu空转。

要解决上面这些问题，需要有一个非常严谨和繁琐的设计，幸运的是我们不需要真的自己去设计这个系统，开源框架Netty已经帮我们封装好了NIO的原生接口，解决了很多问题。关于Netty会在后续的文章中予以介绍。

### NIO与内存管理

[JVM源码分析之堆外内存完全解读](http://lovestblog.cn/blog/2015/05/12/direct-buffer/)

[Java NIO direct buffer的优势在哪儿？](https://www.zhihu.com/question/60892134)

[JAVA的堆外内存(基于NIO的ByteBuffer类实现)](https://blog.csdn.net/u014590757/article/details/79856425)

在读JVM内存管理相关的内容时，我们经常会读到一句话：NIO使用了本地内存，而不是堆内存，因此大量使用NIO可能造成与传统的堆内存OOM表现不太一致的OOM错误。那么这个到底是怎么回事呢？

先说答案的一部分：这是因为NIO中使用的Buffer，在直接分配到本地内存时的IO效率比分配到堆内存上的效率要高一点，所以框架大量使用了本地内存上分配Buffer，而这造成了OOM的隐患，而且与堆内存溢出的OOM表现不太一致。于是引出这么几个问题：

1. 怎么在本地内存上直接分配空间？

   ```java
   Buffer buffer = ByteBuffer.allocateDirect(1024);		// 在本地内存分配空间
   Buffer buffer = ByteBuffer.allocate(1024);					// 在堆内存分配空间
   ```

2. 为什么在堆内存分配缓存空间要慢一点？

   在调用IO的系统调用，将内存从用户空间复制到内核空间时，或者将内核空间的数据复制到用户空间的过程中，要求传入系统调用的内存地址必须满足两个条件：

   + 不能变；
   + 必须是连续的。

   而因为GC的存在，java堆内存会在不确定的时间点发生变化，其实也不是不能pin住单个对象不让他变化，但这就增加了内存收集的复杂性，直接破坏了GC的基本模型不说，这样不可变的内存分配多了，还有可能把内存割裂成一块一块的，另外，GC的新生代回收非常频繁，这样的一些缓存pin在哪里，假如卡IO了，很容易短时间内耗光eden区，整体上来说，有一点得不偿失，所以JVM的作者普遍没有选择这么做。

   此外，byte[]在java规范中并不要求是连续的，虽然绝大多数实现中他们确实是连续的。但并不能保证这一点。

   综合以上两个原因，在堆中分配的Buffer，在真正进行IO的时候，需要复制到本地内存中去。因为对关联的这一块本地内存的清理只在Full GC时进行，频率要低得多，所以适合做这种需要等待的事情。

3. 为什么不只提供直接分配在本地内存中的Buffer，取消分配在堆中的buffer。

   不是不可以，但是直接分配在堆中有个好处，就是可以被GC，而分配在直接内存中的数据只有在fullGC时才能被回收（如果禁用了显式GC，那就回收不了了），容易引发堆内存明明还有挺多，但是却OOM的问题。

那么，这部分内存到底在什么时候回收呢？可以参考:[JVM源码分析之堆外内存完全解读](http://lovestblog.cn/blog/2015/05/12/direct-buffer/)

> DirectByteBuffer对象在创建的时候关联了一个PhantomReference，说到PhantomReference它其实主要是用来跟踪对象何时被回收的，它不能影响gc决策，但是gc过程中如果发现某个对象除了只有PhantomReference引用它之外，并没有其他的地方引用它了，那将会把这个引用放到java.lang.ref.Reference.pending队列里，在gc完毕的时候通知ReferenceHandler这个守护线程去执行一些后置处理，而DirectByteBuffer关联的PhantomReference是PhantomReference的一个子类，在最终的处理里会通过Unsafe的free接口来释放DirectByteBuffer对应的堆外内存块

因此，假如你闲的没事，使用JVM参数`-XX:+DisableExplicitGC`关闭了显式GC，那么这部分内存永远不会释放了，时间长一点就等着OOM吧。

## AIO（Asynchronous I/O）

如果我们仔细观察上面所说的Non-Blocking IO，我们会发现一个问题，即真正的`IO operation`，还是阻塞的。要理解这个问题，首先要精确定义**IO operation**，我们这里所说的`IO operation`，是指把数据从kernel内存中复制到user内存中的这个过程。

在BIO时，我们的线程在IO的整个过程，包括等待连接建立、等待数据就位、从内核态复制数据到用户态的过程中，全都都是阻塞等待的。

在NIO中，我们的线程在IO还没有准备好的时候不用阻塞，当IO的数据准备好了以后，从内核内存复制到用户内存的过程里，是阻塞的。因为这个阻塞过程，意味着用户如果要同时做其他工作，就要控制这个读写的过程不能阻塞太久，不然还是要放到单独的IO线程里面去执行。

那么有没有一种办法，把这个内核态到用户态的复制这个步骤也搞成非阻塞的，等到需要的数据都拷贝到用户内存里以后，再来通知我说复制好了。那这样岂不美哉？甚至更进一步，我直接把一个回调函数传给负责IO的线程，说等你复制完数据以后，直接帮我执行一下这个回调函数就好了。这样主线程写起来是不是就更快乐了？

用取包裹举个例子，BIO同步阻塞年代，是我站在门口痴痴地等待快递小哥到来，NIO时代，是快递小哥到了门口给我打电话我去取，而AIO时代，是快递小哥把东西直接搬到我指定的位置放好，然后给我打个电话说一声。

那么AIO到底解决了什么问题呢？大概就是在IO执行过程中彻底不需要阻塞了，只在数据完全准备好时才通知应用程序处理，而且回调方法的出现，多少简化了编程模型，并且它甚至增加了对线程池的源生支持，可以为回调方法的执行预先准备一个线程池。

多少有点美好。那我们首先来看一下AIO的主要接口。

### AIO的核心组件

#### AsynchronousChannel

我们可以看到，NIO和AIO在前半部分是很像的，所以这个Channel和NIO的Channel其实也是一样的，即代表了我们关注的一些连接。

他的子类包括：

+ AsynchronousFileChannel
+ AsynchronousServerSocketChannel
+ AsynchronousSocketChannel

#### AsynchronousChannelGroup

异步channel的分组管理机制，主要是一个Group可以绑定一个线程池，这个线程池用来处理IO事件和CompletionHandler。AsynchronousServerSocketChannel创建的时候可以传入一个AsynchronousChannelGroup，那么通过AsynchronousServerSocketChannel创建的 AsynchronousSocketChannel将同属于一个组，共享资源。代码如下：

```java
AsynchronousChannelGroup group = AsynchronousChannelGroup.withThreadPool(Executors.newFixedThreadPool(4));
AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open(group).bind(new InetSocketAddress("0.0.0.0", 8013));
```

#### CompletionHandler

回调借口。AIO的API允许两种方式来处理异步操作的结果：返回的Future模式或者注册CompletionHandler。返回Future的方式，因为`Future#get()`是阻塞接口，所以用的稍微不仔细一点，就成了异步阻塞的那一套了，所以可能用CompletionHandler要更好一些，这些handler的调用是由AsynchronousChannelGroup的线程池派发的。因此线程池的使用也与性能相关，线程池的使用不是这里的重点，在这里不详述。

其接口如下：

```java
package java.nio.channels;

public interface CompletionHandler<V,A> {  
  
    void completed(V result, A attachment);  
  
    void failed(Throwable exc, A attachment);  

}
```

其中Result表示IO调用的结果，而A是你需要使用的任意一个对象，在IO调用时传入。

抄一段代码如下：

```java
import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.ByteBuffer;
import java.nio.CharBuffer;
import java.nio.channels.AsynchronousChannelGroup;
import java.nio.channels.AsynchronousServerSocketChannel;
import java.nio.channels.AsynchronousSocketChannel;
import java.nio.channels.CompletionHandler;
import java.nio.charset.Charset;
import java.nio.charset.CharsetEncoder;
import java.util.Date;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;
public class Server {
		private static Charset charset = StandardCharsets.US_ASCII;
    private static CharsetEncoder encoder = charset.newEncoder();

    public static void main(String[] args) throws Exception {
        AsynchronousChannelGroup group = AsynchronousChannelGroup.withThreadPool(Executors.newFixedThreadPool(4));
        AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open(group).bind(new InetSocketAddress("0.0.0.0", 8013));
        server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            @Override
            public void completed(AsynchronousSocketChannel result, Void attachment) {
                server.accept(null, this); // 接受下一个连接
                try {
                    String now = new Date().toString();
                    ByteBuffer buffer = encoder.encode(CharBuffer.wrap(now + "\r\n"));
                    //result.write(buffer, null, new CompletionHandler<Integer,Void>(){...}); //callback or
                    Future<Integer> f = result.write(buffer);
                    f.get();
                    System.out.println("sent to client: " + now);
                    result.close();
                } catch (IOException | InterruptedException | ExecutionException e) {
                    e.printStackTrace();
                }
            }

            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
            }
        });
        group.awaitTermination(Long.MAX_VALUE, TimeUnit.SECONDS);
    }
}
```

这个例子里用了两种方式，其中发送数据用了future。但这个Future其实也就是个数字，表明写了几个字符，没有实际意义。

Client实现：

```java
public class Client {
    public static void main(String[] args) throws Exception {
        AsynchronousSocketChannel client = AsynchronousSocketChannel.open();
        Future<Void> future = client.connect(new InetSocketAddress("127.0.0.1", 8013));
        future.get();

        ByteBuffer buffer = ByteBuffer.allocate(100);
        client.read(buffer, null, new CompletionHandler<Integer, Void>() {
            @Override
            public void completed(Integer result, Void attachment) {
                System.out.println("client received: " + new String(buffer.array()));

            }

            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
                try {
                    client.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }

            }
        });

        Thread.sleep(10000);
    }
}
```

### 完美了吗？

其实我们可以发现，上面的编程模型确实稍微简化了一点。只需要告诉系统IO完了之后执行哪些任务就可以了，Selector用不到了，因为不需要我们去轮询了，任务的分派系统可以自动利用线程池，心智负担确实小了一些。

但是这个也不是就没有问题了。其实他面对的主要问题，和NIO是一样的，就是协议的处理、拆包、定界等这些问题。一样繁琐，实际上还是依赖框架的包装心智负担更小一些。

## BIO NIO AIO的适用场景

这个其实我也没有经验，只要直接COPY网上的东西了：

+ BIO方式适用于连接数目比较小且固定的架构，这种方式对服务器资源要求比较高，并发局限于应用中，JDK1.4以前的唯一选择，但程序直观简单易理解。

+ NIO方式适用于连接数目多且连接比较短（轻操作）的架构，比如聊天服务器，并发局限于应用中，编程比较复杂，JDK1.4开始支持。

+ AIO方式使用于连接数目多且连接比较长（重操作）的架构，比如相册服务器，充分调用OS参与并发操作，编程比较复杂，JDK7开始支持。

## 参考

+ 阻塞和同步异步相关
  + [怎样理解阻塞非阻塞与同步异步的区别](https://www.zhihu.com/question/19732473)
  + [Linux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)
+ BIO相关
  + [githun:BIO-NIO-AIO](https://github.com/Snailclimb/JavaGuide/blob/master/docs/java/BIO-NIO-AIO.md)
+ NIO相关
  - [Java NIO浅析](https://tech.meituan.com/2016/11/04/nio.html)
  - [JAVA NIO学习一：NIO简介、NIO&IO的主要区别](https://www.cnblogs.com/pony1223/p/8138233.html)
  - [一文让你彻底理解 Java NIO 核心组件](https://segmentfault.com/a/1190000017040893)
  - [跟闪电侠学Netty](https://www.jianshu.com/p/a4e03835921a)
  - [Why FileChannel in Java is not non-blocking?](https://stackoverflow.com/questions/3955250/why-filechannel-in-java-is-not-non-blocking)

+ AIO相关
  - [深入理解BIO、NIO、AIO](https://zhuanlan.zhihu.com/p/51453522)
  - [java aio 编程](https://colobu.com/2014/11/13/java-aio-introduction/)
  - [Java NIO系列教程（八）JDK AIO编程](https://www.cnblogs.com/duanxz/p/6782803.html)

## 附

这些IO方式我们现实中可能都很少直接用到，但是IO方式的改进反映出的思想确实很有价值，值得我们参考借鉴。

另外：笔记仅用于自己学习和几位好友的内部技术交流，请勿转发给工作相关人员。如果承蒙错爱想发博客，注明一下作者即可。