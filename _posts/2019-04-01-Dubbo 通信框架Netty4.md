---
layout:     post
title:      Dubbo通信框架Netty4
subtitle:   Netty4
date:       2019-04-01
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo通信框架Netty4

Netty4是2.5.6引入的，2.5.6之前的Netty用的是Netty3。在Dubbo源码中相较于Netty3，添加Netty4主要仅仅改了两个类：NettyServer，NettyClient，还有就是编解码。

**使用方式：**

提供者端：

```xml
<dubbo:protocol server="netty4" />
<dubbo:provider server="netty4" />
```

消费者端：

```xml
<dubbo:consumer client="netty4" />
```

### 一、服务提供者端NettyServer

```java
class NettyServer extends AbstractServer implements Server {

    private static final Logger logger = LoggerFactory.getLogger(NettyServer.class);

    // 绑定到服务器的数据通道映射表
    // {@code <ip:port(消费者), channel(NettyChannel)>}
    private Map<String, Channel> channels;

    // 服务器引导程序
    private ServerBootstrap bootstrap;

    // Netty数据通道(Server Bind)
    private io.netty.channel.Channel channel;

    // Netty主线程的NIO事件轮询组，用于处理事件
    private EventLoopGroup bossGroup;
    // Netty工作者线程的NIO事件轮询组，用于处理具体的请求
    private EventLoopGroup workerGroup;

    // 创建 NettyServer 实例
    // @param url 提供者url
    // @param handler DecodeHandler实例
    NettyServer(URL url, ChannelHandler handler) throws RemotingException {
        super(url, ChannelHandlers.wrap(handler, ExecutorUtil.setThreadName(url, SERVER_THREAD_POOL_NAME)));
    }

    @Override
    protected void doOpen() throws Throwable {
        // 设置logger factory
        NettyHelper.setNettyLoggerFactory();

        bootstrap = new ServerBootstrap();

        bossGroup = new NioEventLoopGroup(1, new DefaultThreadFactory("NettyServerBoss", true));
        workerGroup = new NioEventLoopGroup(getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS),
                new DefaultThreadFactory("NettyServerWorker", true));

        // Netty服务端处理器
        final NettyServerHandler nettyServerHandler = new NettyServerHandler(getUrl(), this);
        channels = nettyServerHandler.getChannels();

        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                // 设置TCP底层相关的属性
                // 是否开启Nagle算法，true表示关闭，要求数据的高实时性
                .childOption(ChannelOption.TCP_NODELAY, Boolean.TRUE)
                // 是否允许重复使用本地地址和端口，true表示允许
                .childOption(ChannelOption.SO_REUSEADDR, Boolean.TRUE)
                // 使用对象池，重用缓冲区
                .childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                .childHandler(new ChannelInitializer<NioSocketChannel>() {
                    @Override
                    protected void initChannel(NioSocketChannel ch) throws Exception {
                        // 构造器参数: <DubboCountCodec，提供者url，ChannelHandler(NettyServer实例)>
                        NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                        ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                                .addLast("decoder", adapter.getDecoder())  // 解码器
                                .addLast("encoder", adapter.getEncoder())  // 编码器
                                .addLast("handler", nettyServerHandler);   // 服务端逻辑处理器，处理入站、出站事件
                    }
                });
        // bind [id: 0x7b324585, /0.0.0.0:20881] 监听在任意ip地址
        ChannelFuture channelFuture = bootstrap.bind(getBindAddress());
        // 同步等待bind完成
        channelFuture.syncUninterruptibly();
        channel = channelFuture.channel();

    }

    @Override
    protected void doClose() throws Throwable {
        try {
            if (channel != null) {
                // unbind.
                channel.close();
            }
        } catch (Throwable e) {
            logger.warn(e.getMessage(), e);
        }
        try {
            Collection<com.alibaba.dubbo.remoting.Channel> channels = getChannels();
            if (channels != null && channels.size() > 0) {
                for (com.alibaba.dubbo.remoting.Channel channel : channels) {
                    try {
                        channel.close();
                    } catch (Throwable e) {
                        logger.warn(e.getMessage(), e);
                    }
                }
            }
        } catch (Throwable e) {
            logger.warn(e.getMessage(), e);
        }
        try {
            if (bootstrap != null) {
                bossGroup.shutdownGracefully();
                workerGroup.shutdownGracefully();
            }
        } catch (Throwable e) {
            logger.warn(e.getMessage(), e);
        }
        try {
            if (channels != null) {
                channels.clear();
            }
        } catch (Throwable e) {
            logger.warn(e.getMessage(), e);
        }
    }

    @Override
    public Collection<Channel> getChannels() {
        Collection<Channel> chs = new HashSet<Channel>();
        for (Channel channel : this.channels.values()) {
            if (channel.isConnected()) {
                chs.add(channel);
            } else {
                // 若发现数据通道已断开连接，则移除
                channels.remove(NetUtils.toAddressString(channel.getRemoteAddress()));
            }
        }
        return chs;
    }

    @Override
    public Channel getChannel(InetSocketAddress remoteAddress) {
        return channels.get(NetUtils.toAddressString(remoteAddress));
    }

    @Override
    public boolean isBound() {
        return channel.isActive();
    }

}
```

