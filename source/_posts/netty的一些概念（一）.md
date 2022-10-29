## 目录

*   [netty的一些概念（一）](#netty的一些概念一)

    *   [协议](#协议)

    *   [BIO/NIO/AIO](#bionioaio)

        *   *   [1、BIO（同步阻塞IO）](#1bio同步阻塞io)

            *   [2、NIO（同步非阻塞IO）](#2nio同步非阻塞io)

            *   [3、AIO（异步非阻塞IO）](#3aio异步非阻塞io)

    *   [Netty架构](#netty架构)

        *   [1. Core 核心层](#1-core-核心层)

        *   [2. Protocol Support 协议支持层](#2-protocol-support-协议支持层)

        *   [3. Transport Service 传输服务层](#3-transport-service-传输服务层)

    *   [二、Netty 逻辑架构](#二netty-逻辑架构)

        *   [1. 网络通信层](#1-网络通信层)

        *   [2. 事件调度层](#2-事件调度层)

        *   [3. 服务编排层](#3-服务编排层)

    *   [三、组件关系梳理](#三组件关系梳理)

    *   [四、Netty 源码结构](#四netty-源码结构)

        *   *   [Core 核心层模块](#core-核心层模块)

            *   [Protocol Support 协议支持层模块](#protocol-support-协议支持层模块)

            *   [Transport Service 传输服务层模块](#transport-service-传输服务层模块)

# netty的一些概念（一）

## 协议

*   网络协议为计算机网络中进行数据交换而建立的规则、标准或约定的集合 例如平时我们签订的合同，主要用于约束两方的一些行为以及必须遵守的规则和约定，网络协议亦是如此，如果想要双方能够达成通信，必须约束双方，如果两个终端使用的字符集不一样，那么两个终端就不能识别对方发送的消息，所以无法完成通信，为了能进行通信，规定每个终端都要将各自字符集中的字符先变换为标准字符集的字符后，才进入网络传送，到达目的终端之后，再变换为该终端字符集的字符

## BIO/NIO/AIO

#### 1、BIO（同步阻塞IO）

*   服务端创建一个**ServerSocket**，客户端就有一个Socket去链接这个ServerSocket，然后ServerSocket接收到客户端的Socket请求之后就会建立一个专属的**Socket+线程**去和**客户端的Socket**去通信（长时间维护）

*   **同步阻塞通信**：客户端发送一个请求，服务端Socket就进行处理后返回，响应必须是等待处理完毕之后才会返回的，在这之前是什么也做不了。

*   缺点：每次一个客户端的接入就会有一个线程+Socket对其进行通信，这会导致客户端接入太多，服务端线程过多，导致崩溃。

*

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/7/17153ff087c520ea\~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

#### 2、NIO（同步非阻塞IO）

*   **Buffer**（缓冲区）：**channel将数据写入Buffer**，然后从Buffer中读取数据，包括int、Long、CharBuffer等多种数据类型。

*   **channel**：通过channel进行数据的读写

*   **selector**（多路复用器）：**selector会轮询channel**，如果某个channel中发生了数据请求，selector就会将通过SelectionKey会哦去有数据请求的channel，进行IO操作。一个Selector（一个线程）可以轮询上千万个channel，也就是客户端可以接入的数量激增。

*   通过一个线程轮询大量的channel，每次获取一批有事件的channel，然后**对每个请求启动一个线程**进行处理，并设置一个线程池，当线程处理完毕以后，就回收线程，就不会像BIO需要一直维持为每个客户端创建的**Socket+线程**。

*

![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2020/4/7/17153ff2daf38c34\~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.image)

#### 3、AIO（异步非阻塞IO）

*   基于Proactor模型，每个连接发送的请求，都会绑定一个Buffer，然后**通知操作系统异步的完成读操作**，此时程序可以去干别的事，操作系统完成数据的读取之后，就会回调接口，将读出的数据给你。

*   将数据进行处理，接着将结果返回

*   写数据的时候也是**给操作系统一个buffer**，让操作系统获取数据完成写操作。

## Netty架构

在 `Netty` 的官网中，给出了一张图，图片如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f5db436d44a47c58cc97e292cfdb4bc\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

上图摘自 [Netty 官网首页](https://link.juejin.cn?target=https://netty.io/ "Netty 官网首页")。

这就是 Netty 的模块划分图，可以清晰的看出，一共分为三个模块：

*   `Core 核心层`；

*   `Protocol Support 协议支持层`；

*   `Transport Services 传输服务层`。

可以看出，Netty 的模块设计具备较高的**通用性和可扩展性**。

### 1. Core 核心层

`Core 核心层`包含了 Netty 最为核心的功能，提供了底层网络通信的通用抽象和实现，包括可**扩展的事件模型、通用的通信 API、支持零拷贝的 ByteBuf 等**。

### 2. Protocol Support 协议支持层

协议支持层基本上覆盖了主流协议的编解码实现，如 `HTTP、SSL、Protobuf、压缩、大文件传输、WebSocket、文本、二进制`等主流协议，此外 Netty 还支持自定义应用层协议。

Netty 丰富的协议支持降低了用户的开发成本，基于 Netty 我们可以快速开发 HTTP、WebSocket 等服务。

### 3. Transport Service 传输服务层

传输服务层提供了网络传输能力的定义和实现方法。它支持 Socket、HTTP 隧道、虚拟机管道等传输方式。

Netty 对 TCP、UDP 等数据传输做了抽象和封装，用户可以更聚焦在业务逻辑实现上，而不必关系底层数据传输的细节。

## 二、Netty 逻辑架构

下图是 Netty 的逻辑处理架构。Netty 的逻辑处理架构为典型网络分层架构设计，共分为`网络通信层、事件调度层、服务编排层`，每一层各司其职。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3f8cfaef5caf4855b1fcf59d3f769565\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

### 1. 网络通信层

**网络通信层的职责是执行网络 I/O 的操作。它支持多种网络协议和 I/O 模型的连接操作。当网络数据读取到内核缓冲区后，会触发各种网络事件，这些网络事件会分发给事件调度层进行处理**。

网络事件有连接事件、读事件、写事件等。

网络通信层的**核心组件**包含 **BootStrap、ServerBootStrap、Channel** 三个组件。

*   **BootStrap & ServerBootStrap**

`Bootstrap` 是“引导”的意思，它主要负责整个 Netty 程序的启动、初始化、服务器连接等过程，它相当于一条主线，串联了 Netty 的其他核心组件。

Netty 中的引导器共分为两种类型：一个为**用于客户端引导的 Bootstrap**，另一个为**用于服务端引导的 ServerBootStrap**，它们都继承自抽象类 `AbstractBootstrap`。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/833253fcd11c4f3f8bc8cf8f7aa02deb\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

`Bootstrap` 和 `ServerBootStrap` 十分相似，两者非常重要的区别在于 `Bootstrap` 可用于连接远端服务器，只绑定一个 `EventLoopGroup`。而 `ServerBootStrap` 则用于服务端启动绑定本地端口，会绑定两个 `EventLoopGroup`，这两个 EventLoopGroup 通常称为 Boss 和 Worker。

ServerBootStrap 中的 Boss 和 Worker 是什么角色呢？它们之间又是什么关系？这里的 Boss 和 Worker 可以理解为“老板”和“员工”的关系。每个服务器中都会有一个 Boss，也会有一群做事情的 Worker。Boss 会不停地接收新的连接，然后将连接分配给一个个 Worker 处理连接。

`Boss` 对应 `Reactor` 模型中的 `MainReactor`，`Worker` 对应 `Reactor` 模型的 `SubReactor`。

这里放一下 `Reactor` 的整体流程图。来源于：[gee.cs.oswego.edu/dl/cpjslide…](https://link.juejin.cn?target=https://gee.cs.oswego.edu/dl/cpjslides/nio.pdf "gee.cs.oswego.edu/dl/cpjslide…")

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/731e45eadd614f73a6909552e77dd33d\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

总结来说就是客户端使用 `Bootstrap` 引导类，服务端使用 `ServerBootStrap` 引导类。

有了 Bootstrap 组件，我们可以更加方便地配置和启动 Netty 应用程序，它是整个 Netty 的入口，串接了 Netty 所有核心组件的初始化工作。

*   **Channel**

`Channel` 的字面意思是“通道”，它是网络通信的载体。

`Channel` 提供了基本的 API 用于网络 I/O 操作，如 `register、bind、connect、read、write、flush` 等。

Netty 自己实现的 Channel 是以 JDK NIO Channel 为基础的，相比较于 JDK NIO，Netty 的 Channel 提供了更高层次的抽象，同时屏蔽了底层 Socket 的复杂性，赋予了 Channel 更加强大的功能，你在使用 Netty 时基本不需要再与 Java Socket 类直接打交道。

下图是 Channel 家族的图谱。`AbstractChannel` 是整个家族的基类，派生出 `AbstractNioChannel、AbstractOioChannel`、，每一种都代表了不同的 I/O 模型和协议类型。常用的 Channel 实现类有：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/81226d36b08b43bea0a42d8baf8babfc\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   NioServerSocketChannel 异步 TCP 服务端。

*   NioSocketChannel 异步 TCP 客户端。

*   OioServerSocketChannel 同步 TCP 服务端。

*   OioSocketChannel 同步 TCP 客户端。

*   NioDatagramChannel 异步 UDP 连接。

*   OioDatagramChannel 同步 UDP 连接。

注意：

*   Nio 为前缀的，代表使用的是 NIO 模型，所以是异步的。

*   Oio 为前缀的，代表使用的是 BIO 模型，所以是同步的。

Channel 会有多种状态，如**连接建立、连接注册、数据读写、连接销毁**等。

常见的状态对应事件如下：

| 事件                  | 说明                               |
| ------------------- | -------------------------------- |
| channelRegistered   | Channel 创建后被注册到 EventLoop 上      |
| channelUnregistered | Channel 创建后未注册或者从 EventLoop 取消注册 |
| channelActive       | Channel 处于就绪状态，可以被读写             |
| channelInactive     | Channel 处于非就绪状态                  |
| channelRead         | Channel 可以从远端读取到数据               |
| channelReadComplete | Channel 读取数据完成                   |

总结一下：

*   BootStrap 和 ServerBootStrap 分别负责客户端和服务端的启动，它们是非常强大的辅助工具类，串联了 Netty 的系列核心组件；

*   Channel 是网络通信的载体，提供了与底层 Socket 交互的能力。

### 2. 事件调度层

**事件调度层的职责是通过 Reactor 线程模型对各类事件进行聚合处理，通过 Selector 主循环线程集成多种事件（ I/O 事件、信号事件、定时事件等），实际的业务处理逻辑是交由服务编排层中相关的 Handler 完成**。

事件调度层的**核心组件**包括 **EventLoopGroup、EventLoop**。

*   **EventLoopGroup & EventLoop**

EventLoopGroup 本质是一个线程池，主要负责接收 I/O 请求，并分配线程执行处理请求。

EventLoopGroup 类图如下：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f171c5963b064ee48424931080803266\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

我们发现它继承了 Executor 类，可以证明它是一个线程池。

那这就说明，由它管理的一个个 EventLoop，就是一个个线程，由 EventLoopGroup 负责分配 EventLoop 进行处理事件。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a4e4c4bf9654c53a8ab2502664aac93\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

从上图中，我们可以总结出 EventLoopGroup、EventLoop、Channel 的几点关系。

1.  一个 EventLoopGroup 往往包含一个或者多个 EventLoop。EventLoop 用于处理 Channel 生命周期内的所有 I/O 事件，如 accept、connect、read、write 等 I/O 事件。

2.  EventLoop 同一时间会与一个线程绑定，每个 EventLoop 负责处理多个 Channel。

3.  每新建一个 Channel，EventLoopGroup 会选择一个 EventLoop 与其绑定。该 Channel 在生命周期内都可以对 EventLoop 进行多次绑定和解绑。

下图是 EventLoopGroup 的家族图谱。可以看出 Netty 提供了 EventLoopGroup 的多种实现，而且 EventLoop 则是 EventLoopGroup 的子接口，所以也可以把 EventLoop 理解为 EventLoopGroup，但是它只包含一个 EventLoop 。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/50f56d8b8ab04768a6a92f6d1611f1b8\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

`EventLoopGroup` 的实现类是 `NioEventLoopGroup`，`NioEventLoopGroup` 也是 Netty 中最被推荐使用的线程模型。

NioEventLoopGroup 继承于 MultithreadEventLoopGroup，是基于 NIO 模型开发的，可以把 NioEventLoopGroup 理解为一个线程池，每个线程负责处理多个 Channel，而同一个 Channel 只会对应一个线程。

EventLoopGroup 是 Netty 的核心处理引擎，那么 EventLoopGroup 和 Reactor 线程模型到底是什么关系呢？

其实 EventLoopGroup 是 Netty Reactor 线程模型的具体实现方式，Netty 通过创建不同的 EventLoopGroup 参数配置，就可以支持 Reactor 的三种线程模型：

1.  **单线程模型**：EventLoopGroup 只包含`一个 EventLoop`，Boss 和 Worker 使用同一个EventLoopGroup；

2.  **多线程模型**：EventLoopGroup 包含`多个 EventLoop`，Boss 和 Worker 使用同一个EventLoopGroup；

3.  **主从多线程模型**：EventLoopGroup 包含多个 EventLoop，Boss 是主 Reactor，Worker 是从 Reactor，它们分别使用不同的 EventLoopGroup，主 Reactor 负责新的网络连接 Channel 创建，然后把 Channel 注册到从 Reactor。

### 3. 服务编排层

**服务编排层的职责是负责组装各类服务，它是 Netty 的核心处理链，用以实现网络事件的动态编排和有序传播**。

服务编排层的**核心组件**包括 **ChannelPipeline**、**ChannelHandler、ChannelHandlerContext**。

*   **ChannelPipeline**

`ChannelPipeline` 是 Netty 的核心编排组件，**负责组装各种 ChannelHandler**，实际数据的编解码以及加工处理操作都是由 ChannelHandler 完成的。

ChannelPipeline 可以理解为**ChannelHandler 的实例列表**——内部通过双向链表将不同的 ChannelHandler 链接在一起。当 I/O 读写事件触发时，ChannelPipeline 会依次调用 ChannelHandler 列表对 Channel 的数据进行拦截和处理。

`ChannelPipeline 是线程安全的`，因为每一个新的 Channel 都会对应绑定一个新的 ChannelPipeline。一个 ChannelPipeline 关联一个 EventLoop，一个 EventLoop 仅会绑定一个线程。

ChannelPipeline、ChannelHandler 都是高度可定制的组件。开发者可以通过这两个核心组件掌握对 Channel 数据操作的控制权。下面我们看一下 ChannelPipeline 的结构图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3a386eede45142f79075eed075a3447c\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

从上图可以看出，ChannelPipeline 中包含入站 ChannelInboundHandler 和出站 ChannelOutboundHandler 两种处理器，我们结合客户端和服务端的数据收发流程来理解 Netty 的这两个概念。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/94ef10fa0234401196cbe8f16abfd40c\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

客户端和服务端都有各自的 ChannelPipeline。以客户端为例，数据从客户端发向服务端，该过程称为**出站**，反之则称为**入站**。数据入站会由一系列 InBoundHandler 处理，然后再以相反方向的 OutBoundHandler 处理后完成出站。

我们经常使用的编码 Encoder 是出站操作，解码 Decoder 是入站操作。服务端接收到客户端数据后，需要先经过 Decoder 入站处理后，再通过 Encoder 出站通知客户端。所以客户端和服务端一次完整的请求应答过程可以分为三个步骤：客户端出站（请求数据）、服务端入站（解析数据并执行业务逻辑）、服务端出站（响应结果）。

*   **ChannelHandler & ChannelHandlerContext**

下图描述了 Channel 与 ChannelPipeline 的关系，从图中可以看出，每创建一个 Channel 都会绑定一个新的 ChannelPipeline，ChannelPipeline 中每加入一个 ChannelHandler 都会绑定一个 ChannelHandlerContext。

由此可见，ChannelPipeline、ChannelHandlerContext、ChannelHandler 三个组件的关系是密切相关的。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/abccbee47d484fd8a066690aaa39d4e7\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

ChannelHandlerContext 用于保存 ChannelHandler 上下文，通过 ChannelHandlerContext 我们可以知道 ChannelPipeline 和 ChannelHandler 的关联关系。

ChannelHandlerContext 可以实现 ChannelHandler 之间的交互，ChannelHandlerContext 包含了 ChannelHandler 生命周期的所有事件，如 connect、bind、read、flush、write、close 等。

此外，你可以试想这样一个场景，如果每个 ChannelHandler 都有一些通用的逻辑需要实现，没有 ChannelHandlerContext 这层模型抽象，你是不是需要写很多相同的代码呢？

## 三、组件关系梳理

Netty 组件交互流程图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f001f6025ba3460c83710d0b0b8f9ccd\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

*   服务端启动初始化时有 `Boss EventLoopGroup` 和 `Worker EventLoopGroup` 两个组件，其中 Boss 负责监听网络连接事件。当有新的网络连接事件到达时，则将 Channel 注册到 Worker EventLoopGroup。

*   Worker EventLoopGroup 会被分配一个 EventLoop 负责处理该 Channel 的读写事件。每个 EventLoop 都是单线程的，通过 Selector 进行事件循环。

*   当客户端发起 I/O 读写事件时，服务端 EventLoop 会进行数据的读取，然后通过 Pipeline 触发各种监听器进行数据的加工处理。

*   客户端数据会被传递到 ChannelPipeline 的第一个 ChannelInboundHandler 中，数据处理完成后，将加工完成的数据传递给下一个 ChannelInboundHandler。

*   当数据写回客户端时，会将处理结果在 ChannelPipeline 的 ChannelOutboundHandler 中传播，最后到达客户端。

## 四、Netty 源码结构

[Netty 源码](https://link.juejin.cn?target=https://github.com/netty/netty "Netty 源码")

Netty 源码结构和 Netty 的模块划分大体相符合：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e06c191b201542229539f50fd4bc9de3\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

#### Core 核心层模块

**netty-common**模块是 Netty 的核心基础包，提供了丰富的工具类，其他模块都需要依赖它。在 common 模块中，常用的包括**通用工具类**和**自定义并发包**。

*   通用工具类：比如定时器工具 TimerTask、时间轮 HashedWheelTimer 等。

*   自定义并发包：比如异步模型 Future & Promise、相比 JDK 增强的 FastThreadLocal 等。

在 **netty-buffer 模块中**Netty自己实现了的一个更加完备的 **ByteBuf 工具类**，用于网络通信中的数据载体。

由于人性化的 Buffer API 设计，它已经成为 Java ByteBuffer 的完美替代品。ByteBuf 的动态性设计不仅解决了 ByteBuffer 长度固定造成的内存浪费问题，而且更安全地更改了 Buffer 的容量。此外 Netty 针对 ByteBuf 做了很多优化，例如缓存池化、减少数据拷贝的 CompositeByteBuf 等。

**netty-resover**模块主要提供了一些有关**基础设施**的解析工具，包括 IP Address、Hostname、DNS 等。

#### Protocol Support 协议支持层模块

**netty-codec**模块主要负责编解码工作，通过编解码实现原始字节数据与业务实体对象之间的相互转化。

如下图所示，Netty 支持了大多数业界主流协议的编解码器，如 HTTP、HTTP2、Redis、XML 等，为开发者节省了大量的精力。此外该模块提供了抽象的编解码类 ByteToMessageDecoder 和 MessageToByteEncoder，通过继承这两个类我们可以轻松实现自定义的编解码逻辑。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1223bbcec83e462487adf0fa213b9238\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

**netty-handler**模块主要负责数据处理工作。Netty 中关于数据处理的部分，本质上是一串有序 handler 的集合。netty-handler 模块提供了开箱即用的 ChannelHandler 实现类，例如日志、IP 过滤、流量整形等，如果你需要这些功能，仅需在 pipeline 中加入相应的 ChannelHandler 即可。

#### Transport Service 传输服务层模块

netty-transport 模块可以说是 Netty 提供数据**处理和传输的核心模块**。该模块提供了很多非常重要的接口，如 Bootstrap、Channel、ChannelHandler、EventLoop、EventLoopGroup、ChannelPipeline 等。

其中 Bootstrap 负责客户端或服务端的启动工作，包括创建、初始化 Channel 等；EventLoop 负责向注册的 Channel 发起 I/O 读写操作；ChannelPipeline 负责 ChannelHandler 的有序编排，这些组件在介绍 Netty 逻辑架构的时候都有所涉及。
