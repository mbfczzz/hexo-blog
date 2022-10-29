## 目录

*   [Pipeline和ChannelHandler](#pipeline和channelhandler)

    *   [ChannelPipeline](#channelpipeline)

        *   [事件处理](#事件处理)

        *   [事件类型](#事件类型)

        *   [ChannelPipeline特性](#channelpipeline特性)

    *   [ChannelHandler](#channelhandler)

        *   [ChannelHandlerAdapter](#channelhandleradapter)

        *   [ByteToMessageDecoder](#bytetomessagedecoder)

        *   [MessageToMessageDecoder](#messagetomessagedecoder)

        *   [LengthFieldBasedFrameDecoder](#lengthfieldbasedframedecoder)

        *   [MessageToByteEncoder](#messagetobyteencoder)

        *   [MessageToMessageEncoder](#messagetomessageencoder)

        *   [LengthFieledPrepender](#lengthfieledprepender)

# Pipeline和ChannelHandler

ChannelPipeline和ChannelHandler是Netty在进行业务处理时的重要组成组件，简单来说，ChannelHandler是进行一个业务处理的处理器，而Pipeline负责将一个个的处理器串联起来，相当于一个容器，Channel中的数据会进入Pipeline，在容器中的各个处理器中按照顺序进行流转。

ChannelPipelie和ChannelHandler的关系图示：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/64979057476941388430f708ec975dd7\~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image)

在Netty中，一个请求会创建一个Channel通道，每个Channel会绑定一个Pipeline,Pipeline是一个双向链表结构，它们是一一对应的关系。ChannelHandlerContext是ChannelHandler的上下文对象，通过该对象可以获取处理器的上下文信息，如：绑定的Channel、Pipeline等。

Netty的ChannelPipeline和ChannelHandler机制类似于Servlet和Filter过滤器，这类拦截器实际上是职责链模式的一种变形，主要是为了方便事件的拦截和用户业务逻辑的定制。

Netty将Channel的数据管道抽象为ChannelPileline，消息在ChannelPileline中流动和传递，ChannelPileline持有I/O事件拦截器ChannelHandler的链表，由ChannelPileline进行I/O事件拦截和处理，可以方便地通过新增和删除ChannelHandler实现不同的业务逻辑定制，不需要对已有的ChannelHandler进行修改，能够实现对修改封闭和对扩展的支持。

## ChannelPipeline

ChannelPipeline是ChannelHandler的容器，它负责ChannelHandler的管理和事件拦截。

### 事件处理

ChannelPipeline事件处理的流程：

*   底层的SocketChannel read()方法读取ByteBuf，触发ChannelRead事件，由I/O线程NioEventLoop调用ChannelPipeline的fireChannelRead(Object msg)方法，将消息(ByteBuf)传输到ChannelPipeline中

*   消息依次被ChannelPipeline中的处理器：(例如)HeadHandler、ChannelHandler1、ChannelHandler2....TailHandler拦截和处理，在这个过程中，任何ChannelHandler都可以中断当前的流程，结束消息的传递

*   调用ChannelHandlerContext的write方法发送消息，消息依次经过：TailHandler、ChannelHandlerN...ChannelHandler2、ChannelHandler1、HeadHandler，最终被添加到消息发送缓冲区中等待刷新和发送，在此过程中也可以中断消息传递，例如当编码失败时，就需要中断消息传递，然后构造异常的Future返回

### 事件类型

Netty的事件类型分为inbound和outbound事件两大类。

**inbound事件**

inbound事件通常由I/O线程触发，例如TCP链路建立、链路关闭事件、读事件、异常通知事件等。触发Inbound事件的方法如下：

*   `ChannelHandlerContext.fireChannelRegistered()`：Channel注册事件

*   `ChannelHandlerContext.fireChannelActive()`：TCP链路建立成功，Channel激活事件

*   `ChannelHandlerContext.fireChannelRead(Object)`：读事件

*   `ChannelHandlerContext.fireChannelReadComplete()`：读操作完成通知事件

*   `ChannelHandlerContext.ExceptionCaught(Throwable)`：异常通知事件

*   `ChannelHandlerContext.fireUserEventTriggered(Object)`：用户自定义事件

*   `ChannelHandlerContext.fireChannelWritabilityChanged()`：Channel的可写状态变化通知事件

*   `ChannelHandlerContext.fireChannelInactive()`：TCP连接关闭，链路不可用通知事件

**outbound事件**

outbound事件通常是由用户主动发起的网络I/O操作，例如用户发起的连接操作，绑定操作，消息发送等，触发outbound事件的方法如下：

*   `ChannelHandlerContext.bind(SocketAddress, ChannelPromis)`：绑定本地地址事件

*   `ChannelHandlerContext.connect(SocketAddress, SocketAddress, ChannelPromis)`：连接服务器事件

*   `ChannelHandlerContext.write(Object, ChannelPromis)`：发送事件

*   `ChannelHandlerContext.flush()`：刷新事件

*   `ChannelHandlerContext.read()`：读事件

*   `ChannelHandlerContext.disconnect(ChannelPromis)`：断开连接事件

*   `ChannelHandlerContext.close(ChannelPromis)`：关闭当前Channel事件

### ChannelPipeline特性

*   支持运行态动态的添加或者删除ChannelHandler。例如在业务高峰期需要对系统做拥堵保护时，就可以根据当前的系统时间进行判断，如果处于业务高峰期，则动态地将系统拥堵保护ChannelHandler添加到当前的ChannelPipeline中，高峰期过后，就可以动态删除拥堵保护ChannelHandler

*   ChannelPipeline是线程安全的。这意味着N个业务线程可以并发地操作ChannelPipeline而不存在多线程并发问题。但是ChannelHandler却不是线程安全的，这意味着尽管ChannelPipeline是线程安全的，但是用户仍然要自己保证ChannelHandler的线程安全。

## ChannelHandler

ChannelHandler类似于Servlet的Filter过滤器，负责对I/O事件或者I/O操作进行拦截和处理，它可以选择性地进行拦截和处理自己感兴趣的事件，也可以透传和终止事件的传递。

基于ChannelHandler接口，用户可以方便地进行业务逻辑定制，例如打印日志、统一封装异常信息，性能统计和消息编解码等。

### ChannelHandlerAdapter

ChannelHandlerAdapter是handler的基类，它的所有接口实现都是事件透传，如果用户ChannelHandler关心某个事件，只需要覆盖ChannelHandlerAdapter对应的方法即可，对于不关心的方法，无需覆盖直接使用父类的方法，这样子类的代码就会非常简洁和清晰。

在Netty中的Handler可以分为以下两大类：

*   `ChannelInboundHandler`：对应上文中的inbound事件处理。主要负责读事件的逻辑处理，比如，我们在一端读到一段数据，首先要解析这段数据，然后对这些数据做一系列逻辑处理，最终把响应写到对端， 在开始组装响应之前的所有的逻辑，都可以放置在 ChannelInboundHandler 里处理，它的一个最重要的方法就是 channelRead()。可以将 ChannelInboundHandler 的逻辑处理过程与 TCP 的七层协议的解析联系起来，收到的数据一层层从物理层上升到我们的应用层。

*   `ChannelOutBoundHandler`：对应上文中的outbound事件处理。是处理写数据的逻辑，它是定义我们一端在组装完响应之后，把数据写到对端的逻辑，比如，我们封装好一个 response 对象，接下来我们有可能对这个 response 做一些其他的特殊逻辑，然后，再编码成 ByteBuf，最终写到对端，它里面最核心的一个方法就是 write()，读者可以将 ChannelOutBoundHandler 的逻辑处理过程与 TCP 的七层协议的封装过程联系起来，我们在应用层组装响应之后，通过层层协议的封装，直到最底层的物理层。

这两个接口都有默认的实现类，分别是：

*   `ChannelInboundHandlerAdapter`

*   `ChanneloutBoundHandlerAdapter`

它们分别实现两个大类接口的所有方法，默认情况下会把读写事件传播到下一个handler。

在开发中，会有一些比较常用的Netty提供的handler供我们使用，方便快速开发，例如：

*   `ByteToMessageDecoder`

*   `MessageToMessageDecoder`

*   `LengthFieledBasedFrameDecoder`

*   `MessageToByteEncoder`

*   `MessageToMessageEncoder`

*   `LengthFieledPrepender`

下面我们会一一介绍它们各自的用途。

### ByteToMessageDecoder

利用NIO进行网络编程时，往往需要将读取到的字节数组或者字节缓冲区解码为业务可以使用的POJO对象，为了方便业务将ByteBuf解码为业务POJO对象，Netty提供了ByteToMessageDecoder抽象工具解码类。

用户解码器handler继承ByteToMessageDecoder，然`后实现void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)`抽象方法即可完成ByteBuf到POJO类的转换。

由于ByteToMessageDecoder并没有考虑TCP粘包和拆包的问题， 使用该解码器时需要用户自行处理该问题。正因为如此，对于大多数场景不会直接继承ByteToMessageDecoder，而是继承其他更高级的解码器来解决粘包拆包问题。

### MessageToMessageDecoder

MessageToMessageDecoder是Netty的二次解码器，它的职责是将一个对象二次解码为其他对象。

为什么叫做二次解码器？从SocketChannel读取到的TCP数据报是ByteBuf，实际上就是字节数组，我们首先需要将ByteBuf字节数组读取处理，转换为Java对象，然后对Java对象根据某些规则进行二次解码，将其解码为另一个POJO对象。因为MessageToMessageDecoder在ByteToMessageDecoder之后，所以称之为二次解码器。

例如：以HTTP+XML协议栈为例，第一次解码往往是将字节数组解码为HttpRequest对象，然后对HttpRequest消息中的消息体字符串进行二次解码，将XML格式的字符串解码为POJO对象，这就用到了二次解码器。

在使用的时候，用户的解码器只需要实现void decode(ChannelHandlerContext ctx, I msg, List out)抽象方法即可，由于它是将一个POJO解码为另一个POJO，所以不存在粘包拆包问题。

### LengthFieldBasedFrameDecoder

该解码器是一个比较常用的解决TCP粘包拆包问题的解码器。

如何区分一个整包消息，通常由如下4种做法：

*   固定长度，例如120个字节代码一个整包消息，不足的前面补0，解码器在处理这类定长消息的时候比较简单，每次读取到指定长度的字节后进行解码

*   通过回车换行符区分消息，例如FTP协议，这类区分消息的方式多用于文本协议

*   通过分隔符区分整包消息

*   通过指定长度来标识整包消息

由于TCP本身存在粘包和拆包问题，所以通常情况下必须自己处理半包消息。利用LengthFieldBasedFrameDecoder解码器可以自动解决半包问题，通常的用法如下：

pipeline.addLast(new LengthFieldBasedFrameDecoder(1024, 0, 4));

pipeline.addLast(new StringDecoder());

复制代码

将LengthFieldBasedFrameDecoder解码器加入ChannelPipeline，指定正确的参数组合，它可以将Netty的ByteBuf解码成单个的整包消息，后面的业务解码器拿到的就是完整的数据报，正常进行解码即可，不需要再考虑半包问题，方便了业务消息的解码。

### MessageToByteEncoder

MessageToByteEncoder负责将POJO对象编码成ByteBuf，用户的编码器继承MessageToByteEncoder，实现`void encode(ChannelHandlerContext ctx, I msg, ByteBuf out)`接口,示例代码：

public class IntegerEncoder extends MessageToByteEncoder\<Integer> {

@Override

protected void encode(ChannelHandlerContext channelHandlerContext, Integer integer, ByteBuf byteBuf) throws Exception {

byteBuf.writeInt(integer);

}

}

复制代码

### MessageToMessageEncoder

将一个POJO对象编码为另一个对象，以HTTP+XML协议为例，它的一种实现发送是：先将POJO对象编码为XML字符串，再将字符串编码为HTTP请求或者应答消息。对于复杂协议，往往需要经历多次编码，为了便于功能扩展，可以通过多个编码器组合来实现相关功能。

用户的解码器继承MessageToMessageEncoder解码器，实现`void encode(ChannelHandlerContext channelHandlerContext, Integer integer, List<Object> list)`方法列表。示例代码如下：

public class IntegerToStringEncoder extends MessageToMessageEncoder\<Integer> {

@Override

protected void encode(ChannelHandlerContext channelHandlerContext, Integer integer, List\<Object> list) throws Exception {

list.add(integer.toString());

}

}

复制代码

### LengthFieledPrepender

如果协议中的第一个字段为长度字段，Netty提供了LengthFieledPrepender编码器， 它可以计算当前待发送的消息的二进制字节长度，将该长度添加到ByteBuf的缓冲区头中。

例如编码前的字符为"HELLO,WORLD"占12字节，通过LengthFieledPrepender编码后，消息组成为消息长度字段+消息字符串本身，总的占14个字节。
