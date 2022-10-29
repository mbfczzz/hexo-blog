## 目录

*   [Channel](#channel)

    *   [Channel工作原理](#channel工作原理)

    *   [Channel核心功能](#channel核心功能)

        *   [网络I/O操作](#网络io操作)

        *   [其他](#其他)

        *   [Channel中的Unsafe](#channel中的unsafe)

# Channel

提起Channel，我们并不陌生，在JDK NIO中也有Channel通道的概念。Channel是网络通信的载体，提供了基本的用于I/O操作的API，如：register、bind、connect、read、write、flush等。

Netty的Channel是在JDK的NIO Channel基础上进行封装的，提供了更高层次的抽象，同时屏蔽了底层Socket的复杂性，赋予了Channel更加强大的功能。

Netty为什么不使用JDK NIO原生的Channel呢？主要是基于以下几个原因：

*   JDK中的SocketChannel和ServerSocketChannel没有统一的Channel接口供业务开发者使用，对于用户而言，没有统一的操作视图，使用起来并不方便

*   JDK中的SocketChannel和ServerSocketChannel的主要职责是网络I/O操作，由于它们是SPI类接口，由具体的虚拟机厂家来提供，所以通过继承SPI功能类来扩展其功能的难度很大；直接实现SocketChannel和ServerSocketChannel，其工作量和重新开发一个新的Channel功能更类是差不多的

*   Netty的Channel需要能够跟Netty的整体架构融合在一起，例如I/O模型、基于ChannelPipeLine的定制模型，以及基于元数据描述配置化的TCP参数等，这些JDK的SocketChannel和ServerSocketChannel都没有提供，需要重新进行封装

*   自定义的Channel，功能实现更加灵活

基于以上原因，Netty自行封装了Channel接口，来代替JDK NIO原生的Channel，使得Channel能够更好地适配Netty整体框架，并且其扩展性也更强。

在Netty中，提供了多种不同的Channel实现，主要的几种实现如下：

*   FileChannel：用于文件操作

*   SelectableChannel：用于网络连接，根据网络协议不同，可以分为：

*   \*   ServerSocketChannel和SocketChannle：用于TCP协议的数据读写，分别对应服务端和客户端的通道

    *   DatagramChannel：用于UDP协议的数据读写

## Channel工作原理

*   一旦有客户端成功与服务端建立连接，将新建一个Channel与该客户端进行绑定

*   Channel从线程组NioEventloopGroup中获取一个NioEventloop，并注册到该NioEventloop，后续该Channel的生命周期内都与该NioEventloop绑定在一起

*   Channel同客户端进行网络连接、关闭和读写，生成对应的even事件，由Selector轮询到后，交给Worker线程组中的调度线程去执行

在不同的生命周期阶段，Channel会有不同的状态，并且能够在不同的状态之间进行流转和切换。

Channel的状态有四种：

*   ChannelUnregistered：已创建但还未被注册到监听器中

*   ChannelRegistered ：已注册到监听器EventLoop中

*   ChannelActive ：连接完成处于活跃状态，此时可以接收和发送数据

*   ChannelInactive ：非活跃状态，代表连接未建立或者已断开

## Channel核心功能

我们先来看一下Channel接口的顶层定义：

public interface Channel extends AttributeMap, ChannelOutboundInvoker, Comparable\<Channel> {

ChannelId id();

EventLoop eventLoop();

Channel parent();

ChannelConfig config();

boolean isOpen();

boolean isRegistered();

boolean isActive();

ChannelMetadata metadata();

SocketAddress localAddress();

SocketAddress remoteAddress();

ChannelFuture closeFuture();

boolean isWritable();

long bytesBeforeUnwritable();

long bytesBeforeWritable();

Channel.Unsafe unsafe();

ChannelPipeline pipeline();

ByteBufAllocator alloc();

Channel read();

Channel flush();

public interface Unsafe {

Handle recvBufAllocHandle();

SocketAddress localAddress();

SocketAddress remoteAddress();

void register(EventLoop var1, ChannelPromise var2);

void bind(SocketAddress var1, ChannelPromise var2);

void connect(SocketAddress var1, SocketAddress var2, ChannelPromise var3);

void disconnect(ChannelPromise var1);

void close(ChannelPromise var1);

void closeForcibly();

void deregister(ChannelPromise var1);

void beginRead();

void write(Object var1, ChannelPromise var2);

void flush();

ChannelPromise voidPromise();

ChannelOutboundBuffer outboundBuffer();

}

}

可以将Channel的功能大概分为两大类：

*   网络I/O操作：完成网络I/O的读写、连接关闭等操作

*   获取Channel通道元数据信息

### 网络I/O操作

针对网络I/O相关的方法如下：

boolean isOpen();

boolean isRegistered();

boolean isActive();

ChannelFuture closeFuture();

boolean isWritable();

long bytesBeforeUnwritable();

long bytesBeforeWritable();

Channel read();

Channel flush();

复制代码

对这些方法的介绍如下：

判断Channel通道状态：

*   isOpen()：判断当前Channel是否已经打开

*   isRegistered()：判断当Channel是否已经注册到NioEventLoop上

*   isActive()：判断当前Channel是否已经处于激活状态

操作：

*   read()：从当前的Channel中读取数据到第一个inbound缓冲区中，如果数据被成功读取，触发ChannelHandler.channelRead(ChannelHandlerContext, Object)事件，读取操作API调用完成之后，紧接着会触发ChannelHandler.channelReadComplete(ChannelHandlerContext)事件，这样业务的ChannelHandler可以决定是否需要继续读取数据。如果已经有读操作请求被挂起，则后续的读操作会被忽略。

*   flush()：将写入的数据刷入Channel

### 其他

ChannelId id();

EventLoop eventLoop();

Channel parent();

ChannelConfig config();

ChannelMetadata metadata();

SocketAddress localAddress();

SocketAddress remoteAddress();

ChannelPipeline pipeline();

复制代码

相关API介绍如下：

*   id()：在客户端连接建立后，生成Channel通道的时候会为每一个Channel分配一个唯一的ID，该ID可能的生成策略有：

*   \*   机器的MAC地址（EUI-48或者EUI-64）等可以代表全局唯一的信息

    *   当前的进程ID

    *   当前系统时间的毫秒

    *   当前系统时间纳秒数

    *   32位的随机整型数

    *   32位自增的序列数

*   eventLoop()：在上面说过Channel建立后会与EventLoopGroop中分配的一个EventLoop线程绑定，该方法就可以获取到Channel绑定的EventLoop。EventLoop本质上就是处理网络I/O读写事件的Reactor线程。在Netty中，它不仅用来处理网络事件，也可以用来执行定时任务和用户自定义NioTask任务等。

*   parent()：返回该Channel的父Channel。对于服务端的Channel而言，它的父Channel为空；对于客户端Channel而言，它的父Channel就是创建它的ServerSocketChannel

*   config()：获取当前Channel的配置信息，例如：CONNECT\_TIMEOUT\_MILLIS

*   metadata()：获取当前Channel的元数据描述信息，包括TCP参数配置等

*   localAddress()：获取当前Channel的本地绑定地址

*   remoteAddress()：获取当前Channel通信的远程Socket地址

*   pipeline()：通过pipeline()方法，可以获取到Channel的ChannelPipeline对象，ChannelPipeline也是Netty的核心组件，它可以理解为是ChannelHandler的容器，用于处理Channel的所有事件

总的来说，Channel顶层接口只定义了一些基础的核心能力，在开发过程中，比较常用的NioServerSocketChannel和NioSocketChannel这两个服务端和客户端的类均继承于：AbstractChannel。Channel的初始化核心操作都是交由该父类来完成的，并且扩充了很多Channel接口中的能力。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4961706b190540fc80737925e4bda4aa\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

在该类中维护了Channel的父Channel，ID，pipeline等重要组件，并且通过构造方法来完成初始化。通过变量定义可以看出，AbstractChannel聚合了所有Channel使用到的能力对象，由AbstractChannel提供初始化和统一的封装，如果功能和子类强相关，则定义为抽象方法由子类来实现。

### Channel中的Unsafe

我们在Channel接口中可以看到内部定义了一个Unsafe类，并且里面定义了很多与Channel功能很像的方法，那这个类到底有什么用呢？

Channel接口中Unsafe接口的定义：

public interface Unsafe {

Handle recvBufAllocHandle();

SocketAddress localAddress();

SocketAddress remoteAddress();

void register(EventLoop var1, ChannelPromise var2);

void bind(SocketAddress var1, ChannelPromise var2);

void connect(SocketAddress var1, SocketAddress var2, ChannelPromise var3);

void disconnect(ChannelPromise var1);

void close(ChannelPromise var1);

void closeForcibly();

void deregister(ChannelPromise var1);

void beginRead();

void write(Object var1, ChannelPromise var2);

void flush();

ChannelPromise voidPromise();

ChannelOutboundBuffer outboundBuffer();

}

复制代码

实际上Unsafe是Channel的一个辅助类，它不直接暴露给用户使用，它是Channel的一个辅助类，但是实际上Channel的网络I/O操作基本上都是由Unsafe负责实现的。

Unsafe继承关系如图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/5c177cccac324024940a3b0e535146a6\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

Unsafe中的核心方法介绍：

*   register()：用于将当前Unsafe对应的Channel注册到EventLoop的多路复用器上，然后调用DefaultChannelPipeLine的fireChannelRegistered方法。如果Channel被激活，则调用DefaultChannelPipeLine的fireChannelActive方法

*   bind()：主要用于绑定指定的端口，对于服务端，用于绑定监听端口，可以设置backlog参数；对于客户端，主要用于指定客户端Channel的本地绑定Socket地址

*   connect()：首先获取当前的连接状态进行缓存，然后发起连接操作，如果连接成功，则返回true；如果没连接上，服务端没有返回ACK应答，连接结果不确定，返回false；连接失败的话直接抛出I/O异常

*   finishConnect方法()：客户端接收到服务端的TCP握手应答消息，通过SocketChannel的finishConnect方法对连接结果进行判断

*   disconnect()：用于客户端或者服务器主动关闭连接

*   close()：在链路关闭之前需要先判断是否处于刷新状态，如果处于刷新状态，说明还有消息尚未发送出去，需要等到所有消息发送完成后再关闭链路，因此将关闭操作封装成Runnable稍后再执行

*   write()：将消息添加到环形发送数组中，并不是真正的写Channel，真正的写入需要调用flush方法

*   flush()方法：将发送缓冲区中待发送的消息全部写入Channel中，并发送给通信方
