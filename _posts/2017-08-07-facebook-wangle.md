---
layout: post
title: Facebook开源网络编程框架wangle简介
---

前一阵为了比较pink的性能，测了下wangle框架，也是因为一开始对框架不熟悉，pingpang测出来200多万的QPS，比pink高了一倍，简直惊呆了，近期仔细看了下，对wangle也有了些许了解，wangle是一异步网络框架，参考了Netty + Finagle中的设计理念。同时wangle用了另一基础库folly，两者都需要gcc支持C++14，用了大量模板，函数式，各种新特性，folly也有一些基本数据结构和对标准库的一些功能扩充，是学习C++的一个不错的样例。

本文主要对wangle整体结构做一个总结，同时找出和pink的异同点，为啥测出来的性能比pink好。

## 设计理念及整体结构

![wangle框架](/public/img/wangle_framework.svg)图 1

图 1所示的是wangle整体架构，左边是常见的eventbase loop + threadpool模型，右边所示的是wangle从netty移植到C++的Pipeline，每一部分成为Handler，可以进行数据编解码，格式转换等操作，对外最终表现的写操作通过pipeline类的写方法实现，读操作用回调函数实现，当socket读到一定数据时，通过pipeline一路回调下去。

整个wangle设计的是一异步模型，通过回调实现读操作，通过folly::Future实现异步写操作，folly::Future是facebook对std::future的扩展，增加了回调功能。

wangle提供了两种ThreadPoolExecutor，I/O密集型和CPU密集型，I/O任务结束可以扔给CPU线程池做一些CPU工作，CPU线程处理完数据后可以方便的通过传入的上下文指针回应客户端。

## Pipeline及其Handlers

pipeline在这里可以翻译为数据管道，多个Handlers组成了pipeline，图 1所示的Wout、Win、Rin、Rout指的是模板参数，作为Handler处理的类型，定义Handler时传入的模板实参也就是这个Handler要做的类型转换目标。

不同方向的pipeline不一定完全对称，图 1所示的pipeline一共有四个Handlers，其中OutboundHandler可以只处理write方向的数据转换，同样的InboundHandler也可以只处理read方向，剩下的两个Handler是两个方向都要处理。每个Handler需要实现父类的read或write虚函数，然后通过ctx->fireRead(Rout)或ctx->fireWrite(Wout)将自己处理过的信息传递到pipeline的下一个Handler，所以Rout和Wout对应的就是下一个Hanler的类型的Rin和Win。其中ctx指的是所属pipeline的Context指针。

pipeline通过addBack()方法将Handler加入自己的Context，最后一个addBack()的Handler将作为写操作的入口。同样第一个addBack()的Handler将作为读操作的入口。

举个栗子：

wangle的echo example里构造了如下的pipeline

```cpp
pipeline->addBack(AsyncSocketHandler(sock));
pipeline->addBack(EventBaseHandler());
pipeline->addBack(LineBasedFrameDecoder(8192, false));
pipeline->addBack(StringCodec());
pipeline->addBack(EchoHandler());
```

其中StringCodec的类定义如下

```cpp
class StringCodec : public Handler<std::unique_ptr<folly::IOBuf>, std::string,
                                   std::string, std::unique_ptr<folly::IOBuf>> {
 public:
  typedef typename Handler<
   std::unique_ptr<folly::IOBuf>, std::string,
   std::string, std::unique_ptr<folly::IOBuf>>::Context Context;

  void read(Context* ctx, std::unique_ptr<folly::IOBuf> buf) override {
    std::string data(...);
    ctx->fireRead(data);
  }

  folly::Future<folly::Unit> write(Context* ctx, std::string msg) override {
    auto buf = folly::IOBuf::copyBuffer(msg.data(), msg.length());
    return ctx->fireWrite(std::move(buf));
  }
};
```

在上例中`Rin = std::unique_ptr<folly::IOBuf>` `Rout = std::string` `Win = std::string` `Wout = std::unique_ptr<folly::IOBuf>`。

作为它之后的EchoHandler定义为所有模板实参都为`std::string`，和StringCodec的Rout，Win对应了起来，也就是说EchoHandler的read()实现会接收由StringCodec通过fireRead(std::string)发来的数据，write()实现在自己操作结束后需要通过fireWrite(std::string)向StringCodec发送数据。

LineBasedFrameDecoder这个Handler定义为一个`InboundHanlder Rin = folly::IOBufQueue& Rout =std::unique_ptr<folly::IOBuf> `，Rout对应了StringCodec的Rin，也就是只处理read的操作，而write数据不会经过此Handler。

