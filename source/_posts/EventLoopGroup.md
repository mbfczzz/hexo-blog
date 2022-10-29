## 目录

*   [EventLoopGroup](#eventloopgroup)

    *   [前言](#前言)

    *   [三种Reactor线程模型](#三种reactor线程模型)

    *   [Netty线程模型最佳实践](#netty线程模型最佳实践)

# EventLoopGroup

## 前言

线程模型是Netty框架的核心，模型设计的好坏决定了框架的性能、并发量和安全性等架构质量。

Netty的线程模型被精心的设计，既提升了框架的并发性能，又在很大程度避免锁，局部实现了无锁化设计。

因此这篇文章将介绍Netty的线程模型，看看它的线程模型是如何设计用于支持高并发高性能的。

## 三种Reactor线程模型

提到线程模型，比较经典的是Reactor线程模型，尽管不同的NIO框架对Reactor模型的实现有所差异，但是本质上还是遵循了Reactor的基础线程模型。

**什么是Reactor线程模型？**

Reactor线程模型是对于传统的I/O线程模型的一种优化。

传统的I/O线程模型采用阻塞I/O来获取输入流数据，并且每个连接都需要独立的线程完成数据的输入、业务处理、数据返回等一个完整的操作链路。这种模型在高并发场景下，有两个比较明显的缺点：

*   每个连接都需要创建一个对应线程，线程大量创建占用大量的服务器资源

*   线程没有数据可读情况下的阻塞会对性能造成很大的影响

Reactor线程模型为了解决这两个问题，提供了以下解决方案：

*   基于I/O多路复用：多个客户端连接共用一个阻塞对象，应用程序只需要在一个阻塞对象等待，无需阻塞等待所有连接。当某个连接有新的数据可以处理时，通过事件驱动通知应用程序，线程从阻塞状态返回，开始进行业务处理

*   基于线程池技术减少线程创建：基于线程池，不必再为每一个连接创建线程，将连接完成后的业务处理分配给线程池进行调度

**Reactor线程模型图：**

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6d892cee9b44493c987be58a4af86bac\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

Reactor线程模型

Reactor在一个单独的线程中进行，负责监听和分发事件。

Reactor的两个核心组件：

*   EventDispatch：监听和分发事件，分发给适当的处理程序来对IO事件做出反应

*   handlers是处理程序执行IO事件要完成的实际事件，Reactor 通过调度适当地处理程序来响应I/O事件，处理程序执行非阻塞操作。

有

Reactor模式使用I/O复用监听事件，收到事件后，分发给某个线程去处理，这也是能进行网络高并发处理的关键。

Netty的线程模型不是一成不变的，它实际取决于用户的启动参数配置。通过设置不同的启动参数，Netty可以同时支持Reactor单线程模型、多线程模型和主从Reactor多线程模型。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fa5766f4f69443ffa74fc932f9c03369\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp)

Netty线程模型

netty线程模型抽象出两组线程池：

*   BossGroup：专门负责接收客户端的连接

*   WorkerGroup：专门负责网络的读写，业务处理

两者的类型都是NioEventLoopGroup，NioEventLoopGroup相当于一个事件循环组，这个组中有多个事件循环，每个事件循环是NioEventLoop。

NioEventLoop表示一个不断循环的执行处理任务的线程，每个NioEventLoop都有一个selector，用于监听绑定在其上的socket网络通信。

NioEventLoopGroup可以有多个线程，即可以有多个NioEventLoop（数量可以指定）

每个Boss NioEventLoop执行的步骤：

*   轮询accept事件

*   处理accept事件，与client事件建立连接，生成NIoSocketChannel，并将其注册到某个worker NIoEventLoop上的selector上

*   处理任务队列的任务，即runTasks

每个Worker NioEventLoop循环执行的步骤：

*   轮询read/write事件

*   处理IO事件，在对应的NIoSocketChannel 上进行处理

*   处理任务队列的任务，即runTasks

每个Worker NioEventLoop处理业务时，会通过pipeline（管道），pipeline中包含了channel，管道中维护了很多的处理器。

通过调整线程池的线程个数、是否共享线程池等方式，Netty的Reactor线程模型可以在单线程、多线程和主从线程模型之间切换，这种灵活配置方式可以最大程度满足不同用户的个性化定制。

为了尽可能提升性能，Netty在很大地方进行了无锁化设计，例如在I/O线程内部进行串行操作，避免多线程竞争导致的性能下降问题。表面上看，串行设计似乎CPU利用率不高，并发程度不够，但是通过调整NIO线程池的线程参数，可以同时启动多个串行化的线程并行运行，这种局部无锁化的串行线程设计相比一个队列多个工作线程的模型更优。

Netty的NioEventLoop读取到消息之后，直接调用ChannelPipeline的fireChannelRead(Object msg)。只要用户不主动切换线程，一直都是由NioEventLoop调用用户的Handler，期间不进行线程切换。这种串行化处理方式避免了多线程操作导致的锁的竞争。从性能角度看是最优的。

Netty的NioEventLoop并不是一个纯粹的I/O线程，它除了负责I/O的读写之外，还可以处理以下两类任务：

*   系统Task：通过调用NioEventLoop的execute(Runnable task)方法实现，Netty中有很多系统Task，主要用于：当I/O线程和用户线程同时操作网络资源时，为了防止并发操作导致的锁竞争，将用户线程的操作封装成Task放入消息队列中，由I/O线程负责执行，这样就实现了无锁化。

*   定时Task：通过调用NioEventLoop的schedule(Runnable command, long delay, TimeUnit unit)方法实现

## Netty线程模型最佳实践

*   创建两个NioEventLoopGroup，用于逻辑分离NIO Acceptor和NIO I/O线程

    NioEventLoopGroup bossGroup = new NioEventLoopGroup();&#x20;

    NioEventLoopGroup workerGroup = new NioEventLoopGroup();

    复制代码

BossGroup和WorkerGroup的线程（NioEventLoop）数量：默认是CPU核心数的两倍。

可以通过构造函数传入线程数量指定线程池的线程数量。

*   尽量不要在ChannleHandler中启动用户线程

*   解码要放在NIO线程调用的解码handler中进行，不要切换到用户线程中完成消息的解码

*   如果业务处理逻辑很简单，没有复杂的业务逻辑计算，没有可能会导致线程被阻塞的磁盘操作、数据库操作、网络操作等，可以直接在NIO线程上完成业务逻辑编排，不需要切换到用户线程。

*   如果业务逻辑处理复杂，不要在NIO线程上完成，建议将解码后的POJO消息封装成Task，派发到业务线程池中由业务线程执行，以保证NIO线程尽快被释放，处理其他I/O操作

推荐的线程数量计算公式有两种：

*   公式一：线程数量=（线程总时间/瓶颈资源时间）X 瓶颈资源的线程并行数

*   公式二：QPS=1000/线程总时间 X 线程数

由于用户场景的不同，对于一些复杂的系统，实际上很难计算出最优的线程配置，只能根据测试数据和用户场景，结合公式给出一个相对合理的范围，然后对范围内的数据进行性能测试，选择相对最优值。
