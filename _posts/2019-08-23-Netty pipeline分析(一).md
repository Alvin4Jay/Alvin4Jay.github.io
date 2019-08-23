---
layout:     post
title:      Netty pipeline分析(一)
subtitle:   ChannelPipeline/ChannelHandlerContext
date:       2019-08-23
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Netty
---

# Netty pipeline分析(一)

从前面[Netty服务端启动流程分析](https://xuanjian1992.top/2019/07/28/Netty%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)、[新连接接入分析](https://xuanjian1992.top/2019/08/21/Netty%E6%96%B0%E8%BF%9E%E6%8E%A5%E6%8E%A5%E5%85%A5%E5%88%86%E6%9E%90/)等文章可以知道，在Netty服务端启动创建NioServerSocketChannel和客户端新连接接入创建NioSocketChannel时，会创建对应的ChannelPipeline。它可以看作是一条流水线，原始的原料(字节流)进来，经过加工，最后输出。本文以[新连接接入](https://xuanjian1992.top/2019/08/21/Netty%E6%96%B0%E8%BF%9E%E6%8E%A5%E6%8E%A5%E5%85%A5%E5%88%86%E6%9E%90/)为例，来分析ChannelPipeline的初始化、ChannelHandler的添加和删除等内容。

### 一、pipeline的初始化

在新连接建立时，会创建NioSocketChannel对象，同时会创建NioSocketChannel的几大组件，如DefaultChannelPipeline、NioSocketChannelUnsafe、ChannelId、NioSocketChannelConfig等。pipeline在NioSocketChannel的父类构造器中被创建:

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId(); // 唯一标识符
    unsafe = newUnsafe(); // TCP读写相关 NioSocketChannelUnsafe;
    pipeline = newChannelPipeline(); // DefaultChannelPipeline初始化
}
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
```

##### 1.pipeline初始化

下面来看DefaultChannelPipeline的构造器:

```java
protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

DefaultChannelPipeline构造器中保存了NioSocketChannel的引用，且pipeline是一个双向链表结构，链表中的节点是ChannelHandlerContext对象，默认情况下链表中存在head、tail两个节点，即HeadContext、TailContext。

##### 2.HeadContext

下面分析HeadContext、TailContext的结构，在分析之前先看下HeadContext、TailContext两者公共的父类AbstractChannelHandlerContext的类结构:

```java
abstract class AbstractChannelHandlerContext extends DefaultAttributeMap
        implements ChannelHandlerContext, ResourceLeakHint {

    volatile AbstractChannelHandlerContext next;
    volatile AbstractChannelHandlerContext prev;

    private final boolean inbound;
    private final boolean outbound;
    private final DefaultChannelPipeline pipeline;
    private final String name;
    private final boolean ordered;

    AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                                  boolean inbound, boolean outbound) {
        this.name = ObjectUtil.checkNotNull(name, "name");
        this.pipeline = pipeline;
        this.executor = executor;
        this.inbound = inbound;
        this.outbound = outbound;
        ordered = executor == null || executor instanceof OrderedEventExecutor;
    }
}    
```

AbstractChannelHandlerContext有两个字段next、prev用于指向链表中的前后节点；同时使用inbound、outbound两个字段标识该AbstractChannelHandlerContext对应的ChannelHandler是inbound还是outbound类型的handler，或者两者都是；然后保存了pipeline的引用。AbstractChannelHandlerContext有一个默认实现DefaultChannelHandlerContext:

```java
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
        return handler instanceof ChannelInboundHandler;
    }

    private static boolean isOutbound(ChannelHandler handler) {
        return handler instanceof ChannelOutboundHandler;
    }
}
```

DefaultChannelHandlerContext初始化时，在构造器中使用isInbound()、isOutbound()两个静态方法判断对应的ChannelHandler是否是inbound或者outbound类型，并将参数传给父类AbstractChannelHandlerContext构造器初始化。

介绍完AbstractChannelHandlerContext的结构，下面看HeadContext的结构:

