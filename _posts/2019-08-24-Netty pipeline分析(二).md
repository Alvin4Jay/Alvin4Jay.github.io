---
layout:     post
title:      Netty pipeline分析(二)
subtitle:   ChannelPipeline/ChannelHandlerContext
date:       2019-08-24
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Netty
---

# Netty pipeline分析(二)

[Netty pipeline分析(一)](https://xuanjian1992.top/2019/08/23/Netty-pipeline%E5%88%86%E6%9E%90(%E4%B8%80)/)这篇文章分析了pipeline在Netty中所处的角色，像是一条流水线，控制着字节流的读写。本文在这个基础上继续深挖pipeline在事件传播、异常传播等方面的细节。主要分为以下几点:

- Netty中的Unsafe
- inbound事件的传播
- outbound事件的传播
- 异常事件的传播

### 一、Netty中的Unsafe

pipeline中所有的IO操作最终都会由Unsafe完成，因此在分析pipeline事件传播方面的细节之前，先介绍Unsafe的作用。

##### 1.Unsafe接口

Unsafe接口在Channel接口中定义，属于Channel的内部类，表明Unsafe和Channel密切相关。

```java
interface Unsafe {
    RecvByteBufAllocator.Handle recvBufAllocHandle();

    SocketAddress localAddress();
    SocketAddress remoteAddress();

    void register(EventLoop eventLoop, ChannelPromise promise);
    void bind(SocketAddress localAddress, ChannelPromise promise);
    void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
    void disconnect(ChannelPromise promise);
    void close(ChannelPromise promise);
    void closeForcibly();
    void deregister(ChannelPromise promise);
    void beginRead();
    void write(Object msg, ChannelPromise promise);
    void flush();

    ChannelPromise voidPromise();
    ChannelOutboundBuffer outboundBuffer();
}
```

从Unsafe接口中定义的方法可以看出，Unsafe接口的主要功能是接收数据时分配内存、获取本地与远端地址、将channel注册到事件循环、绑定端口、socket的连接和关闭、socket的读写等，这些操作都和JDK底层相关。

##### 2.Unsafe继承结构

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20pipeline-3.jpg?x-oss-process=style/markdown-pic)

- Unsafe作为顶层接口，AbstractUnsafe是其抽象实现，子类必须继承AbstractUnsafe。

- 针对基于Selector的NIO读写场景，增加了NioUnsafe接口，其扩展了Unsafe接口，并增加了以下方法:

  ```java
  public interface NioUnsafe extends Unsafe {
      SelectableChannel ch();
      void finishConnect();
      void read();
      void forceFlush();
  }
  ```

  NioUnsafe增加了获取底层JDK NIO SelectableChannel的方法，同时定义了从SelectableChannel读取数据的方法。

- AbstractNioUnsafe是基于Selector的NIO读写场景下的Unsafe抽象实现，能够通过其外部类AbstractNioChannel的相关方法如selectionKey()、javaChannel()等方法获得SelectionKey、SelectableChannel等。

- NioByteUnsafe和NioSocketChannelUnsafe作为客户端NioSocketChannel的Unsafe实现，实现了channel基本的IO操作，如数据的读写，这些都与JDK底层相关。

- NioMessageUnsafe和NioByteUnsafe是处在同一层次的实现，用于读取新连接。Netty将一个新连接的建立也当作一个IO操作来处理，这里的message的含义可以当作是一个SelectableChannel，读的意思就是accept一个SelectableChannel。

##### 3.Unsafe的读写操作

从以上继承结构来看，可以总结出两种类型的Unsafe实现，一种是与连接的字节数据读写相关的NioByteUnsafe，一种是与新连接建立操作相关的NioMessageUnsafe。

> NioByteUnsafe中的读：委托到外部类NioSocketChannel

```java
protected int doReadBytes(ByteBuf byteBuf) throws Exception {
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    allocHandle.attemptedBytesRead(byteBuf.writableBytes());
    return byteBuf.writeBytes(javaChannel(), allocHandle.attemptedBytesRead());
}
```

可以看到，doReadBytes()方法从JDK  SocketChannel中读取字节数据到Netty的ByteBuf中。

> NioByteUnsafe中的写：委托到外部类NioSocketChannel

