---
layout: post
title: 对网络编程框架的理解
---

网络框架是分布式系统中不可或缺的组件，我认为评价一个框架的好坏需要从性能和是否方便使用两方面考虑，在这总结下在参与pika、zeppelin、zeppelin-s3-gateway期间使用pink[1]，调研优化pink，以及开发新网络框架Procyon[2]产生的一些关于网络框架的想法。

所谓框架应该就是提供了大部分代码，用户只需填充规定的部分代码即可，不用从头接触各种系统调用，所以网络框架一般情况下指的就是对TCP协议的socket各种封装，随着计算机处理能力的增强，以及应用规模的增大，程序员需要更好的利用硬件和操作系统提供的各种资源，所以随之出现了各种处理网络链接的模型，多线程，event loop等等，网络编程框架也就是对一种模型的实现，对于不同应用一般是通用的，方便了项目开发。

### Eventbased Concurrency [3]

关于对一个客户端链接的处理很容易想到的一个办法就是每个链接建一个进程（后来改进为用线程）处理，这也是当时学操作系统时介绍的进程的一个作用(在FTP服务中，父进程接收新链接，然后建立子进程向客户端回复数据)。但是这种方法的缺点就是为不同客户端服务时会带来进程/线程切换的开销，从而无法服务太多的客户端。

这时出现了event-base的方法，在一个event loop中监听所有链接event，当然这些event优先级是相同的，触发多个event时由调用者决定先处理哪个，所以可以想象得到，event数量多了性能也不会太好。

这时，把多线程和event-base结合起来将是一个很好的选择，可以充分利用系统资源，如下图所示，每个thread里运行一个event loop，每个线程处理自己的event，最大程度上提高了并发度。

pink及procyon都采用了这种模型。