```java
final class HeadContext extends AbstractChannelHandlerContext
        implements ChannelOutboundHandler, ChannelInboundHandler {

    private final Unsafe unsafe;

    HeadContext(DefaultChannelPipeline pipeline) {
        super(pipeline, null, HEAD_NAME, false, true);
        unsafe = pipeline.channel().unsafe();
        setAddComplete();
    }

    @Override
    public ChannelHandler handler() {
        return this;
    }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        // NOOP
    }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        // NOOP
    }

    @Override
    public void bind(
            ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
            throws Exception {
        unsafe.bind(localAddress, promise); // unsafe: NioMessageUnsafe
    }

    @Override
    public void connect(
            ChannelHandlerContext ctx,
            SocketAddress remoteAddress, SocketAddress localAddress,
            ChannelPromise promise) throws Exception {
        unsafe.connect(remoteAddress, localAddress, promise);
    }

    @Override
    public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
        unsafe.disconnect(promise);
    }

    @Override
    public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
        unsafe.close(promise);
    }

    @Override
    public void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
        unsafe.deregister(promise);
    }

    @Override
    public void read(ChannelHandlerContext ctx) {
        unsafe.beginRead();
    }

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        unsafe.write(msg, promise);
    }

    @Override
    public void flush(ChannelHandlerContext ctx) throws Exception {
        unsafe.flush();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        ctx.fireExceptionCaught(cause);
    }

    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        invokeHandlerAddedIfNeeded();
        ctx.fireChannelRegistered();
    }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelUnregistered();

        // Remove all handlers sequentially if channel is closed and unregistered.
        if (!channel.isOpen()) {
            destroy();
        }
    }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelActive();

        readIfIsAutoRead();
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelInactive();
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        ctx.fireChannelRead(msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelReadComplete();

        readIfIsAutoRead();
    }

    private void readIfIsAutoRead() {
        if (channel.config().isAutoRead()) { // true
            channel.read(); // 开始读取数据
        }
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        ctx.fireUserEventTriggered(evt);
    }

    @Override
    public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {
        ctx.fireChannelWritabilityChanged();
    }
}
```

从类继承关系可以看出，HeadContext继承自AbstractChannelHandlerContext，同时实现了ChannelOutboundHandler、ChannelInboundHandler这两个接口，拥有了这两个接口的方法。在HeadContext创建时，构造器传入了pipeline引用，在调用父类AbstractChannelHandlerContext构造器时，直接指定了inbound为false、outbound为true，因此HeadContext是一个outbound类型的ChannelHandler。然后保存了NioSokcetChannel对应的Unsafe对象，再调用setAddComplete()方法，将自己标示为ADD_COMPLETE状态:

```java
final void setAddComplete() {
    for (;;) {
        int oldState = handlerState;
        if (oldState == REMOVE_COMPLETE || HANDLER_STATE_UPDATER.compareAndSet(this, oldState, ADD_COMPLETE)) {
            return;
        }
    }
}
```

setAddComplete方法中如果handler已被删除，则直接返回；否则通过CAS将handler状态标示为ADD_COMPLETE。

HeadContext对IO事件的处理可以分为对inbound和outbound事件的处理。对于inbound事件，如channelActive、channelRead等，直接简单的将事件往后传播；对于outbound事件，如read、write、flush等，都会使用保存的unsafe对象进行实际的操作。unsafe相关的操作后续会详细分析。

##### 3.TailContext

下面看TailContext的结构:

```java
final class TailContext extends AbstractChannelHandlerContext implements ChannelInboundHandler {

    TailContext(DefaultChannelPipeline pipeline) {
        super(pipeline, null, TAIL_NAME, true, false);
        setAddComplete();
    }

    @Override
    public ChannelHandler handler() {
        return this;
    }

    @Override
    public void channelRegistered(ChannelHandlerContext ctx) throws Exception { }

    @Override
    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception { }

    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception { }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception { }

    @Override
    public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception { }

    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception { }

    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception { }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        // This may not be a configuration error and so don't log anything.
        // The event may be superfluous for the current pipeline configuration.
        ReferenceCountUtil.release(evt);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        onUnhandledInboundException(cause);
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        onUnhandledInboundMessage(msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception { }
}
```