```java
protected int doWriteBytes(ByteBuf buf) throws Exception {
    final int expectedWrittenBytes = buf.readableBytes();
    return buf.readBytes(javaChannel(), expectedWrittenBytes);
}
```

doWriteBytes()方法将Netty ByteBuf中的字节数据写出到JDK SocketChannel中。

> NioMessageUnsafe中的读：委托到外部类NioServerSocketChannel

```java
protected int doReadMessages(List<Object> buf) throws Exception {
    // 调用JDK ServerSocketChannel.accept()方法接受连接
    SocketChannel ch = javaChannel().accept(); 

    try {
        if (ch != null) {
            // 将JDK连接的SocketChannel封装成Netty NioSocketChannel
            buf.add(new NioSocketChannel(this, ch)); 
            return 1; // 成功接收连接，返回1
        }
    } catch (Throwable t) {
				// 省略代码...
    }

    return 0;
}
```

NioMessageUnsafe的读操作中调用JDK ServerSocketChannel的accept()方法，接收一条新连接，并包装成Netty NioSocketChannel。

### 二、inbound事件的传播

##### 1.inbound事件

首先看ChannelInboundHandler接口的定义:

```java
public interface ChannelInboundHandler extends ChannelHandler {
    void channelRegistered(ChannelHandlerContext ctx) throws Exception;
    void channelUnregistered(ChannelHandlerContext ctx) throws Exception;
    void channelActive(ChannelHandlerContext ctx) throws Exception;
    void channelInactive(ChannelHandlerContext ctx) throws Exception;
    void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception;
    void channelReadComplete(ChannelHandlerContext ctx) throws Exception;
    void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception;
    void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception;
    void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception;
}
```

从ChannelInboundHandler接口的定义可以看出，inbound事件包含channelRegistered、channelUnregistered、channelActive、channelInactive、channelRead、channelReadComplete、exceptionCaught等事件，而且当比如channelRead()方法被调用时，该事件已经发生，handler是被动触发的。

##### 2.channelRead事件的传播

下面以channelRead事件的传播为例，来分析inbound事件的传播细节。先看服务端ServerBootstrap的配置:

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .childOption(ChannelOption.TCP_NODELAY, true)
        .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
                ch.pipeline().addLast(new InBoundHandlerA());
                ch.pipeline().addLast(new InBoundHandlerC());
                ch.pipeline().addLast(new InBoundHandlerB());
            }
        });
```

可知服务端在接收新连接时会为客户端channel pipeline添加InBoundHandlerA、InBoundHandlerC、InBoundHandlerB三个ChannelInboundHandler。

```java
public class InBoundHandlerA extends ChannelInboundHandlerAdapter {
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("InBoundHandlerA: " + msg);
        ctx.fireChannelRead(msg);
    }
}
public class InBoundHandlerB extends ChannelInboundHandlerAdapter {
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("InBoundHandlerB: " + msg);
        ctx.fireChannelRead(msg);
    }
    public void channelActive(ChannelHandlerContext ctx) {
        ctx.channel().pipeline().fireChannelRead("hello world");  // 1
//        ctx.fireChannelRead("hello world");  // 2
    }
}
public class InBoundHandlerC extends ChannelInboundHandlerAdapter {
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        System.out.println("InBoundHandlerC: " + msg);
        ctx.fireChannelRead(msg);
    }
}

