---
layout:     post
title:      Dubbo 服务暴露之Netty3使用实例
subtitle:   Netty Server/Client
date:       2019-03-03
author:     Jay
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo 服务暴露之Netty3使用实例

由于`Dubbo`默认使用的是`netty3`进行通信的，这里简单的列出一个`netty3`通信的例子，客户端与服务端之间发送字符串消息。

- `Server`端
  - `Server`引导类
  - `ServerHandler`服务端逻辑处理器
- `Client`端
  - `Client`引导类
  - `ClientHandler`客户端逻辑处理器

### 一、Server端

##### 1.Server引导类

```java
package netty.server;

import org.jboss.netty.bootstrap.ServerBootstrap;
import org.jboss.netty.channel.ChannelFactory;
import org.jboss.netty.channel.ChannelPipeline;
import org.jboss.netty.channel.ChannelPipelineFactory;
import org.jboss.netty.channel.Channels;
import org.jboss.netty.channel.socket.nio.NioServerSocketChannelFactory;
import org.jboss.netty.handler.codec.string.StringDecoder;
import org.jboss.netty.handler.codec.string.StringEncoder;

import java.net.InetSocketAddress;
import java.util.concurrent.Executors;

/**
 * 服务端引导类
 *
 * @author xuanjian.xuwj
 */
public class Server {
    public static void main(String[] args){
        ChannelFactory channelFactory = new NioServerSocketChannelFactory(
                Executors.newCachedThreadPool(), // boss线程池
                Executors.newCachedThreadPool(), // worker线程池
                8 // worker线程数
        );

        ServerBootstrap serverBootstrap = new ServerBootstrap(channelFactory);

        // 对于每一个连接channel, server都会调用PipelineFactory为该连接创建一个ChannelPipline
        serverBootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            public ChannelPipeline getPipeline() throws Exception {
                ChannelPipeline channelPipeline = Channels.pipeline();
                channelPipeline.addLast("decoder", new StringDecoder()); // 字符串解码器
                channelPipeline.addLast("serverHandler", new ServerHandler()); // 服务端逻辑处理器
                channelPipeline.addLast("encoder", new StringEncoder()); // 字符串编码器
                return channelPipeline;
            }
        });

        serverBootstrap.bind(new InetSocketAddress("127.0.0.1", 8080)); // 绑定ip port
        System.out.println("服务端启动成功...");
    }
}
```

步骤：

- 首先创建了`NioServerSocketChannelFactory`：创建boss线程池，创建worker线程池以及worker线程数。（boss线程数默认为1个）；
- 创建`ServerBootstrap server`端启动引导类；
- 为`ServerBootstrap`设置`ChannelPipelineFactory`工厂，并为`ChannelPipelineFactory`将来创建出的`ChannelPipeline`设置编码器／解码器／逻辑处理器；
- 使用`ServerBootstrap`绑定监听ip地址和端口。

##### 2.服务端逻辑处理器ServerHandler

```java
package netty.server;

import org.jboss.netty.channel.ChannelHandlerContext;
import org.jboss.netty.channel.ChannelStateEvent;
import org.jboss.netty.channel.ExceptionEvent;
import org.jboss.netty.channel.MessageEvent;
import org.jboss.netty.channel.SimpleChannelHandler;

/**
 * 服务端Channel逻辑处理器
 *
 * @author xuanjian.xuwj
 */
public class ServerHandler extends SimpleChannelHandler {
    @Override
    public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
        System.out.println("客户端连接成功, channel: " + e.getChannel().toString());
    }

    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        String message = (String) e.getMessage();
        System.out.println("接收到了客户端的消息: " + message);

        String toMsg = "服务端已收到消息，message: [" + message + "]";
        e.getChannel().write(toMsg); // 写消息发给client端

        System.out.println("服务端发送数据: " + toMsg + "完成");
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e) throws Exception {
        e.getCause().printStackTrace();
        e.getChannel().close();
    }
}
```


说明：