Netty4的写法与Netty3有很大不同，下面是[Netty3](https://xuanjian1992.top/2019/03/03/Dubbo-%E6%9C%8D%E5%8A%A1%E6%9A%B4%E9%9C%B2%E4%B9%8B%E6%9C%8D%E5%8A%A1%E8%BF%9C%E7%A8%8B%E6%9A%B4%E9%9C%B2-%E5%88%9B%E5%BB%BAExporter%E4%B8%8E%E5%90%AF%E5%8A%A8Netty%E6%9C%8D%E5%8A%A1%E7%AB%AF/)

```java
// 启动Netty服务，监听客户端连接
protected void doOpen() throws Throwable {
    // 设置logger factory
    NettyHelper.setNettyLoggerFactory();
    // boss worker线程池
    ExecutorService boss = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerBoss", true));
    ExecutorService worker = Executors.newCachedThreadPool(new NamedThreadFactory("NettyServerWorker", true));
    ChannelFactory channelFactory = new NioServerSocketChannelFactory(boss, worker,
            getUrl().getPositiveParameter(Constants.IO_THREADS_KEY, Constants.DEFAULT_IO_THREADS));
    bootstrap = new ServerBootstrap(channelFactory); // 线程模型、IO模型
    // getUrl()----提供者url dubbo://....
    // ChannelHandler 处理入站、出站事件
    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    // <ip:port(消费者), channel(NettyChannel)>
    channels = nettyHandler.getChannels();
    // https://issues.jboss.org/browse/NETTY-365
    // https://issues.jboss.org/browse/NETTY-379
    // final Timer timer = new HashedWheelTimer(new NamedThreadFactory("NettyIdleTimer", true));
    // 设置pipeline创建工厂
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        @Override
        public ChannelPipeline getPipeline() {
            // 构造器参数: <DubboCountCodec，提供者url，ChannelHandler(NettyServer实例)>
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
            ChannelPipeline pipeline = Channels.pipeline();
            /*int idleTimeout = getIdleTimeout();
            if (idleTimeout > 10000) {
                pipeline.addLast("timer", new IdleStateHandler(timer, idleTimeout / 1000, 0, 0));
            }*/
            pipeline.addLast("decoder", adapter.getDecoder()); // 解码器
            pipeline.addLast("encoder", adapter.getEncoder()); // 编码器
            pipeline.addLast("handler", nettyHandler); // 服务端逻辑处理器，处理入站、出站事件
            return pipeline;
        }
    });
    // bind [id: 0x7b324585, /0.0.0.0:20881] 监听在任意ip地址
    channel = bootstrap.bind(getBindAddress());
}
```

### 二、消费者端NettyClient

```java
class NettyClient extends AbstractClient {

    private static final Logger logger = LoggerFactory.getLogger(NettyClient.class);

    // Netty客户端的NIO事件轮询事件组
    private static final NioEventLoopGroup nioEventLoopGroup = new NioEventLoopGroup(Constants.DEFAULT_IO_THREADS,
            new DefaultThreadFactory("NettyClientWorker", true));

    // 客户端引导程序
    private Bootstrap bootstrap;

    // Netty数据通道
    private volatile Channel channel;

    // 初始化
    // @param url 合并消费者参数等之后的提供者url
    // @param handler DecodeHandler实例
    NettyClient(final URL url, final ChannelHandler handler) throws RemotingException {
        // wrapChannelHandler(url, handler)--MultiMessageHandler实例
        super(url, wrapChannelHandler(url, handler));
    }

    @Override
    protected void doOpen() throws Throwable {
        // 设置logger factory
        NettyHelper.setNettyLoggerFactory();

        bootstrap = new Bootstrap();
        bootstrap.group(nioEventLoopGroup)
                // 设置TCP底层相关的属性
                // 这个选项用于可能长时间没有数据交流的连接。当设置该选项以后，如果在两小时内没有数据的通信时，
                // TCP会自动发送一个活动探测数据报文。
                .option(ChannelOption.SO_KEEPALIVE, true)
                // 是否开启Nagle算法，true表示关闭，要求数据的高实时性
                .option(ChannelOption.TCP_NODELAY, true)
                // 使用对象池，重用缓冲区
                .option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
                //.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout())
                .channel(NioSocketChannel.class);

        // 设置连接超时时间。getTimeout--RPC调用超时时间
        if (getTimeout() < 3000) {
            bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000);
        } else {
            bootstrap.option(ChannelOption.CONNECT_TIMEOUT_MILLIS, getTimeout());
        }

        // Netty客户端处理器
        final NettyClientHandler nettyClientHandler = new NettyClientHandler(getUrl(), this);
        bootstrap.handler(new ChannelInitializer() {

            @Override
            protected void initChannel(Channel ch) throws Exception {
                // 构造器参数: <DubboCountCodec，合并消费者参数之后的提供者url，ChannelHandler(NettClient实例)>
                NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);
                ch.pipeline()//.addLast("logging",new LoggingHandler(LogLevel.INFO))//for debug
                        .addLast("decoder", adapter.getDecoder()) // 解码器
                        .addLast("encoder", adapter.getEncoder()) // 编码器
                        .addLast("handler", nettyClientHandler);  // 客户端逻辑处理器，处理入站、出站事件
            }
        });
    }

    @Override
    protected void doConnect() throws Throwable {
        long start = System.currentTimeMillis(); // 连接开始时间
        // getConnectAddress()返回服务端地址
        ChannelFuture future = bootstrap.connect(getConnectAddress());  // 异步操作，返回的是一个future
        try {
            // 在指定的时间内，等待连接建立
            boolean ret = future.awaitUninterruptibly(3000, TimeUnit.MILLISECONDS);

            // 指定时间内建立连接成功
            if (ret && future.isSuccess()) {
                Channel newChannel = future.channel();
                try {
                    // 关闭旧的连接(copy reference)
                    Channel oldChannel = NettyClient.this.channel;
                    if (oldChannel != null) {
                        try {
                            if (logger.isInfoEnabled()) {
                                logger.info("Close old netty channel " + oldChannel + " on create new netty channel " + newChannel);
                            }
                            oldChannel.close();
                        } finally {
                            // 移除channel缓存
                            NettyChannel.removeChannelIfDisconnected(oldChannel);
                        }
                    }
                } finally {
                    // 客户端是否关闭
                    if (NettyClient.this.isClosed()) {
                        try {
                            if (logger.isInfoEnabled()) {
                                logger.info("Close new netty channel " + newChannel + ", because the client closed.");
                            }
                            newChannel.close();
                        } finally {
                            NettyClient.this.channel = null;
                            NettyChannel.removeChannelIfDisconnected(newChannel);
                        }
                    } else {
                        // 这里成员变量channel进行复制
                        NettyClient.this.channel = newChannel;
                    }
                }
            } else if (future.cause() != null) {
                // 建立连接失败，发生了异常
                throw new RemotingException(this, "client(url: " + getUrl() + ") failed to connect to server "
                        + getRemoteAddress() + ", error message is:" + future.cause().getMessage(), future.cause());
            } else {
                // 在建立连接过程中超时了
                throw new RemotingException(this, "client(url: " + getUrl() + ") failed to connect to server "
                        + getRemoteAddress() + " client-side timeout "
                        + getConnectTimeout() + "ms (elapsed: " + (System.currentTimeMillis() - start) + "ms) from netty client "
                        + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion());
            }
        } finally {
            if (!isConnected()) {
                //future.cancel(true);
            }
        }
    }

    @Override
    protected void doDisconnect() throws Throwable {
        try {
            // 移除channel缓存
            NettyChannel.removeChannelIfDisconnected(channel);
        } catch (Throwable t) {
            logger.warn(t.getMessage());
        }
    }

    @Override
    protected void doClose() throws Throwable {
        //can't shutdown nioEventLoopGroup
        //nioEventLoopGroup.shutdownGracefully();
    }

    @Override
    protected com.alibaba.dubbo.remoting.Channel getChannel() {
        Channel c = channel;
        // channel为null或channel连接断开，则为null
        if (c == null || !c.isActive()) {
            return null;
        }
        return NettyChannel.getOrAddChannel(c, getUrl(), this);
    }

}
```

Netty3：

```java
@Override
protected void doOpen() throws Throwable {
    // 设置logger factory
    NettyHelper.setNettyLoggerFactory();
    bootstrap = new ClientBootstrap(channelFactory);
    // config 配置Channel TCP 属性
    // @see org.jboss.netty.channel.socket.SocketChannelConfig
    bootstrap.setOption("keepAlive", true); // TCP心跳
    bootstrap.setOption("tcpNoDelay", true); // 关闭Nagle算法，实时性高
    bootstrap.setOption("connectTimeoutMillis", getTimeout()); // 连接超时
    // ChannelHandler 处理入站、出站事件
    final NettyHandler nettyHandler = new NettyHandler(getUrl(), this);
    bootstrap.setPipelineFactory(new ChannelPipelineFactory() {
        @Override
        public ChannelPipeline getPipeline() {
            // 构造器参数: <DubboCountCodec，提供者url(消费者端是合并消费者参数之后的提供者url)，ChannelHandler(NettClient实例)>
            NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyClient.this);

            ChannelPipeline pipeline = Channels.pipeline();
            pipeline.addLast("decoder", adapter.getDecoder()); // 解码器
            pipeline.addLast("encoder", adapter.getEncoder()); // 编码器
            pipeline.addLast("handler", nettyHandler); // 客户端逻辑处理器，处理入站、出站事件
            return pipeline;
        }
    });
}

@Override
protected void doConnect() throws Throwable {
    long start = System.currentTimeMillis(); // 连接开始时间
    // getConnectAddress()返回服务端地址
    ChannelFuture future = bootstrap.connect(getConnectAddress()); // 异步操作，返回的是一个future
    try {
        // 在指定的超时时间内，等待连接建立
        boolean ret = future.awaitUninterruptibly(getConnectTimeout(), TimeUnit.MILLISECONDS);
        // 指定时间内建立连接成功
        if (ret && future.isSuccess()) {
            Channel newChannel = future.getChannel();
            newChannel.setInterestOps(Channel.OP_READ_WRITE); // 设置Channel感兴趣的事件是读写事件
            try {
                // 关闭旧的连接
                Channel oldChannel = NettyClient.this.channel; // copy reference
                if (oldChannel != null) {
                    try {
                        if (logger.isInfoEnabled()) {
                            logger.info("Close old netty channel " + oldChannel + " on create new netty channel " + newChannel);
                        }
                        oldChannel.close();
                    } finally {
                        // 移除channel缓存
                        NettyChannel.removeChannelIfDisconnected(oldChannel);
                    }
                }
            } finally {
                // 客户端是否关闭
                if (NettyClient.this.isClosed()) {
                    try {
                        if (logger.isInfoEnabled()) {
                            logger.info("Close new netty channel " + newChannel + ", because the client closed.");
                        }
                        newChannel.close();
                    } finally {
                        NettyClient.this.channel = null;
                        NettyChannel.removeChannelIfDisconnected(newChannel);
                    }
                } else {
                    // 这里成员变量channel进行复制
                    NettyClient.this.channel = newChannel;
                }
            }
        } else if (future.getCause() != null) {
            // 建立连接失败，发生了异常
            throw new RemotingException(this, "client(url: " + getUrl() + ") failed to connect to server "
                    + getRemoteAddress() + ", error message is: " + future.getCause().getMessage(), future.getCause());
        } else {
            // 在建立连接过程中超时了
            throw new RemotingException(this, "client(url: " + getUrl() + ") failed to connect to server "
                    + getRemoteAddress() + " client-side timeout "
                    + getConnectTimeout() + "ms (elapsed: " + (System.currentTimeMillis() - start) + "ms) from netty client "
                    + NetUtils.getLocalHost() + " using dubbo version " + Version.getVersion());
        }
    } finally {
        if (!isConnected()) {
            // 取消连接
            future.cancel();
        }
    }
}
```

### 三、编解码(后面详细分析)

Netty4：

```java
final class NettyCodecAdapter {

    // 消息编码器
    private final ChannelHandler encoder = new InternalEncoder();

    // 消息解码器
    private final ChannelHandler decoder = new InternalDecoder();

    private final Codec2         codec;
    
    private final URL            url;

    // 数据通道处理器
    private final com.alibaba.dubbo.remoting.ChannelHandler handler;

    // 初始化
    // @param codec  DubboCountCodec实例
    // @param url 提供者url(消费者端是合并消费者参数之后的提供者url)
    // @param handler NettyServer/NettyClient
    NettyCodecAdapter(Codec2 codec, URL url, com.alibaba.dubbo.remoting.ChannelHandler handler) {
        this.codec = codec;
        this.url = url;
        this.handler = handler;
    }

    ChannelHandler getEncoder() {
        return encoder;
    }

    ChannelHandler getDecoder() {
        return decoder;
    }

    // 编码器
    private class InternalEncoder extends MessageToByteEncoder {

        @Override
        protected void encode(ChannelHandlerContext ctx, Object msg, ByteBuf out) throws Exception {
            // 包装Netty ByteBuf
            com.alibaba.dubbo.remoting.buffer.ChannelBuffer buffer = new NettyBackedChannelBuffer(out);
            Channel ch = ctx.channel();
            NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);
            try {
                // 编码
                codec.encode(channel, buffer, msg);
            } finally {
                NettyChannel.removeChannelIfDisconnected(ch);
            }
        }
    }

    // 解码器
    private class InternalDecoder extends ByteToMessageDecoder {

        @Override
        protected void decode(ChannelHandlerContext ctx, ByteBuf input, List<Object> out) throws Exception {
            // 包装Netty ByteBuf
            ChannelBuffer message = new NettyBackedChannelBuffer(input);

            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.channel(), url, handler);

            Object msg;

            int saveReaderIndex;

            try {
                // decode object.
                do {
                    saveReaderIndex = message.readerIndex();
                    msg = codec.decode(channel, message);
                    // 解码时发现需要更多的字节数据
                    if (msg == Codec2.DecodeResult.NEED_MORE_INPUT) {
                        // 则设置读指针
                        message.readerIndex(saveReaderIndex);
                        break;
                    } else {
                        //is it possible to go here ?
                        if (saveReaderIndex == message.readerIndex()) {
                            throw new IOException("Decode without read data.");
                        }
                        if (msg != null) {
                            out.add(msg);
                        }
                    }
                } while (message.readable());
            } finally {
                NettyChannel.removeChannelIfDisconnected(ctx.channel());
            }
        }
    }
}
```

Netty3：

```java
final class NettyCodecAdapter {

    private final ChannelHandler encoder = new InternalEncoder();

    private final ChannelHandler decoder = new InternalDecoder();

    private final Codec2 codec;

    private final URL url;

    private final int bufferSize;

    private final com.alibaba.dubbo.remoting.ChannelHandler handler;

    // @param codec DubboCountCodec实例
    // @param url 提供者url(消费者端是合并消费者参数之后的提供者url)
    // @param handler NettyServer/NettyClient
    public NettyCodecAdapter(Codec2 codec, URL url, com.alibaba.dubbo.remoting.ChannelHandler handler) {
        this.codec = codec;
        this.url = url;
        this.handler = handler;
        int b = url.getPositiveParameter(Constants.BUFFER_KEY, Constants.DEFAULT_BUFFER_SIZE);
        this.bufferSize = b >= Constants.MIN_BUFFER_SIZE && b <= Constants.MAX_BUFFER_SIZE ? b : Constants.DEFAULT_BUFFER_SIZE;
    }

    public ChannelHandler getEncoder() {
        return encoder;
    }

    public ChannelHandler getDecoder() {
        return decoder;
    }

    // 出站写事件 编码
    @Sharable
    private class InternalEncoder extends OneToOneEncoder {

        @Override
        protected Object encode(ChannelHandlerContext ctx, Channel ch, Object msg) throws Exception {
            com.alibaba.dubbo.remoting.buffer.ChannelBuffer buffer =
                    com.alibaba.dubbo.remoting.buffer.ChannelBuffers.dynamicBuffer(1024);
            NettyChannel channel = NettyChannel.getOrAddChannel(ch, url, handler);
            try {
                codec.encode(channel, buffer, msg);
            } finally {
                NettyChannel.removeChannelIfDisconnected(ch);
            }
            return ChannelBuffers.wrappedBuffer(buffer.toByteBuffer());
        }
    }

    // 入站读事件ChannelHandler 解码
    private class InternalDecoder extends SimpleChannelUpstreamHandler {

        private com.alibaba.dubbo.remoting.buffer.ChannelBuffer buffer =
                com.alibaba.dubbo.remoting.buffer.ChannelBuffers.EMPTY_BUFFER;

        @Override
        public void messageReceived(ChannelHandlerContext ctx, MessageEvent event) throws Exception {
            Object o = event.getMessage();
            if (!(o instanceof ChannelBuffer)) {
                ctx.sendUpstream(event);
                return;
            }

            ChannelBuffer input = (ChannelBuffer) o;
            int readable = input.readableBytes();
            if (readable <= 0) {
                return;
            }

            com.alibaba.dubbo.remoting.buffer.ChannelBuffer message;
            if (buffer.readable()) {
                if (buffer instanceof DynamicChannelBuffer) {
                    buffer.writeBytes(input.toByteBuffer());
                    message = buffer;
                } else {
                    int size = buffer.readableBytes() + input.readableBytes();
                    message = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.dynamicBuffer(
                            size > bufferSize ? size : bufferSize);
                    message.writeBytes(buffer, buffer.readableBytes());
                    message.writeBytes(input.toByteBuffer());
                }
            } else {
                message = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.wrappedBuffer(
                        input.toByteBuffer());
            }

            NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
            Object msg;
            int saveReaderIndex;

            try {
                // decode object.
                do {
                    saveReaderIndex = message.readerIndex();
                    try {
                        msg = codec.decode(channel, message);
                    } catch (IOException e) {
                        buffer = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.EMPTY_BUFFER;
                        throw e;
                    }
                    if (msg == Codec2.DecodeResult.NEED_MORE_INPUT) {
                        message.readerIndex(saveReaderIndex);
                        break;
                    } else {
                        if (saveReaderIndex == message.readerIndex()) {
                            buffer = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.EMPTY_BUFFER;
                            throw new IOException("Decode without read data.");
                        }
                        if (msg != null) {
                            Channels.fireMessageReceived(ctx, msg, event.getRemoteAddress());
                        }
                    }
                } while (message.readable());
            } finally {
                if (message.readable()) {
                    message.discardReadBytes();
                    buffer = message;
                } else {
                    buffer = com.alibaba.dubbo.remoting.buffer.ChannelBuffers.EMPTY_BUFFER;
                }
                NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
            }
        }

        @Override
        public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e) throws Exception {
            ctx.sendUpstream(e);
        }
    }
}
```