```

因此客户端channel pipeline的结构如下图所示:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20pipeline-4.jpg)

可以看到，在新连接接入时，会先回调channelActive()方法，此时InBoundHandlerB的channelActive()方法得到执行，触发客户端pipeline.fireChannelRead()方法，将channlRead事件传播至pipeline。在实际工作中一般是由NioEventLoop轮询到读IO事件，并触发NioByteUnsafe.read()操作，如下:

```java
// NioEventLoop.processSelectedKey方法局部
if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
    // 对于NioSocketChannel，是NioByteUnsafe.read()
    unsafe.read();
    // ...
}
// NioByteUnsafe
public final void read() {
    final ChannelConfig config = config(); // NioSocketChannel配置、pipeline
    final ChannelPipeline pipeline = pipeline();
    final ByteBufAllocator allocator = config.getAllocator(); // ByteBuf分配
    final RecvByteBufAllocator.Handle allocHandle = recvBufAllocHandle(); // AdaptiveRecvByteBufAllocator.HandleImpl
    allocHandle.reset(config);

    ByteBuf byteBuf = null;
    boolean close = false;
    do {
        byteBuf = allocHandle.allocate(allocator);  // doubt 分配ByteBuf
        allocHandle.lastBytesRead(doReadBytes(byteBuf)); // 将数据读取到分配的ByteBuf中去
        if (allocHandle.lastBytesRead() <= 0) {
            // nothing was read. release the buffer.
            byteBuf.release();
            byteBuf = null;
            close = allocHandle.lastBytesRead() < 0;
            break;
        }

        allocHandle.incMessagesRead(1);
        readPending = false;
        pipeline.fireChannelRead(byteBuf); // 触发事件，将会引发pipeline的读事件传播
        byteBuf = null;
    } while (allocHandle.continueReading());  // doubt

    allocHandle.readComplete(); // doubt
    pipeline.fireChannelReadComplete();
}
```

在read()方法中读取到IO数据之后，调用`pipeline.fireChannelRead(byteBuf);`代码将channelRead事件传播至pipeline，并从HeadContext开始处理该读到的数据。

这里为了分析方便，使用InBoundHandlerB的channelActive()方法模拟触发客户端channel读取到数据并传播至pipeline的逻辑，并分为两种情况分析:

-  `ctx.channel().pipeline().fireChannelRead("hello world");`: 调用pipeline的fireChannelRead()方法传播事件；
- `ctx.fireChannelRead("hello world");`: 调用ChannelHandlerContext的fireChannelRead()方法传播事件。

分析两种触发方式的区别。

> `ctx.channel().pipeline().fireChannelRead("hello world");`: 调用pipeline的fireChannelRead()方法传播事件

```java
// DefaultChannelPipeline
public final ChannelPipeline fireChannelRead(Object msg) {
    AbstractChannelHandlerContext.invokeChannelRead(head, msg);
    return this;
}
// AbstractChannelHandlerContext
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(m);
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(m);
            }
        });
    }
}
// HeadContext
private void invokeChannelRead(Object msg) {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg);
    }
}
```

DefaultChannelPipeline.fireChannelRead()方法首先调用到HeadContext.channelRead()方法:

```java
// HeadContext
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    ctx.fireChannelRead(msg);
}
```

HeadContext.channelRead()方法将事件往后传播:

```java
// HeadContext
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(), msg);
    return this;
}
private AbstractChannelHandlerContext findContextInbound() {
    AbstractChannelHandlerContext ctx = this; // 遍历链表
    do {
        ctx = ctx.next; // 往后查找inbound handler
    } while (!ctx.inbound);
    return ctx;
}

```

此时找到InBoundHandlerA，并调用invokeChannelRead()方法:

```java
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
    final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRead(m);
    } else {
        executor.execute(new Runnable() {
            @Override
            public void run() {
                next.invokeChannelRead(m);
            }
        });
    }
}
private void invokeChannelRead(Object msg) {
    if (invokeHandler()) {
        try {
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg);
    }
}
```

就这样，InBoundHandlerA的channelRead()方法就会回调到。类似的，InBoundHandlerC、InBoundHandlerB的channelRead()方法也会回调到。最终channelRead()事件到达TailContext:

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    onUnhandledInboundMessage(msg);
}
protected void onUnhandledInboundMessage(Object msg) {
    try {
        logger.debug(
                "Discarded inbound message {} that reached at the tail of the pipeline. " +
                        "Please check your pipeline configuration.", msg);
    } finally {
        ReferenceCountUtil.release(msg);
    }
}
```

对于未处理的msg，TailContext只是打印了一条debug日志并释放内存。

根据上面的分析可知，通过pipleline触发inbound事件传播时，从HeadContext开始传播。对于inbound事件，会按照ChannelInboundHandler添加的顺序处理该事件，HeadContext首先处理该事件，然后依次传递到pipeline中的ChannelInboundHandler中。

> `ctx.fireChannelRead("hello world");`: 调用ChannelHandlerContext的fireChannelRead()方法传播事件

```java
public ChannelHandlerContext fireChannelRead(final Object msg) {
    invokeChannelRead(findContextInbound(), msg);
    return this;
}
```

通过调用ChannelHandlerContext的fireChannelRead()方法传播channelRead事件时，直接查找到当前节点的下一个inbound节点，将事件传播至该节点，不会从HeadContext开始传递。