从TailContext的构造器可以看出，该TailContext是一个inbound类型的ChannelHandler。对于inbound的IO事件，TailContext实现的大部分方法没有任何逻辑，需要注意的是exceptionCaught、channelRead两个方法。

exceptionCaught方法中调用了onUnhandledInboundException方法，用于处理pipeline中前面handler未处理的异常:

```java
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

onUnhandledInboundException方法中简单的打印了一条warn日志，同时调用ReferenceCountUtil.release方法释放内存。

channelRead方法中调用了onUnhandledInboundMessage方法，用于处理pipeline中前面handler未处理的消息:

```java
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

onUnhandledInboundMessage方法中简单的打印了一条debug日志，同时调用ReferenceCountUtil.release方法释放内存。

可以看到，TailContext做的事情就是pipeline的收尾工作，对于有些未处理的inbound事件(exceptionCaught、channelRead)进行一些提醒。

### 二、添加ChannelHandler

以下是一段常见的服务端ServerBootstrap配置的代码:

```java
ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
        .channel(NioServerSocketChannel.class)
        .childOption(ChannelOption.TCP_NODELAY, true)
        .childAttr(AttributeKey.newInstance("childAttr"), "childAttrValue")
        .childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) {
								ch.pipeline().addLast(new AuthHandler());
                ch.pipeline().addLast(new InBoundHandlerA());
                ch.pipeline().addLast(new InBoundHandlerC());
                ch.pipeline().addLast(new InBoundHandlerB());
                ch.pipeline().addLast(new OutBoundHandlerA());
                ch.pipeline().addLast(new OutBoundHandlerB());
                ch.pipeline().addLast(new OutBoundHandlerC());
            }
        });
```

新连接接入时，会调用ChannelInitializer.initChannel方法初始化客户端NioSocketChannel的pipeline。在initChannel方法中会向客户端NioSocketChannel的pipeline添加ChannelHandler。下面看详细代码:

```java
public final ChannelPipeline addLast(ChannelHandler... handlers) {
    return addLast(null, handlers);
}

public final ChannelPipeline addLast(EventExecutorGroup executor, ChannelHandler... handlers) {
    if (handlers == null) {
        throw new NullPointerException("handlers");
    }

    for (ChannelHandler h: handlers) {
        if (h == null) {
            break;
        }
        addLast(executor, null, h); // 循环添加
    }

    return this;
}
```

当调用addLast方法添加ChannelHandler时，最终会调用到如下代码:

```java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        checkMultiplicity(handler); // 检查handler是否是@Sharable，是否已添加

        newCtx = newContext(group, filterName(name, handler), handler); // 新建ChannelHandlerContext

        addLast0(newCtx); // 添加到链表最后，tail之前

        if (!registered) { // channel未注册到eventloop之前的处理逻辑
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        // channel注册到eventloop之后的处理逻辑
        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

添加ChannelHandler的过程可以分为以下三步:

- ①判断是否重复添加；
- ②创建节点并添加至链表尾部；
- ③回调handler添加完成函数。

##### 1.判断是否重复添加

```java
checkMultiplicity(handler); // 检查handler是否是@Sharable，是否已添加
```

```java
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        if (!h.isSharable() && h.added) {
            throw new ChannelPipelineException(
                    h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        h.added = true;
    }
}
```

checkMultiplicity方法判断handler是否是ChannelHandlerAdapter实例，如果不是，直接返回。对于ChannelHandlerAdapter实例，判断handler是否isSharable，即是否可以共享，如果不是共享的且已经添加，则抛出异常；对于可共享的handler或者不可共享但未添加过，则添加状态added设置为true。

判断isSharable的逻辑如下:

```java
// ChannelHandlerAdapter类
boolean added;