- 监听与客户端连接成功事件；
- 监听接收到来自客户端的消息，之后写回给客户端消息；
- 捕捉异常事件。

### 二、Client端

##### 1.Client引导类

```java
package netty.client;

import org.jboss.netty.bootstrap.ClientBootstrap;
import org.jboss.netty.channel.ChannelFactory;
import org.jboss.netty.channel.ChannelPipeline;
import org.jboss.netty.channel.ChannelPipelineFactory;
import org.jboss.netty.channel.Channels;
import org.jboss.netty.channel.socket.nio.NioClientSocketChannelFactory;
import org.jboss.netty.handler.codec.string.StringDecoder;
import org.jboss.netty.handler.codec.string.StringEncoder;

import java.net.InetSocketAddress;
import java.util.concurrent.Executors;

/**
 * 客户端引导类
 *
 * @author xuanjian.xuwj
 */
public class Client {
    public static void main(String[] args){
        ChannelFactory channelFactory = new NioClientSocketChannelFactory(
                Executors.newCachedThreadPool(),
                Executors.newCachedThreadPool(),
                8
        );
        ClientBootstrap clientBootstrap = new ClientBootstrap(channelFactory);
        clientBootstrap.setPipelineFactory(new ChannelPipelineFactory() {
            public ChannelPipeline getPipeline() throws Exception {
                ChannelPipeline channelPipeline = Channels.pipeline();
                channelPipeline.addLast("decoder", new StringDecoder());
                channelPipeline.addLast("clientHandler", new ClientHandler());
                channelPipeline.addLast("encoder", new StringEncoder());
                return channelPipeline;
            }
        });
        clientBootstrap.connect(new InetSocketAddress("127.0.0.1", 8080));
        System.out.println("客户端启动成功...");
    }
}
```

步骤：（与Server几乎相同）

- 首先创建了`NioClientSocketChannelFactory`：创建boss线程池，创建worker线程池以及worker线程数。（boss线程数默认为1个）；
- 创建`ClientBootstrap client`端启动引导类；
- 为`ClientBootstrap`设置`ChannelPipelineFactory`工厂，并为`ChannelPipelineFactory`将来创建出的`ChannelPipeline`设置编码器／解码器／逻辑处理器；
- 使用`ClientBootstrap`连接Server端监听的地址和端口。

#####2.客户端逻辑处理器ClientHandler

```java
package netty.client;

import org.jboss.netty.channel.ChannelHandlerContext;
import org.jboss.netty.channel.ChannelStateEvent;
import org.jboss.netty.channel.ExceptionEvent;
import org.jboss.netty.channel.MessageEvent;
import org.jboss.netty.channel.SimpleChannelHandler;
import org.jboss.netty.channel.WriteCompletionEvent;

/**
 * 客户端Channel逻辑处理器
 *
 * @author xuanjian.xuwj
 */
public class ClientHandler extends SimpleChannelHandler {
    @Override
    public void channelConnected(ChannelHandlerContext ctx, ChannelStateEvent e) throws Exception {
        System.out.println("客户端连接成功...");
        e.getChannel().write("hi Server"); // 异步发送
    }

    @Override
    public void writeComplete(ChannelHandlerContext ctx, WriteCompletionEvent e) throws Exception {
        System.out.println("客户端写消息完成");
    }

    @Override
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        String message = (String) e.getMessage();
        System.out.println("客户端接收到消息: " + message);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, ExceptionEvent e) throws Exception {
        e.getCause().printStackTrace();
        e.getChannel().close();
    }
}
```

说明：

- 监听与服务端连接成功事件，连接成功后，写消息给服务端；
- 监听向服务端写消息完成的事件；
- 监听接收到来自服务端的消息；
- 捕捉异常事件。

### 三、Server端与Client端输出

##### 1.Server端

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/netty3-server%E8%BE%93%E5%87%BA.png)

##### 2.Client端

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/netty3-client%E8%BE%93%E5%87%BA.png)