以上便是inbound事件传播的流程分析。

##### 3.SimpleChannelInboundHandler介绍

```java
public abstract class SimpleChannelInboundHandler<I> extends ChannelInboundHandlerAdapter {

    private final boolean autoRelease;

    protected SimpleChannelInboundHandler() {
        this(true);
    }
    protected SimpleChannelInboundHandler(boolean autoRelease) {
        matcher = TypeParameterMatcher.find(this, SimpleChannelInboundHandler.class, "I");
        this.autoRelease = autoRelease;
    }

    protected SimpleChannelInboundHandler(Class<? extends I> inboundMessageType) {
        this(inboundMessageType, true);
    }

    protected SimpleChannelInboundHandler(Class<? extends I> inboundMessageType, boolean autoRelease) {
        matcher = TypeParameterMatcher.get(inboundMessageType);
        this.autoRelease = autoRelease;
    }

    public boolean acceptInboundMessage(Object msg) throws Exception {
        return matcher.match(msg);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        boolean release = true;
        try {
            if (acceptInboundMessage(msg)) {
                @SuppressWarnings("unchecked")
                I imsg = (I) msg;
                channelRead0(ctx, imsg); // 处理数据
            } else {
                release = false;
                ctx.fireChannelRead(msg);
            }
        } finally {
            if (autoRelease && release) {
                ReferenceCountUtil.release(msg); // 释放内存
            }
        }
    }

    protected abstract void channelRead0(ChannelHandlerContext ctx, I msg) throws Exception;
}
```

用户一般可以继承SimpleChannelInboundHandler类来实现自己的ChannelInboundHandler，同时可以指定该handler想要处理的数据类型。这样channelRead事件传播至该handler时，可以根据指定的数据类型决定是否处理传入的数据，如果匹配，则将传入的数据转为指定的类型，并调用channelRead0()方法，并在事件处理完毕后释放内存；如果类型不匹配，直接将事件往pipeline后面传播。

### 三、outbound事件的传播

##### 1.outbound事件

首先来看ChannelOutboundHandler接口:

```java
public interface ChannelOutboundHandler extends ChannelHandler {
    void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception;
    void connect(
            ChannelHandlerContext ctx, SocketAddress remoteAddress,
            SocketAddress localAddress, ChannelPromise promise) throws Exception;
    void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception;
    void read(ChannelHandlerContext ctx) throws Exception;
    void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception;
    void flush(ChannelHandlerContext ctx) throws Exception;
}
```

从ChannelOutboundHandler接口的定义可以看出，outbound事件包括端口绑定bind、连接connect、断开连接disconnect、关闭连接close、取消channel在EventLoop的注册、读写数据、刷新数据等。这些操作一般都是由**用户主动触发**的，这与inbound事件(如channelRead)**被动触发**的情况不同。

##### 2.write事件的传播

下面以write事件的传播为例，来分析outbound事件的传播细节。先看服务端ServerBootstrap的配置:

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .childOption(ChannelOption.TCP_NODELAY, true)
        .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
                ch.pipeline().addLast(new OutBoundHandlerA());
                ch.pipeline().addLast(new OutBoundHandlerC());
                ch.pipeline().addLast(new OutBoundHandlerB());
            }
        });