public boolean isSharable() {
    Class<?> clazz = getClass();
    Map<Class<?>, Boolean> cache = InternalThreadLocalMap.get().handlerSharableCache();
    Boolean sharable = cache.get(clazz);
    if (sharable == null) {
        sharable = clazz.isAnnotationPresent(Sharable.class);
        cache.put(clazz, sharable);
    }
    return sharable;
}
```

isSharable方法中使用了线程相关的InternalThreadLocalMap来保证不同线程之间缓存的隔离性，而缓存的内容是不同handler与它的共享状态。如果缓存中存在对应handler的共享状态，直接返回；否则判断handler类上是否标注@Shareable注解，如果标注了该注解，则表明该handler是共享的handler，否则不是。

##### 2.创建节点并添加至链表尾部

ChannelHandlerContext节点创建的过程如下:

```java
newCtx = newContext(group, filterName(name, handler), handler);
```

首先调用`filterName(name, handler)`获取handler的名称:

```java
private String filterName(String name, ChannelHandler handler) {
    if (name == null) {
        return generateName(handler);
    }
    checkDuplicateName(name);
    return name;
}
```

如果handler添加时未传入name，则调用generateName方法创建一个name并返回:

```java
private String generateName(ChannelHandler handler) {
    Map<Class<?>, String> cache = nameCaches.get();
    Class<?> handlerType = handler.getClass();
    String name = cache.get(handlerType); // 先查看缓存中是否已缓存默认name
    if (name == null) {
        name = generateName0(handlerType); // 生成默认name
        cache.put(handlerType, name); // 缓存默认name
    }

    if (context0(name) != null) { // 查看默认name是否冲突，如果冲突，name后面的数据递增
        String baseName = name.substring(0, name.length() - 1); // Strip the trailing '0'.
        for (int i = 1;; i ++) {
            String newName = baseName + i;
            if (context0(newName) == null) {
                name = newName;
                break;
            }
        }
    }
    return name;
}
private static String generateName0(Class<?> handlerType) {
    return StringUtil.simpleClassName(handlerType) + "#0";
}
```

generateName方法中首先根据handler的类型从缓存中获取默认的name，若默认name不存在则生成默认name并缓存，如DemoHandler#0。得到默认name之后，调用context0方法检查默认name对应的节点是否已存在:

```java
private AbstractChannelHandlerContext context0(String name) {
    AbstractChannelHandlerContext context = head.next;
    while (context != tail) { // 根据name，遍历获取对应的AbstractChannelHandlerContext
        if (context.name().equals(name)) {
            return context;
        }
        context = context.next;
    }
    return null;
}
```

context0方法使用遍历的方式获取AbstractChannelHandlerContext节点，如果节点已存在，则在generateName方法中将生成的默认name后面的序号递增，直到名称对应的节点不存在为止。

在filterName方法获取name时如果传入了name，则直接调用checkDuplicateName方法检查name是否重复:

```java
private void checkDuplicateName(String name) {
    if (context0(name) != null) {
        throw new IllegalArgumentException("Duplicate handler name: " + name);
    }
}
```

checkDuplicateName方法中，如果name对应的AbstractChannelHandlerContext节点已存在，直接抛出异常。

在获取到name之后，调用newContext方法创建AbstractChannelHandlerContext节点:

```java
newCtx = newContext(group, filterName(name, handler), handler);
```

```java
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
    return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}
private EventExecutor childExecutor(EventExecutorGroup group) {
    if (group == null) {
        return null;
    }
    // 省略...
}
```

由于group为null，所以childExecutor(group)为null。上面已经介绍过DefaultChannelHandlerContext的构造器，在构造器中会通过isInbound()、isOutbound()两个方法判断handler是否ChannelInboundHandler或ChannelOutboundHandler类型的handler。

```java
DefaultChannelHandlerContext(
        DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
    super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler; // 保存handler
}
```

在创建完DefaultChannelHandlerContext之后，调用addLast0方法，将ChannelHandlerContext节点添加至链表尾部:

```java
addLast0(newCtx); // 添加到链表最后，tail之前