## folly::EventBase

本部分会结合着上一部分pipeline的内容，按照读写流程介绍下工作流程。wangle使用folly::EventBase类来处理socket通信，EventBase底层是通过libevent实现的，主要是为了跨平台，这里主要考虑Linux上的epoll。

folly socket数据传输通过AsyncTransportWrapper类中接口实现，其中较重要的有ReadCallback和WriteCallBack类型，顾名思义是两个异步读写操作需要的回调。读回调是通知数据已经准备好，写回调是通知数据写入了缓冲区。

AcceptorThread中的Eventbase loop接受到新连接后，通过callback将客户端fd传到IOThread的Eventbase loop中。IOThread中的Acceptor为每个连接创建AsyncSocket对象，关联fd和该线程的eventbase对象，并注册epoll监听该fd的读事件（仅监听读事件，在pipeline中设置readCallback的时候注册）。当读到足够数据时，会回调readDataAvailable()传送数据，然后调用pipeline的read流程。

写操作是随时可以从pipeline的write开始进行的，如果请求写的时候连接还没建立或者缓冲区已满，由于写操作是异步实现，会马上返回，同时将该写操作插入请求队列，并向epoll注册监听写事件，当写事件再次可用时就会将所有待写数据写完。

## 和Pink的相似及不同之处

- 相同点：
  - 都是在每个I/O Thread中执行event base loop。
  - 通过一个wangle::AccpetorThread(pink::DispatchThread)接收新客户端链接，分发给wangle::IOThreadPoolExecutor(pink::WorkerThread)中的Eventbase loop处理每个连接的I/O<sup>[7]</sup>。
  - pink::PinkConn类也可以看作是简单的wangle::Pipeline，通过子类实现特殊的协议解析
  - pink::BGThread类似于wangle::CPUThreadPoolExecutor，但是BGThread不能方便得使用Eventbase
- 不同点：
  - wangle采用异步模型，读写不同步，并且可以在不同线程，pink算是同步模型
  - wangle使用更细致的异步Pipeline模式，能相对高效的解码，提高了CPU、网卡资源利用率


## 总结及问题

1. 关于EPOLLOUT事件，会在刚建立连接时（connect()客户端行为）、上一次write返回了EAGAIN然后数据发送完缓冲区可写了和通过epoll_ctl强制设置这三种情况下会触发，socket建立后在EAGAIN之前一直是可写的<sup>[6]</sup>。
2. wangle的异步实现在pingpong测试中，如果客户端忽略pingpang同步，发送缓冲区会一直保持满的状态，能大大提高了网卡和CPU利用率，也就是为什么性能会比pink好；一旦客户端使用同步pingpang方式测试，由于需要用锁来同步，性能大不如pink。
3. 和pink相比的情况下，主要差异还是同步和异步，同步异步主要都在什么场合下用？异步都会比同步好吗，况且Linux平台是用线程模拟异步实现。
4. 如果pika使用wangle这样的框架执行类似keys *的耗时长命令，就可以在解析完命令之后把任务扔给CPUThreadPool执行，在任务内也可以向客户端写数据，这样腾出了IOThread执行其它连接发来的短任务。
5. 像这样的分支预测优化平时是不是也可以用一下

```c
#define likely(x)       __builtin_expect((x),1)
#define unlikely(x)     __builtin_expect((x),0)
```

3. I/O(网络、磁盘)操作比CPU慢太多了。



## 参考文献

1. [https://code.facebook.com/posts/1661982097368498](https://code.facebook.com/posts/1661982097368498)
2. [https://code.facebook.com/posts/215466732167400/wangle-an-asynchronous-c-networking-and-rpc-library](https://code.facebook.com/posts/215466732167400/wangle-an-asynchronous-c-networking-and-rpc-library)
3. [https://github.com/facebook/wangle](https://github.com/facebook/wangle)
4. [https://github.com/facebook/folly](https://github.com/facebook/folly)
5. [https://github.com/Qihoo360/pink](https://github.com/Qihoo360/pink)
6. [https://stackoverflow.com/questions/13568858/epoll-wait-always-sets-epollout-bit/13568962](https://stackoverflow.com/questions/13568858/epoll-wait-always-sets-epollout-bit/13568962)
7. [http://baotiao.github.io/2016/11/26/concurrency](http://baotiao.github.io/2016/11/26/concurrency)