```

可知服务端在接收新连接时会为客户端channel pipeline添加OutBoundHandlerA、OutBoundHandlerC、OutBoundHandlerB三个ChannelOutboundHandler。

```java
public class OutBoundHandlerA extends ChannelOutboundHandlerAdapter {
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("OutBoundHandlerA: " + msg);
        ctx.write(msg, promise);
    }
}
public class OutBoundHandlerB extends ChannelOutboundHandlerAdapter {
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("OutBoundHandlerB: " + msg);
        ctx.write(msg, promise);
    }
    public void handlerAdded(final ChannelHandlerContext ctx) {
        ctx.executor().schedule(() -> { // 定时任务。模拟用户写操作
            ctx.channel().write("hello world"); // 1
//            ctx.write("hello world");  // 2
        }, 3, TimeUnit.SECONDS);
    }
}
public class OutBoundHandlerC extends ChannelOutboundHandlerAdapter {
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        System.out.println("OutBoundHandlerC: " + msg);
        ctx.write(msg, promise);
    }
}
```

因此客户端channel pipeline的结构如下图所示:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20pipeline-5.jpg)

可以看到，在新连接接入时，会先回调OutBoundHandlerB的handlerAdded()方法，该方法会调度一个定时任务，模拟用户触发的写操作，将write事件传播至pipeline。

这里分两种情况进行分析，介绍两种触发方式的区别:

-  `ctx.channel().write("hello world");`: 调用channel(也即pipeline)的write()方法传播事件；
- `ctx.write("hello world");`: 调用ChannelHandlerContext的write()方法传播事件。

> `ctx.channel().write("hello world");`: 调用channel(也即pipeline)的write()方法传播事件

```java
// DefaultChannelPipeline
public final ChannelFuture write(Object msg) {
    return tail.write(msg);
}
// TailContext
public ChannelFuture write(Object msg) {
    return write(msg, newPromise());
}
// TailContext
public ChannelFuture write(final Object msg, final ChannelPromise promise) {
    try {
        if (!validatePromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return promise;
        }
    } catch (RuntimeException e) {
        ReferenceCountUtil.release(msg);
        throw e;
    }
    write(msg, false, promise); // 这里！！！

    return promise;
}
// TailContext
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound(); // 找到下一个outbound handler
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) { // false
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise); // 这里!!!
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}
```

执行 `ctx.channel().write("hello world");`时，会调用到TailContxt.write()方法。TailContxt.write()方法中首先找出下一个outbound handler:

```java
// TailContext
private AbstractChannelHandlerContext findContextOutbound() {
    AbstractChannelHandlerContext ctx = this;
    do {
        ctx = ctx.prev; // 反向遍历
    } while (!ctx.outbound);
    return ctx;
}
```

findContextOutbound()方法通过链表反向遍历的方式查找下一个outbound handler，这里是找到了OutBoundHandlerB，并调用OutBoundHandlerB.invokeWrite()方法。

```java
// OutBoundHandlerB对应的ChannelHandlerContext
private void invokeWrite(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
    } else {
        write(msg, promise);
    }
}
// OutBoundHandlerB对应的ChannelHandlerContext
private void invokeWrite0(Object msg, ChannelPromise promise) {
    try {
        ((ChannelOutboundHandler) handler()).write(this, msg, promise);
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
// OutBoundHandlerB
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    System.out.println("OutBoundHandlerB: " + msg);
    ctx.write(msg, promise);
}
```

在调用完OutBoundHandlerB.write()方法后，通过`ctx.write(msg, promise);`继续传播事件:

```java
// OutBoundHandlerB对应的ChannelHandlerContext
public ChannelFuture write(final Object msg, final ChannelPromise promise) {
    try {
        if (!validatePromise(promise, true)) {
            ReferenceCountUtil.release(msg);
            // cancelled
            return promise;
        }
    } catch (RuntimeException e) {
        ReferenceCountUtil.release(msg);
        throw e;
    }
    write(msg, false, promise);

    return promise;
}
// OutBoundHandlerB对应的ChannelHandlerContext
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound();
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}
```

跟上面类似的逻辑，调用`ctx.write(msg, promise);`时直接查找下一个outbound handler，这里是OutBoundHandlerC。接下来是通过`next.invokeWrite(m, promise);`调用OutBoundHandlerC.write()方法，与上面相同。就这样，write事件将传播至HeadContext。

```java
// HeadContext
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```

HeadContext将调用NioByteUnsafe.write()方法，最终处理这个写出的数据:

```java
public final void write(Object msg, ChannelPromise promise) {

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
        msg = filterOutboundMessage(msg); // mas转换，比如堆内存转为直接内存
        size = pipeline.estimatorHandle().size(msg); // 计算消息的大小
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }

    outboundBuffer.addMessage(msg, size, promise); // 消息添加到ChannelOutboundBuffer
}
```

根据上面的分析可知，通过channel(也即pipleline)触发outbound事件传播时，从TailContext开始传播。对于outbound事件，会按照ChannelOutboundHandler添加的顺序**逆序**处理该事件，TailContext由于是inbound类型的ChannelHandler，它直接将outbound事件传播至下一个outbound节点，然后逐渐传递到pipeline中的HeadContext节点，最终事件由HeadContext节点处理。

> `ctx.write("hello world");`: 调用ChannelHandlerContext的write()方法传播事件

```java
// OutBoundHandlerB对应的ChannelHandlerContext
public ChannelFuture write(Object msg) {
    return write(msg, newPromise());
}
public ChannelFuture write(final Object msg, final ChannelPromise promise) {
   
  	// ... 省略
  
    write(msg, false, promise);

    return promise;
}
private void write(Object msg, boolean flush, ChannelPromise promise) {
    // 直接查找下一个outbound handler
    AbstractChannelHandlerContext next = findContextOutbound(); 
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise); // 这里!!!
        } 
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}
```

调用ChannelHandlerContext的write()方法传播outbound事件时，直接从当前节点开始反向遍历context链表，查找下一个outbound handler，并调用其write方法，然后将write事件传播至HeadContext。

至此，outbound事件传播分析完毕。

### 四、异常事件的传播

下面分析异常事件传播的细节。先看服务端ServerBootstrap的配置:

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .childOption(ChannelOption.TCP_NODELAY, true)
        .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
                ch.pipeline().addLast(new InBoundHandlerA());
                ch.pipeline().addLast(new InBoundHandlerB());
                ch.pipeline().addLast(new InBoundHandlerC());
                ch.pipeline().addLast(new OutBoundHandlerA());
                ch.pipeline().addLast(new OutBoundHandlerB());
                ch.pipeline().addLast(new OutBoundHandlerC());
            }
        });
```