![img](http://i.imgur.com/XXfibpV.png)

### 分阶段处理

上文提到的每个线程中运行一个event loop方法也存在比较明显的缺点，一个worker thread内部的所有事件只能顺序处理，如果其中一个处理得非常慢，那么会影响后面所有的时间。因为所有的event是同等优先级的。

Matt Welsh提出了SEDA[4]架构，主要想法就是将一个网络请求分成多个阶段，每个阶段通过一个线程池去处理，同时每个阶段的线程池可以动态调整，通过这种方法将一个线程内部的顺序处理event优化成并发处理，进一步提高了资源利用率。

我认为这里的阶段分为两阶段就足够了，处理阶段分的太细带来的线程切换，同步开销，以及对调试便利性的影响也不能忽视。这里第一阶段就是读事件触发后协议数据的接收及解析，这一阶段在event loop同一线程处理即可，因为这一阶段用时一般不会太长，对后续事件的影响不会太大。第二阶段就是对协议内容的处理，具体看用户操作，可以是原地计算，也可以是本地存储，或者是访问其他远端服务器，这一阶段的操作耗时对于用户来说一般是可推断的，比如前面所说的三种操作耗时就是依次增加的。

对于耗时长的任务阶段，可以交给另外的线程处理，就不会阻塞后续的事件处理，所以上图中标为Worker Thread的线程在这种情况下可以称为IOThread，专门负责网络I/O，而处理第二阶段任务的线程称为WorkerThread，后文将使用这两种线程称呼。

### 抽象

pink提供了两种线程模型，HolyThread：通过单线程accept并处理所有链接；DispatchThread：一个线程负责接收新链接然后分发给IOThread处理。感觉这里的HolyThread比较鸡肋，大部分代码逻辑无法复用，而且还可以用只有一个IOThread的DispatchThread替代，虽然多了一个线程，但也不会有太多影响，而且清晰多了。

在pink中，每个客户端链接的抽象——Connection对象被视为每个IOThread的组成部分，每个IOThread从逻辑和实际上来管理自己处理的Connection对象，用户通过继承Connection子类实现TCP数据读取，并解析协议，以及实现自己的回调处理函数来进行上文提到的第二阶段协议处理。但是所有Connection的子类都需要实现自己的读TCP逻辑，其实这一段逻辑可以抽象出来。

所以Connection应该只负责TCP数据读取，在不同子类中（比如HTTPConn，RedisConn）实现不同的协议解析，同时用户定义handle来处理协议数据。将不同的逻辑清晰抽象出来。

在procyon中Connection的管理集中在DispatchThread中，建立一个新客户端链接后新建Connection对象并存储管理，这样比分散到各个IOThread中方便多了。

在pink中，DispatchThread和IOThread实现了自己的eventloop代码，这一段eventloop可以独立抽象出，因为都是对事件的监听，只是有不同的处理。所以在procyon中一个eventloop就是运行在一个线程内的Runnable对象，不同线程对自己的eventloop注册不同的回调即可。

### 缓冲区管理

在pink中对于链接缓冲区的管理采用的是固定地址，大小动态改变。这种方式一般无法很好的预测所需缓冲区的大小，一旦需要扩大就需要将原来位置的数据拷贝到新位置，在一定程度上增加了开销和复杂性。同时对于要回复给客户端的数据pink是先将数据复制到一个缓冲区再通过write调用写回，对于一些不需序列化的协议（Redis、HTTP等），在这里的复制完全是多余的。

在procyon中参考folly[5]实现了IOBuf，IOBuf可以理解为一块缓冲区的"视图"，多个IOBuf可以指向同一缓冲区的不同位置，共享同一块内存，同时实现引用计数，最后一个释放的IOBuf会释放掉内存空间，如下图所示：

```
               +-----------+     +---------+
               |  IOBuf 1  |     | IOBuf 2 |
 buffer        +-----------+     +---------+
       |        |         | _____/         |
       |   data |    tail |/    data       | tail
       |        v         v                v
       -> +--------------------+     +---------------------+
          |     |         |    |     |     |               |
          +--------------------+  -> +---------------------+
                   Block          |           Block
                           buffer-|
```

有了IOBuf，可以方便的扩充缓冲区大小，避免内存拷贝。

### 性能 

procyon性能在qps跑分上来看是下降的（6%左右），如下图所示。但我认为qps不能代表一切，只能表示框架本身带来的性能损耗多少，但实际应用并不是空跑echo程序，所以测试带有负载时的性能才比较准确。

>Xeon E5-2630，24c，160GB Mem，1000 Mbps，ServerIOThread 24 thread

![image-20180411092701048](/public/img/networkframe/performance.png)

在实际应用中，性能瓶颈并不在网络框架上，同时procyon的性能下降是由于IOBuf动态申请的原因，但我认为IOBuf带来的便利性足以抵消这点性能下降。

上文也提到过，如果有处理非常慢的请求，会影响到该线程中其他的请求处理速度，所以在测试的时候需要模拟这种场景。再加入1%的慢请求（sleep 5ms）后procyon的表现优于pink（8%左右），如下图所示：

> Xeon E5-2630，24c，160GB Mem，1000 Mbps，ServerIOThread 10 thread

![image-20180411092711267](/public/img/networkframe/performance_with_longreq.png)

此时我们对正常请求的的延迟测试也表示长请求会影响到正常的请求处理。下图（a）是在pink中的测试结果，（b）是procyon的测试结果。

>Xeon E5-2630，24c，160GB Mem，1000 Mbps，ServerIOThread 10 thread，100 client，1% slow request。
>
>./redis-benchmark -n 100000 -c 20 -r 1000 -t set -l -d 4

![屏幕快照 2018-04-11 10.18.43](/public/img/networkframe/latency.png)

但是有一个奇怪的现象，就是在网卡跑满后，procyon的CPU利用率要高于pink，所以qps也会高不少，同时上下文切换数量也比较高，这里的原因还不太清楚。

### 总结

此外，在pink中新建Server时需要传入一堆参数，包括定时任务的各种handle等，这一些可以整合到一个Options结构体中，使代码简洁清晰。

在每个请求处理handle里用户可以直接操作Connection对象，关闭链接或者回复数据。这种方式也方便了将请求转发给其它线程处理的想法实现。

procyon参考了brpc[6]和wangle[7]的一部分设计。其中brpc通过bthread实现不同fd之间，以及请求不同阶段的并发处理，bthread是一个M:N线程库，N对应的是pthread，M远大于N。bthread通过work stealing和butex实现一个bthread阻塞不会影响其它bthread的功能。卡住指的是调用bthread API阻塞，也就是因butex卡住，会将当前对应的pthread调度给其它bthread使用；但如果因为系统API阻塞整个pthread还是会阻塞，而运行在这个pthread上的bthread会挪到其它pthread上执行。

上文也提到不少有关于减少内存拷贝的优化，但在存储类应用中，系统的性能瓶颈往往在磁盘I/O上，内存拷贝相对来说非常快，一个单机存储类应用最大qps一般在200K以下，而网络框架提供的性能远远超过需求，所以有很大的空间可以损失一些性能来换取一些更便捷的功能。

在wangle和brpc中都提供了异步客户端功能。在pika、zeppelin中客户端用于发送binlog，不需要接收回复；在floyd中客户端用户rpc请求，需要同步接收回复。所以虽然异步客户端能让跑分更高，但需要异步的场景不多，感觉异步客户端用处不大。

### 参考

1. [https://github.com/PikaLabs/pink](https://github.com/PikaLabs/pink)
2. [https://github.com/gaodq/procyon](https://github.com/gaodq/procyon)
3. [talk about event based concurrency](http://baotiao.github.io/2016/11/26/concurrency)
4. [SEDA: An Architecture for Well-Conditioned, Scalable Internet Services](http://www.sosp.org/2001/papers/welsh.pdf)
5. [https://github.com/facebook/folly](https://github.com/facebook/folly)
6. [https://github.com/brpc/brpc](https://github.com/brpc/brpc)
7. [https://github.com/facebook/wangle](https://github.com/facebook/wangle)