private void addLast0(AbstractChannelHandlerContext newCtx) {
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```

addLast0方法只是简单的链表指针操作:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20pipeline-1.jpg)

至此，pipeline添加节点的操作就完成了。

##### 3.回调添加完成函数

```java
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        // ...省略

        if (!registered) { // channel未注册到eventloop之前的处理逻辑
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        // channel注册到eventloop之后的处理逻辑
        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    callHandlerAdded0(newCtx);
    return this;
}
```

由于此时Channel已注册到EventLoop，则registered为true。同时`executor.inEventLoop()`为true，则直接调用` callHandlerAdded0(newCtx)`方法。

```java
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    try {
        ctx.handler().handlerAdded(ctx); // 回调handlerAdded()方法
        ctx.setAddComplete(); // 设置handlerAdded()方法调用完成状态
    } catch (Throwable t) {
        // 省略...
    }
}
```

callHandlerAdded0方法中直接调用ctx对应handler的handlerAdded方法，如:

```java
public class DemoHandler extends SimpleChannelInboundHandler<...> {
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
        // 节点被添加完毕之后回调到此
        // do something
    }
}
```

接下来，设置该节点的状态为ADD_COMPLETE。

```java
final void setAddComplete() {
    for (;;) {
        int oldState = handlerState;
        if (oldState == REMOVE_COMPLETE || HANDLER_STATE_UPDATER.compareAndSet(this, oldState, ADD_COMPLETE)) {
            return;
        }
    }
}
```

### 三、删除ChannelHandler

Netty有个最大的特性之一就是Handler可插拔，做到动态编织pipeline，比如在首次建立连接的时候，需要进行权限认证，在认证通过之后，就可以将此context(handler)移除，下次pipeline在传播事件的时候就就不会调用到权限认证处理器。

下面是权限认证Handler最简单的实现，第一个数据包传来的是认证信息，如果校验通过，就删除此Handler，否则直接关闭连接:

```java
public class AuthHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf data) throws Exception {
        if (verify(authDataPacket)) {
            ctx.pipeline().remove(this); // 验证通过，移除AuthHandler
        } else {
            ctx.close(); // 验证失败，关闭连接
        }
    }

    private boolean verify(ByteBuf byteBuf) {
        //...
    }
}
```

重点在 `ctx.pipeline().remove(this)` 这段代码:

```java
public final ChannelPipeline remove(ChannelHandler handler) {
    remove(getContextOrDie(handler));
    return this;
}
```

remove操作可以分为三个步骤：

- 找到待删除的节点
- 调整双向链表指针删除节点
- 回调handler删除函数

##### 1.找到待删除的节点

```java
private AbstractChannelHandlerContext getContextOrDie(ChannelHandler handler) {
    // 根据handler获取对应的AbstractChannelHandlerContext
    AbstractChannelHandlerContext ctx = (AbstractChannelHandlerContext) context(handler);
    if (ctx == null) {
        throw new NoSuchElementException(handler.getClass().getName());
    } else {
        return ctx;
    }
}
public final ChannelHandlerContext context(ChannelHandler handler) {
    if (handler == null) {
        throw new NullPointerException("handler");
    }

    AbstractChannelHandlerContext ctx = head.next;
    for (;;) { // 遍历pipeline，根据handler获取对应的AbstractChannelHandlerContext

        if (ctx == null) {
            return null;
        }

        if (ctx.handler() == handler) {
            return ctx;
        }

        ctx = ctx.next;
    }
}
```

getContextOrDie方法中通过调用context(ChannelHandler)方法查找handler对应的AbstractChannelHandlerContext节点。context(ChannelHandler)方法查找AbstractChannelHandlerContext过程中，通过遍历的方式查找节点，判断节点的handler是否是传入的handler，找到返回节点；找不到返回null。

getContextOrDie方法中如果找到了handler对应的AbstractChannelHandlerContext节点就返回，否则直接抛出异常。

##### 2.调整双向链表指针删除节点

```java
private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
    assert ctx != head && ctx != tail; // head、tail不能被删除

    synchronized (this) { // pipeline链表删除节点时需要同步
        remove0(ctx); // 从链表删除节点

        if (!registered) {  // channel未注册到eventloop之前的处理逻辑
            callHandlerCallbackLater(ctx, false);
            return ctx;
        }
        // channel注册到eventloop之后的处理逻辑
        EventExecutor executor = ctx.executor();
        if (!executor.inEventLoop()) {
            executor.execute(new Runnable() { // 用户线程
                @Override
                public void run() {
                    callHandlerRemoved0(ctx);
                }
            });
            return ctx;
        }
    }
    callHandlerRemoved0(ctx); // 调用handlerRemoved()方法(Reactor线程)
    return ctx;
}
```

删除节点之前首先断言节点不能是head、tail节点，然后使用synchronized关键字对删除节点逻辑做同步，防止多线程并发修改链表结构。接着调用`remove0(ctx);`删除节点:

```java
private static void remove0(AbstractChannelHandlerContext ctx) {
    AbstractChannelHandlerContext prev = ctx.prev;
    AbstractChannelHandlerContext next = ctx.next;
    prev.next = next;
    next.prev = prev;
}
```

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20pipeline-2.jpg)

从上面这幅图，可以很清晰地了解权限验证Handler的工作原理。在节点刚删除时，传播到它的事件还能往后传播，因为被删除节点的next指针依然指向下一个有效的节点；然后，在后续的事件传播中，事件都不会经过该被删除的节点。另外，被删除的节点因为没有对象引用到，过段时间就会被GC自动回收。

##### 3.回调handler删除函数

```java
private AbstractChannelHandlerContext remove(final AbstractChannelHandlerContext ctx) {
    assert ctx != head && ctx != tail; // head、tail不能被删除

    synchronized (this) { // pipeline链表删除节点时需要同步
        remove0(ctx); // 从链表删除节点

        if (!registered) {  // channel未注册到eventloop之前的处理逻辑
            callHandlerCallbackLater(ctx, false);
            return ctx;
        }
        // channel注册到eventloop之后的处理逻辑
        EventExecutor executor = ctx.executor();
        if (!executor.inEventLoop()) {
            executor.execute(new Runnable() { // 用户线程
                @Override
                public void run() {
                    callHandlerRemoved0(ctx);
                }
            });
            return ctx;
        }
    }
    callHandlerRemoved0(ctx); // 调用handlerRemoved()方法(Reactor线程)
    return ctx;
}
```

此时Channel已注册到Eventloop，因此registered为true，且`executor.inEventLoop()`为true，则直接调用`callHandlerRemoved0(ctx)`。

```java
private void callHandlerRemoved0(final AbstractChannelHandlerContext ctx) {
    // Notify the complete removal.
    try {
        try {
            ctx.handler().handlerRemoved(ctx);
        } finally {
            ctx.setRemoved();
        }
    } catch (Throwable t) {
        fireExceptionCaught(new ChannelPipelineException(
                ctx.handler().getClass().getName() + ".handlerRemoved() has thrown an exception.", t));
    }
}
```

callHandlerRemoved0方法中回调了hander的handlerRemoved方法，同时调用` ctx.setRemoved()`将ctx状态置为REMOVE_COMPLETE。

```java
public class DemoHandler extends SimpleChannelInboundHandler<...> {
    @Override
    public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
        // 节点被删除完毕之后回调到此，可做一些资源清理
        // do something
    }
}
```

### 四、总结

1.本文以[新连接建立](https://xuanjian1992.top/2019/08/21/Netty%E6%96%B0%E8%BF%9E%E6%8E%A5%E6%8E%A5%E5%85%A5%E5%88%86%E6%9E%90/)为例，新连接创建的过程中创建NioSocketChannel，而在创建NioSocketChannel的过程中创建了该NioSocketChannel对应的pipeline，创建完pipeline之后，自动给该pipeline添加了两个节点，即HeadContext、TailContext。pipeline中的节点为ChannelHandlerContext实例。

2.pipeline是一个双向链表结构，添加和删除节点均只需要调整链表结构。

3.pipeline中的每个节点包含具体的处理器`ChannelHandler`，节点根据`ChannelHandler`的类型是`ChannelInboundHandler`还是`ChannelOutboundHandler`来判断该节点属于inbound还是outbound或者两者都是。

### 参考文章

- [netty源码分析之pipeline(一)](https://www.jianshu.com/p/6efa9c5fa702)
- [netty源码分析之pipeline(二)](https://www.jianshu.com/p/087b7e9a27a2)