可知服务端在接收新连接时会为客户端channel pipeline添加InBoundHandlerA、InBoundHandlerB、InBoundHandlerC、OutBoundHandlerA、OutBoundHandlerB、OutBoundHandlerC六个ChannelHandler。

因此客户端channel pipeline的结构如下图所示:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20pipeline-6.jpg)

```java
public class InBoundHandlerA extends ChannelInboundHandlerAdapter {
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("InBoundHandlerA.exceptionCaught()");
        ctx.fireExceptionCaught(cause);
    }
}
public class InBoundHandlerB extends ChannelInboundHandlerAdapter {
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        throw new BusinessException("from InBoundHandlerB"); // 抛出异常
    }
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("InBoundHandlerB.exceptionCaught()");
        ctx.fireExceptionCaught(cause);
    }
}
public class InBoundHandlerC extends ChannelInboundHandlerAdapter {
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("InBoundHandlerC.exceptionCaught()");
        ctx.fireExceptionCaught(cause);
    }
}
public class OutBoundHandlerA extends ChannelOutboundHandlerAdapter {
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("OutBoundHandlerA.exceptionCaught()");
        ctx.fireExceptionCaught(cause);
    }
}
public class OutBoundHandlerB extends ChannelOutboundHandlerAdapter {
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("OutBoundHandlerB.exceptionCaught()");
        ctx.fireExceptionCaught(cause);
    }
}
public class OutBoundHandlerC extends ChannelOutboundHandlerAdapter {
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        System.out.println("OutBoundHandlerC.exceptionCaught()");
        ctx.fireExceptionCaught(cause);
    }
}
public class BusinessException extends Exception {
    public BusinessException(String message) {
        super(message);
    }
}

```

从InBoundHandlerB的定义可以看出，在接收到channelRead事件时将抛出BusinessException，这种情况模拟了inbound事件在pipeline传播以及处理过程中发生的异常。下面先来分析inbound事件传播过程中发生异常时，异常事件传播的细节。

##### 1.inbound事件传播过程中发生异常

假设channel读取到了一定数据，并回调了InBoundHandlerB.channelRead()方法，此时抛出BusinessException异常:

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
    throw new BusinessException("from InBoundHandlerB");
}
```

```java
private void invokeChannelRead(Object msg) {
    if (invokeHandler()) {
        try {
          	// handler(): InBoundHandlerB
            ((ChannelInboundHandler) handler()).channelRead(this, msg);
        } catch (Throwable t) {
            notifyHandlerException(t);
        }
    } else {
        fireChannelRead(msg);
    }
}
```

此时将进入`notifyHandlerException(t);`:

```java
private void notifyHandlerException(Throwable cause) {
    if (inExceptionCaught(cause)) {
        if (logger.isWarnEnabled()) {
            logger.warn(
                    "An exception was thrown by a user handler " +
                            "while handling an exceptionCaught event", cause);
        }
        return;
    }

    invokeExceptionCaught(cause);
}
```

notifyHandlerException()方法直接调用`invokeExceptionCaught(cause);`传播异常事件:

```java
private void invokeExceptionCaught(final Throwable cause) {
    if (invokeHandler()) {
        try {
            handler().exceptionCaught(this, cause);
        } catch (Throwable error) {
            // 省略...
        }
    } else {
        fireExceptionCaught(cause);
    }
}
```

在inbound事件传播过程中发生异常时，首先调用发生异常所在handler的exceptionCaught方法，即InBoundHandlerB.exceptionCaught():

```java
// InBoundHandlerB
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
    System.out.println("InBoundHandlerB.exceptionCaught()");
    ctx.fireExceptionCaught(cause);
}
```

然后调用`ctx.fireExceptionCaught(cause);`继续传播异常事件:

```java
// InBoundHandlerB对应的ChannelHandlerContext
public ChannelHandlerContext fireExceptionCaught(final Throwable cause) {
  	// next: InBoundHandlerC
    invokeExceptionCaught(next, cause); // 直接调用next节点的exceptionCaught方法
    return this;
}
static void invokeExceptionCaught(final AbstractChannelHandlerContext next, final Throwable cause) {
    ObjectUtil.checkNotNull(cause, "cause");
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeExceptionCaught(cause); // 这里!!!
    } else {
        try {
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    next.invokeExceptionCaught(cause);
                }
            });
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to submit an exceptionCaught() event.", t);
                logger.warn("The exceptionCaught() event that was failed to submit was:", cause);
            }
        }
    }
}
private void invokeExceptionCaught(final Throwable cause) {
    if (invokeHandler()) {
        try {
            handler().exceptionCaught(this, cause); // InBoundHandlerC
        } catch (Throwable error) {
            // 省略..
        }
    } else {
        fireExceptionCaught(cause);
    }
}
```

可以看到，在异常发生节点InBoundHandlerB继续传播事件时，是直接调用了InBoundHandlerB对应context节点的next节点InBoundHandlerC的exceptionCaught方法，而不管下一个节点是inbound还是outbound类型。就这样，异常事件按顺序经过InBoundHandlerB、InBoundHandlerC、OutBoundHandlerA、OutBoundHandlerB、OutBoundHandlerC，最终到达TailContext:

```java
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
    onUnhandledInboundException(cause);
}
protected void onUnhandledInboundException(Throwable cause) {
    try {
        logger.warn(
                "An exceptionCaught() event was fired, and it reached at the tail of the pipeline. " +
                        "It usually means the last handler in the pipeline did not handle the exception.",
                cause);
    } finally {
        ReferenceCountUtil.release(cause);
    }
}
```

如果TailContext之前的handler都未处理该异常事件，在TailContext将以warn日志的方式记录该异常信息，并释放内存。

以上是inbound事件传播过程中发生异常，异常事件的传播过程。

##### 2.outbound事件传播过程中发生异常

下面以channel.writeAndFlush事件的传播为例，分析outbound事件传播过程中发生异常时，异常事件的传播细节。

```java
// AbstractChannelHandlerContext
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound(); // 某个outbound handler
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise); // 这里！！！
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}
private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise);
        invokeFlush0(); // 异常捕获
    } else {
        writeAndFlush(msg, promise);
    }
}
```

假设writeAndFlush事件传播至context链表中的某个节点，因此将调用以上的invokeWriteAndFlush()方法。继续看 `invokeWrite0(msg, promise);`中的逻辑:

```java
private void invokeWrite0(Object msg, ChannelPromise promise) {
    try {
        ((ChannelOutboundHandler) handler()).write(this, msg, promise);
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
private static void notifyOutboundHandlerException(Throwable cause, ChannelPromise promise) {
    if (!(promise instanceof VoidChannelPromise)) {
        PromiseNotificationUtil.tryFailure(promise, cause, logger); // promise设置为失败
    }
}
```

可见，如果在ChannelOutboundHandler.write()方法中发生异常，只是调用notifyOutboundHandlerException()方法，将promise设置为失败状态，不抛出任何异常。

再来看` invokeFlush0();`的逻辑:

```java
private void invokeFlush0() {
    try {
        ((ChannelOutboundHandler) handler()).flush(this);
    } catch (Throwable t) {
        notifyHandlerException(t);
    }
}
```

可见，如果在ChannelOutboundHandler.flush()方法中发生异常，将调用notifyHandlerException()方法:

```java
private void notifyHandlerException(Throwable cause) {
    // ...省略

    invokeExceptionCaught(cause);
}
 private void invokeExceptionCaught(final Throwable cause) {
    if (invokeHandler()) {
        try {
            handler().exceptionCaught(this, cause);
        } catch (Throwable error) {
            // 省略...
        }
    } else {
        fireExceptionCaught(cause);
    }
}
```

同样的，会触发异常事件从当前节点向后传播，最后到达TailContext。

##### 3.异常事件传播的总结

如果在inbound或者outbound事件传播的过程中抛出异常，异常事件exceptionCaught会按照ChannelHandler添加的顺序，从当前节点开始传播到尾部TailContext(与inbound、outbound无关)。

##### 4.异常处理的最佳实践

根据以上异常事件传播的细节，针对上面的例子，异常处理的最佳实践如下:

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .childOption(ChannelOption.TCP_NODELAY, true)
        .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
                ch.pipeline().addLast(new InBoundHandlerA());
                ch.pipeline().addLast(new InBoundHandlerB());
                ch.pipeline().addLast(new InBoundHandlerC());
                ch.pipeline().addLast(new OutBoundHandlerA());
                ch.pipeline().addLast(new OutBoundHandlerB());
                ch.pipeline().addLast(new OutBoundHandlerC());
                ch.pipeline().addLast(new ExceptionCaughtHandler()); // 异常处理器
            }
        });

public class ExceptionCaughtHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        // ..
        if (cause instanceof BusinessException) {
            System.out.println("BusinessException");
        }
    }
}
```

由于异常事件最终会传播到pipeline尾部，因此可以在channel pipeline的尾部增加全局异常处理器，针对不同异常做不同的处理，用于处理pipeline前面的ChanelHandler未捕获的异常。

### 五、面试问题

##### 1.Netty是如何判断ChannelHandler类型的？

​	在添加ChannelHandler并创建ChannelHandlerContext的时候，通过instanceof判断handler是否是ChannelInboundHandler和ChannelOutboundHanler，并将结果保存到AbstractChannelHandlerContext的inbound和outbound两个boolean变量中。

```java
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
    return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}
final class DefaultChannelHandlerContext extends AbstractChannelHandlerContext {

    private final ChannelHandler handler;

    DefaultChannelHandlerContext(
            DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
        super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
        if (handler == null) {
            throw new NullPointerException("handler");
        }
        this.handler = handler; // 保存handler
    }

    @Override
    public ChannelHandler handler() {
        return handler;
    }

    private static boolean isInbound(ChannelHandler handler) {
        return handler instanceof ChannelInboundHandler; // 判断是否是ChannelInboundHandler
    }

    private static boolean isOutbound(ChannelHandler handler) { // 判断是否是ChannelOutboundHandler
        return handler instanceof ChannelOutboundHandler;
    }
}
```

##### 2.对于ChannelHandler的添加应该遵循什么样的顺序？

   对于inbound事件的传播，事件的处理顺序与ChannelInboundHandler的添加顺序相同；对于outbound事件的传播，事件的处理顺序与ChannelOutboundHandler的添加顺序相反。对于异常事件的传播，事件的处理顺序与ChannelHandler的添加顺序相同，与inbound、outbound无关。

##### 3.用户手动触发事件传播，不同的触发方式有什么样的区别？

​	(1) `ctx.channel.xxx()`: 对于inbound事件，从pipeline头部节点head开始传播；对于outbound事件，从pipeline尾部节点tail开始传播。

​	(2) `ctx.xxx()` : 对于inbound事件，从当前节点下一节点开始传播(指向尾部tail)；对于outbound事件，从当前节点下一节点开始传播(指向头部head)。

`xxx()`方法指fireChannelRead、write等方法。

### 参考文章

- [netty源码分析之pipeline(一)](https://www.jianshu.com/p/6efa9c5fa702)
- [netty源码分析之pipeline(二)](https://www.jianshu.com/p/087b7e9a27a2)

