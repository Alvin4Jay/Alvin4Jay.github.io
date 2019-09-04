---
layout:     post
title:      Netty writeAndFlush分析
subtitle:   MessageToByteEncoder/HeadContext
date:       2019-09-04
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Netty
---

# Netty writeAndFlush分析

前面[Netty pipeline分析(一)](https://xuanjian1992.top/2019/08/23/Netty-pipeline%E5%88%86%E6%9E%90(%E4%B8%80)/)、[Netty pipeline分析(二)](https://xuanjian1992.top/2019/08/24/Netty-pipeline%E5%88%86%E6%9E%90(%E4%BA%8C)/)两篇文章介绍了Channel pipeline的事件传播机制，本篇文章主要介绍Netty写事件writeAndFlush()的传播细节。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20Write-1.jpg)

如上图所示是channel的pipeline结构。其中biz channelhandler里调用`ctx.channel.writeAndFlush()`写出数据。该writeAndFlush事件将从Tail节点开始，传播至Head节点，并经过encoder handler的编码。

### 一、writeAndFlush()抽象步骤

##### 1.从Tail节点往前传播

```java
// AbstractChannel
public ChannelFuture writeAndFlush(Object msg) {
    return pipeline.writeAndFlush(msg);
}
// DefaultChannelPipeline
public final ChannelFuture writeAndFlush(Object msg) {
    return tail.writeAndFlush(msg);
}
// AbstractChannelHandlerContext(Tail)
public ChannelFuture writeAndFlush(Object msg) {
    return writeAndFlush(msg, newPromise());
}
public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    write(msg, true, promise);
    return promise;
}
private void write(Object msg, boolean flush, ChannelPromise promise) {
    AbstractChannelHandlerContext next = findContextOutbound(); // 找到outbound handler
    final Object m = pipeline.touch(msg, next);
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise); // 写并刷新
        } else {
            next.invokeWrite(m, promise); // 写
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise); // 写并刷新任务
        }  else {
            task = WriteTask.newInstance(next, m, promise); // 写任务
        }
        safeExecute(executor, task, promise, m);
    }
}

```

可以看到，`ctx.channel.writeAndFlush()`将从pipeline Tail节点开始传播，并最终调用到`next.invokeWriteAndFlush(m, promise);`这段代码，next表示下一个outbound handler。

##### 2.逐个调用ChannelOutboundHandler的write方法

```java
private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    if (invokeHandler()) {
        invokeWrite0(msg, promise); // 写
        invokeFlush0(); // 刷新，异常捕获
    } else {
        writeAndFlush(msg, promise); // 向前传播事件
    }
}
```

在调用ChannelOutboundHandler.invokeWriteAndFlush()方法时，分为两部分：`invokeWrite0(msg, promise);`和`invokeFlush0();`。

首先看`invokeWrite0(msg, promise);`:

```java
private void invokeWrite0(Object msg, ChannelPromise promise) {
    try {
        ((ChannelOutboundHandler) handler()).write(this, msg, promise); // 此处可以将Java对象编码为ByteBuf
    } catch (Throwable t) {
        notifyOutboundHandlerException(t, promise);
    }
}
```

在调用invokeWrite0过程中，会将write事件向pipeline传播，逐个调用ChannelOutboundHandler的write方法，最终传播至HeadContext。

```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
```

##### 3.逐个调用ChannelOutboundHandler的flush方法

上述invokeWriteAndFlush()方法中，调用完`invokeWrite0(msg, promise);`(即逐个调用ChannelOutboundHandler的write方法)之后，将调用`invokeFlush0()`:

```java
private void invokeFlush0() {
    try {
        ((ChannelOutboundHandler) handler()).flush(this);
    } catch (Throwable t) {
        notifyHandlerException(t);
    }
}
```

在调用invokeFlush0()过程中，会将flush事件向pipeline传播，逐个调用ChannelOutboundHandler的flush()方法，最终传播至HeadContext。

```java
public void flush(ChannelHandlerContext ctx) throws Exception {
    unsafe.flush();
}
```

至此，writeAndFlush()事件传播完毕。

### 二、MessageToByteEncoder对Java对象的编码

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20Write-1.jpg)

从这幅图可以直到，用户写出的数据(一般是Java对象)会经过一系列ChannelOutboundHandler的处理，其中就包括encoder(MessageToByteEncoder)。该编码器主要用于对用户写出的Java对象，编码为ByteBuf。

```java
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    ByteBuf buf = null;
    try {
        if (acceptOutboundMessage(msg)) { // 对象类型匹配检查
            @SuppressWarnings("unchecked")
            I cast = (I) msg;
            buf = allocateBuffer(ctx, cast, preferDirect); // 内存分配，preferDirect: true
            try {
                encode(ctx, cast, buf); // 编码实现
            } finally {
                ReferenceCountUtil.release(cast); // cast有时是ByteBuf，需要释放对象
            }

            if (buf.isReadable()) {
                ctx.write(buf, promise); // 传播数据
            } else {
                buf.release(); // 释放内存
                ctx.write(Unpooled.EMPTY_BUFFER, promise); // 传播空ByteBuf，保证事件传播
            }
            buf = null;
        } else {
            ctx.write(msg, promise); // 无法处理，数据往前传播
        }
    } catch (EncoderException e) { // 出现异常
        throw e;
    } catch (Throwable e) {
        throw new EncoderException(e);
    } finally {
        if (buf != null) { // 出现异常，释放buf内存
            buf.release();
        }
    }
}
```

MessageToByteEncoder主要是在write事件传播过程中完成Java对象的编码工作。该编码器首先检查对象类型是否是编码器能处理的，如果不是，直接往前传播write事件，不作任何处理。类型匹配的情况下，完成对象类型的转换，并分配ByteBuf：

```java
buf = allocateBuffer(ctx, cast, preferDirect);

protected ByteBuf allocateBuffer(ChannelHandlerContext ctx, @SuppressWarnings("unused") I msg,
                           boolean preferDirect) throws Exception {
    if (preferDirect) {
        return ctx.alloc().ioBuffer(); // 堆外内存
    } else {
        return ctx.alloc().heapBuffer(); // 堆内存
    }
}
```

allocateBuffer()方法得到的一般是堆外内存。

然后调用子类实现的`encode`方法，完成实际的编码工作并释放msg内存。

```java
try {
    encode(ctx, cast, buf); // 编码实现
} finally {
    ReferenceCountUtil.release(cast); // cast有时是ByteBuf，需要释放对象
}
protected abstract void encode(ChannelHandlerContext ctx, I msg, ByteBuf out) throws Exception;
```

在编码完成之后，判断ByteBuf是否可读，可读即表示编码完成，写出数据；否则，传播空ByteBuf，以保证事件得到传播。

```java
if (buf.isReadable()) {
    ctx.write(buf, promise); // 传播数据
} else {
    buf.release(); // 释放内存
    ctx.write(Unpooled.EMPTY_BUFFER, promise); // 传播空ByteBuf，保证事件传播
}
```

如果在编码过程中出现异常，最终也会释放分配的ByteBuf的内存：

```java
if (buf != null) { // 出现异常，释放buf内存
    buf.release();
}
```

经过以上几步的处理之后，Java对象就编码为了ByteBuf。

### 三、write事件传播至HeadContext

```java
// HeadContext
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}
// AbstractUnsafe
public final void write(Object msg, ChannelPromise promise) {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        safeSetFailure(promise, WRITE_CLOSED_CHANNEL_EXCEPTION);
        ReferenceCountUtil.release(msg);
        return;
    }

    int size;
    try {
        msg = filterOutboundMessage(msg); // msg转换，比如堆内存转为堆外内存
        size = pipeline.estimatorHandle().size(msg); // 计算消息的大小
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }

    outboundBuffer.addMessage(msg, size, promise); // 数据添加到ChannelOutboundBuffer
}
```

当write事件传播至HeadContext时，会将前面编码的ByteBuf缓存到ChannelOutboundBuffer中，实际上并未写出到底层JDK channel。

##### 1.堆外内存化ByteBuf

调用AbstractUnsafe.write()方法时，首先会调用`filterOutboundMessage(msg)`，将堆内存转为堆外内存。

```java
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (buf.isDirect()) {
            return msg; // 已经是直接内存，返回
        }

        return newDirectBuffer(buf); // 堆内存转为直接内存
    }

    if (msg instanceof FileRegion) {
        return msg;
    }

    throw new UnsupportedOperationException(
            "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```

假如传播的msg(ByteBuf)为堆内存，则调用`newDirectBuffer(buf)`转为堆外内存。

```java
protected final ByteBuf newDirectBuffer(ByteBuf buf) {
    final int readableBytes = buf.readableBytes();
    if (readableBytes == 0) {
        ReferenceCountUtil.safeRelease(buf);
        return Unpooled.EMPTY_BUFFER; // buf可读字节数为空，返回EMPTY_BUFFER
    }

    final ByteBufAllocator alloc = alloc();
    if (alloc.isDirectBufferPooled()) { // 池化内存
        ByteBuf directBuf = alloc.directBuffer(readableBytes);
        directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
        ReferenceCountUtil.safeRelease(buf);
        return directBuf;
    }

    final ByteBuf directBuf = ByteBufUtil.threadLocalDirectBuffer();
    if (directBuf != null) {
        directBuf.writeBytes(buf, buf.readerIndex(), readableBytes);
        ReferenceCountUtil.safeRelease(buf);
        return directBuf;
    }

    // Allocating and deallocating an unpooled direct buffer is very expensive; give up.
    return buf;
}
```

##### 2.ByteBuf添加至ChannelOutboundBuffer单链表缓存结构

```java
outboundBuffer.addMessage(msg, size, promise); // 数据添加到ChannelOutboundBuffer
```

```java
public void addMessage(Object msg, int size, ChannelPromise promise) {
    // 创建一个待写出的消息节点
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        flushedEntry = null;
        tailEntry = entry;
    } else {
        Entry tail = tailEntry; // 新Entry插入到链表尾部
        tail.next = entry;
        tailEntry = entry;
    }
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    // increment pending bytes after adding message to the unflushed arrays.
    incrementPendingOutboundBytes(size, false);
}
```

在调用ChannelOutboundBuffer.addMessage()方法添加消息时，首先会获取一个Entry实例并初始化:

```java
static Entry newInstance(Object msg, int size, long total, ChannelPromise promise) {
    Entry entry = RECYCLER.get(); // 从对象池获取Entry
    entry.msg = msg; // 初始化
    entry.pendingSize = size;
    entry.total = total;
    entry.promise = promise;
    return entry;
}
```

Entry对象的获取利用了Netty 对象池技术，后面另开文章分析。

在得到Entry对象之后，然后会将该Entry添加至ChannelOutboundBuffer内部的单链表缓存结构中。链表结构如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20Write-2.jpg)

该链表由3个指针表示其状态：

- flushedEntry：下面将要介绍的HeadContext的flush操作由两步组成：标记Entry为flushed和实际写出数据。因此，这里的flushedEntry表示链表中已标记为flushed的Entry中的第一个，这些Entry将在flush操作的第二步中被写出。
- unflushedEntry：表示未被标记为flushed的Entry中的第一个，这种Entry需要再次调用flush()操作，标记为flushed之后，才能写出。
- tailEntry：表示链表的最后一个Entry。

如果是第一个调用write()方法，此时链表为空，则第一次调用write后，链表结构如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20Write-3.jpg)

第二次调用write后，链表结构如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20Write-4.jpg)

第n次调用write后，链表结构如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20Write-5.jpg)

##### 3.设置可写状态

```java
// increment pending bytes after adding message to the unflushed arrays.
incrementPendingOutboundBytes(size, false);
```

在将ByteBuf添加至ChannelOutboundBuffer单链表缓存结构，会设置channel的可写状态。

```java
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
    if (size == 0) {
        return;
    }

    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size); // 累加size，返回最新pending值
    if (newWriteBufferSize > channel.config().getWriteBufferHighWaterMark()) { // high: 64kB
        setUnwritable(invokeLater); // 设置为不可写状态
    }
}
```

首先更新totalPendingSize变量， 统计总的ChannelOutboundBuffer待写字节数newWriteBufferSize。如果newWriteBufferSize超出WriteBufferHighWaterMark(默认64KB)，则将channel设置为不可写状态：

```java
private void setUnwritable(boolean invokeLater) {
    for (;;) {
        final int oldValue = unwritable; // unwritable: 0，可写；1，不可写
        final int newValue = oldValue | 1; // 设置为1，不可写
        if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
            if (oldValue == 0 && newValue != 0) {
                fireChannelWritabilityChanged(invokeLater); // 传播写状态变化：可写->不可写
            }
            break;
        }
    }
}
```

setUnwritable()方法通过自旋+CAS的方式更新unwritable变量为1，即不可写。如果unwritable由0变为1，即可写->不可写，则触发channelWritabilityChanged事件传播：

```java
private void fireChannelWritabilityChanged(boolean invokeLater) {
    final ChannelPipeline pipeline = channel.pipeline();
    if (invokeLater) {
        Runnable task = fireChannelWritabilityChangedTask;
        if (task == null) {
            fireChannelWritabilityChangedTask = task = new Runnable() {
                @Override
                public void run() {
                    pipeline.fireChannelWritabilityChanged();
                }
            };
        }
        // 若invokeLater=true，则将传播写状态变化的任务放入任务队列执行
        channel.eventLoop().execute(task); 
    } else {
        pipeline.fireChannelWritabilityChanged(); // 立即传播事件
    }
}
```

### 四、flush事件传播至HeadContext

```java
// HeadContext
public void flush(ChannelHandlerContext ctx) throws Exception {
    unsafe.flush();
}
// AbstractUnsafe
public final void flush() {
    assertEventLoop();

    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null) {
        return;
    }

    outboundBuffer.addFlush(); // 添加刷新标志
    flush0();
}
```

当flush事件传播至HeadContext时，HeadContext会调用AbstractUnsafe.flush()方法。该方法中首先将ChannelOutboundBuffer链表中的Entry标记为flushed，即设置刷新标志，然后再将flushed Entry的数据写出到底层JDK SocketChannel。

##### 1.添加刷新标志

```java
outboundBuffer.addFlush(); // 添加刷新标志
```

```java
public void addFlush() {
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            flushedEntry = entry;
        }
        do {
            flushed ++; // 标记为flushed的，未写出的Entry(添加刷新标志)
            // 标记为不可取消，如果Entry对应的promise已被取消，则释放msg内存，设置写状态
            if (!entry.promise.setUncancellable()) { 
                int pending = entry.cancel(); // msg大小
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);

        // All flushed so reset unflushedEntry
        unflushedEntry = null;
    }
}
```

首先调用ChannelOutboundBuffer.addFlush()方法设置Entry为flushed，调用之后链表如下图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Netty%20Write-6.jpg)

##### 2.写出数据

```java
// AbstractUnsafe
public final void flush() {
    outboundBuffer.addFlush(); // 添加刷新标志
    flush0();
}
```

在调用`outboundBuffer.addFlush();`之后，调用flush0()写出数据，该方法由子类AbstractNioUnsafe重写：

```java
protected final void flush0() {
    // Flush immediately only when there's no pending flush.
    // If there's a pending flush operation, event loop will call forceFlush() later,
    // and thus there's no need to call it now.
    // 判断现在channel是否不可写，如果不可写，直接返回。NioEventLoop在后面会调用forceFlush()写出数据
    if (isFlushPending()) {
        return;
    }
    super.flush0(); // 否则写出数据
}
```

flush0()方法首先判断channel现在是否可写：

```java
private boolean isFlushPending() {
    SelectionKey selectionKey = selectionKey();
    // interestOps包含OP_WRITE，表示channel现在不可写
    return selectionKey.isValid() && (selectionKey.interestOps() & SelectionKey.OP_WRITE) != 0;
}
```

Netty在channel不可写的时候，会使SelectionKey.interestOps包含SelectionKey.OP_WRITE标志。因此，如果isFlushPending()返回true，就表示channel不可写，此时直接返回。NioEventLoop在后面会调用forceFlush()写出数据，即

```java
// NioEventLoop.processSelectedKey(SelectionKey k, AbstractNioChannel ch)部分代码
// Process OP_WRITE first as we may be able to write some queued buffers and so free memory. 释放内存
if ((readyOps & SelectionKey.OP_WRITE) != 0) { // 处理写事件(写pending)
    // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
    ch.unsafe().forceFlush(); // pending的数据写完之后，会清除readyOps的OP_WRITE标志
}
```

在调用forceFlush()方法并写出数据之后，会清除SelectionKey.OP_WRITE标志。

如果现在channel可写，则调用父类AbstractUnsafe.flush0()方法：

```java
protected void flush0() {
    if (inFlush0) { // 已经在flush，直接返回
        // Avoid re-entrance
        return;
    }

    final ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
    if (outboundBuffer == null || outboundBuffer.isEmpty()) { // 无消息可写，直接返回
        return;
    }

    inFlush0 = true;
    try {
        doWrite(outboundBuffer); // 写出数据
    } catch (Throwable t) {
        if (t instanceof IOException && config().isAutoClose()) {
            close(voidPromise(), t, FLUSH0_CLOSED_CHANNEL_EXCEPTION, false);
        } else {
            outboundBuffer.failFlushed(t, true);
        }
    } finally {
        inFlush0 = false;
    }
}
```

AbstractUnsafe.flush0()方法将调用doWrite(ChannelOutboundBuffer)方法，该方法由NioSokcetChannel实现：

```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    for (;;) {
        int size = in.size(); // 当前待写出的Entry个数
        if (size == 0) {
            // All written so clear OP_WRITE
            clearOpWrite(); // 所有数据都已写，所以清除OP_WRITE标志
            break;
        }
        long writtenBytes = 0; // 写出的字节数
        boolean done = false; // 是否已写完
        boolean setOpWrite = false; // 是否设置OP_WRITE标志

        // Ensure the pending writes are made of ByteBufs only.
        ByteBuffer[] nioBuffers = in.nioBuffers(); // 获取标记为flushed的Entrys(ByteBuf)对应的JDK ByteBuffer[]数组
        int nioBufferCnt = in.nioBufferCount(); // 获取标记为flushed的Entrys(ByteBuf)对应的JDK ByteBuffer[]数组中，ByteBuffer元素的个数
        long expectedWrittenBytes = in.nioBufferSize(); // 预期的写字节数
        SocketChannel ch = javaChannel(); // JDK SocketChannel

        // Always us nioBuffers() to workaround data-corruption.
        // See https://github.com/netty/netty/issues/2761
        switch (nioBufferCnt) {
            case 0:
                // 这里表明Entry的msg不是ByteBuf，可能是FileRegion，则调用父类的doWrite(ChannelOutboundBuffer)方法写出数据
                // We have something else beside ByteBuffers to write so fallback to normal writes.
                super.doWrite(in);
                return;
            case 1:
                // Only one ByteBuf so use non-gathering write 只有一个ByteBuffer，使用非gathering写
                ByteBuffer nioBuffer = nioBuffers[0];
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) { // writeSpinCount: 16
                    final int localWrittenBytes = ch.write(nioBuffer); // 已写字节数
                    if (localWrittenBytes == 0) {
                        setOpWrite = true; // 写不出去，设置OP_WRITE标志
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes; // 更新剩余需写出的字节数expectedWrittenBytes
                    writtenBytes += localWrittenBytes; // 更新总的已写出的字节数
                    if (expectedWrittenBytes == 0) { // 若已全部写完，则设置done为true，退出
                        done = true;
                        break;
                    }
                }
                break;
            default:
                for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
                    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt); // gathering写，返回已写字节数
                    if (localWrittenBytes == 0) {
                        setOpWrite = true; // 不可写，设置OP_WRITE标志
                        break;
                    }
                    expectedWrittenBytes -= localWrittenBytes; // 更新剩余需写出的字节数expectedWrittenBytes
                    writtenBytes += localWrittenBytes; // 更新总的已写出的字节数
                    if (expectedWrittenBytes == 0) { // 若已全部写完，则设置done为true，退出
                        done = true;
                        break;
                    }
                }
                break;
        }

        // Release the fully written buffers, and update the indexes of the partially written buffer.
        // 移除所有已写的Entry，对于部分写的Entry，更新readerIndex
        in.removeBytes(writtenBytes);

        if (!done) {
            // Did not write all buffers completely.
            incompleteWrite(setOpWrite); // 不完全写
            break;
        }
    }
}
```

doWrite()方法首先判断flushed Entry的个数，如果为0，表示所有数据都已写，所以清除SelecttonKey.interestOps的OP_WRITE标志，直接返回。

```java
int size = in.size(); // 当前待写出的Entry个数
if (size == 0) {
    // All written so clear OP_WRITE
    clearOpWrite(); // 所有数据都已写，所以清除OP_WRITE标志
    break;
}

protected final void clearOpWrite() {
    final SelectionKey key = selectionKey();
    if (!key.isValid()) {
        return;
    }
    final int interestOps = key.interestOps();
    if ((interestOps & SelectionKey.OP_WRITE) != 0) {
        key.interestOps(interestOps & ~SelectionKey.OP_WRITE); // 感兴趣的事件集去除OP_WRITE事件
    }
}
```

然后获取标记为flushed的Entrys(ByteBuf)对应的JDK ByteBuffer[]数组，得到ByteBuffer元素的个数以及预期的写字节数。

```java
ByteBuffer[] nioBuffers = in.nioBuffers(); // 获取标记为flushed的Entrys(ByteBuf)对应的JDK ByteBuffer[]数组
int nioBufferCnt = in.nioBufferCount(); // 获取标记为flushed的Entrys(ByteBuf)对应的JDK ByteBuffer[]数组中，ByteBuffer元素的个数
long expectedWrittenBytes = in.nioBufferSize(); // 预期的写字节数
```

接着进入swich语句，根据nioBufferCnt的值，进入不同的逻辑写出数据。

> nioBufferCnt为0

如果nioBufferCnt为0，表示flushed Entry的msg不是ByteBuf，可能是FileRegion，则调用父类AbstractNioByteChannel的doWrite(ChannelOutboundBuffer)方法写出数据:

```java
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    int writeSpinCount = -1; // 一次写操作，总的自旋次数

    boolean setOpWrite = false; // 是否设置OP_WRITE标志
    for (;;) {
        Object msg = in.current();  // 拿到第一个需要flush的节点的数据
        if (msg == null) {
            // Wrote all messages.
            clearOpWrite(); // 所有数据已写出，清除OP_WRITE标记
            // Directly return here so incompleteWrite(...) is not called.
            return;
        }

        if (msg instanceof ByteBuf) { // 写出ByteBuf
            // 强转为ByteBuf，若发现没有数据可读，直接删除该节点
            ByteBuf buf = (ByteBuf) msg;
            int readableBytes = buf.readableBytes(); // 可读字节数
            if (readableBytes == 0) {
                in.remove(); // 移除当前Entry
                continue;
            }

            boolean done = false; // 表示当前Entry msg的数据是否已经全部写出
            long flushedAmount = 0; // 已写出的字节数
            if (writeSpinCount == -1) {  // 拿到总的自旋迭代次数，默认16
                writeSpinCount = config().getWriteSpinCount(); // 16
            }
            // 自旋，将当前节点写出
            for (int i = writeSpinCount - 1; i >= 0; i --) {
                int localFlushedAmount = doWriteBytes(buf); // 返回写出的字节数
                if (localFlushedAmount == 0) { // JDK底层不可写，setOpWrite设置为true
                    setOpWrite = true;
                    break;
                }

                flushedAmount += localFlushedAmount;
                if (!buf.isReadable()) { // 数据已写完
                    done = true;
                    break;
                }
            }

            in.progress(flushedAmount); // 记录写出的字节数

            if (done) {
                in.remove();  // 当前节点已写出，删除节点
            } else {
                // Break the loop and so incompleteWrite(...) is called.
                break; // 当前节点未写完，退出循环
            }
        } else if (msg instanceof FileRegion) {
            FileRegion region = (FileRegion) msg;
            boolean done = region.transferred() >= region.count(); // 是否已写完毕

            if (!done) { // 未写出完毕
                long flushedAmount = 0; // 已写出的字节数
                if (writeSpinCount == -1) { // 拿到总的自旋次数
                    writeSpinCount = config().getWriteSpinCount();
                }

                for (int i = writeSpinCount - 1; i >= 0; i--) {
                    long localFlushedAmount = doWriteFileRegion(region); //写出数据，返回已写字节数
                    if (localFlushedAmount == 0) {
                        setOpWrite = true; // 不可写，设置OP_WRITE标志
                        break;
                    }

                    flushedAmount += localFlushedAmount;
                    if (region.transferred() >= region.count()) {
                        done = true; // 传输完毕
                        break;
                    }
                }

                in.progress(flushedAmount); // 记录写出的字节数
            }

            if (done) {
                in.remove(); // 当前节点已写出，删除节点
            } else {
                // Break the loop and so incompleteWrite(...) is called.
                break;  // 当前节点未写完，退出循环
            }
        } else {
            // Should not reach here.
            throw new Error();
        }
    }
    incompleteWrite(setOpWrite); // setOpWrite: true
}
```

AbstractNioByteChannel的doWrite(ChannelOutboundBuffer)同样先判断标记为flushed的Entry是否已全部写出，如果已完全写出，则直接清除OP_WRITE标记，返回。下面分析FileRegion写出逻辑：在写出FileRegion时，通过自旋的方式，调用doWriteFileRegion方法将数据写出到底层JDK channel，同时判断每次写出的字节数，如果字节数localFlushedAmount为0，表示channel不可写，则直接退出自旋逻辑；否则自旋直到数据全部写出。假设channel不可写，则调用incompleteWrite(setOpWrite)方法(setOpWrite为true)：

```java
protected final void incompleteWrite(boolean setOpWrite) {
    // Did not write completely. 数据没有完全写完
    if (setOpWrite) {
        setOpWrite(); // SelectionKey interestOps设置OP_WRITE，表示写pending(channel现在不可写出)
    } else {
        // Schedule flush again later so other tasks can be picked up in the meantime
        Runnable flushTask = this.flushTask;
        if (flushTask == null) {
            flushTask = this.flushTask = new Runnable() {
                @Override
                public void run() {
                    flush(); // 再次flush
                }
            };
        }
        eventLoop().execute(flushTask);
    }
}
```

调用incompleteWrite()方法，即表示数据未全部写出，则调用`setOpWrite()`将SelectionKey interestOps包含OP_WRITE标志，表示写pending(channel现在不可写出)。

> nioBufferCnt为1

如果nioBufferCnt为1，表示flushed Entrys的ByteBufs对应的JDK ByteBuffer只有一个:

```java
// Only one ByteBuf so use non-gathering write 只有一个ByteBuffer，使用非gathering写
ByteBuffer nioBuffer = nioBuffers[0];
for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) { // 自旋写，writeSpinCount: 16
    final int localWrittenBytes = ch.write(nioBuffer); // 已写字节数
    if (localWrittenBytes == 0) {
        setOpWrite = true; // 写不出去，设置OP_WRITE标志
        break;
    }
    expectedWrittenBytes -= localWrittenBytes; // 更新剩余需写出的字节数expectedWrittenBytes
    writtenBytes += localWrittenBytes; // 更新总的已写出的字节数
    if (expectedWrittenBytes == 0) { // 若已全部写完，则设置done为true，退出
        done = true;
        break;
    }
}
```

于是通过自旋的方式调用JDK API将数据写出。写出过程中，同样会判断channel是否可写，即`localWrittenBytes == 0`，如果成立，则退出循环，此时done为false；若成功写出，此时done为true。

> nioBufferCnt>1

```java
for (int i = config().getWriteSpinCount() - 1; i >= 0; i --) {
    final long localWrittenBytes = ch.write(nioBuffers, 0, nioBufferCnt); // gathering写，返回已写字节数
    if (localWrittenBytes == 0) {
        setOpWrite = true; // 不可写，设置OP_WRITE标志
        break;
    }
    expectedWrittenBytes -= localWrittenBytes; // 更新剩余需写出的字节数expectedWrittenBytes
    writtenBytes += localWrittenBytes; // 更新总的已写出的字节数
    if (expectedWrittenBytes == 0) { // 若已全部写完，则设置done为true，退出
        done = true;
        break;
    }
}
break;
```

如果nioBufferCnt>1，表示flushed Entrys的ByteBufs对应的JDK ByteBuffer大于一个，则调用JDK API通过自旋+gathering write的方式写出数据。写出过程中，同样会判断channel是否可写，即`localWrittenBytes == 0`，如果成立，则退出循环，此时done为false；若成功写出，此时done为true。

##### 3.设置写状态

在NioSocketChannel调用doWrite(ChannelOutboundBuffer)方法写出数据的最后，会设置channel的写状态：

```java
// Release the fully written buffers, and update the indexes of the partially written buffer.
// 移除所有已写的Entry，对于部分写的Entry，更新readerIndex
in.removeBytes(writtenBytes);
```

这里会调用`in.removeBytes(writtenBytes)`，移除所有已写的Entry，对于部分写的Entry，更新readerIndex：

```java
public void removeBytes(long writtenBytes) {
    for (;;) {
        Object msg = current(); // 获取第一个flushed节点的数据
        if (!(msg instanceof ByteBuf)) {
            assert writtenBytes == 0;
            break;
        }

        final ByteBuf buf = (ByteBuf) msg;
        final int readerIndex = buf.readerIndex(); // 读指针
        final int readableBytes = buf.writerIndex() - readerIndex;// 可读字节数

        if (readableBytes <= writtenBytes) {
            if (writtenBytes != 0) {
                progress(readableBytes); // 记录写出的字节数
                writtenBytes -= readableBytes;
            }
            remove(); // 移除当前Entry
        } else { // readableBytes > writtenBytes
            if (writtenBytes != 0) { // 没写完一个ByteBuf
                buf.readerIndex(readerIndex + (int) writtenBytes); // 设置读指针
                progress(writtenBytes);
            }
            break;
        }
    }
    clearNioBuffers(); // 清空统计的ByteBuffer[]数组元素
}
```

在removeBytes()方法中，根据已写出的字节数writtenBytes，通过不断循环，对于已写出的Entry，进行删除Entry操作；对于不完全写出的Entry，更新读指针。

Entry删除操作如下：

```java
public boolean remove() {
    Entry e = flushedEntry;
    if (e == null) {
        clearNioBuffers();
        return false;
    }
    Object msg = e.msg;

    ChannelPromise promise = e.promise;
    int size = e.pendingSize;

    removeEntry(e); // 删除Entry

    if (!e.cancelled) {
        // only release message, notify and decrement if it was not canceled before.
        ReferenceCountUtil.safeRelease(msg); // 释放内存
        safeSuccess(promise);
        decrementPendingOutboundBytes(size, false, true); // 设置写状态
    }

    // recycle the entry
    e.recycle(); // 回收Entry到对象池

    return true;
}
```

remove()方法中实际调用了 removeEntry(e)方法删除Entry:

```java
private void removeEntry(Entry e) {
    if (-- flushed == 0) { // 如果flushed=0，表示待写的entry为0
        // processed everything 已处理完所有标记的Entry
        flushedEntry = null;
        if (e == tailEntry) {
            tailEntry = null;
            unflushedEntry = null;
        }
    } else {
        flushedEntry = e.next; // flushedEntry指向下一个待写的Entry
    }
}
```

然后在删除Entry之后释放msg内存并设置channel写状态：

```java
ReferenceCountUtil.safeRelease(msg); // 释放内存
safeSuccess(promise);
decrementPendingOutboundBytes(size, false, true); // 设置写状态
```

decrementPendingOutboundBytes()方法设置写状态逻辑如下：

```java
private void decrementPendingOutboundBytes(long size, boolean invokeLater, boolean notifyWritability) {
    if (size == 0) {
        return;
    }
		 // totalPendingSize减去size
    long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, -size);
    if (notifyWritability && newWriteBufferSize < channel.config().getWriteBufferLowWaterMark()) { // 32kB
        setWritable(invokeLater); // 设置可写
    }
}
```

同样先更新totalPendingSize，然后判断totalPendingSize是否小于WriteBufferLowWaterMark(默认32KB)，如果成立，则调用`setWritable(invokeLater)`设置可写状态：

```java
private void setWritable(boolean invokeLater) {
    for (;;) {
        final int oldValue = unwritable;
        final int newValue = oldValue & ~1; // 设置为0，可写
        if (UNWRITABLE_UPDATER.compareAndSet(this, oldValue, newValue)) {
            if (oldValue != 0 && newValue == 0) {
                fireChannelWritabilityChanged(invokeLater); // 不可写->可写，传播写状态变化
            }
            break;
        }
    }
}
```

setWritable()方法中将unwritable变量设置为0，表示channel可写。如果unwritable由1变为0，即不可写->可写，则传播写状态变化事件。

调用`in.removeBytes(writtenBytes)`方法之后，NioSocketChannel.doWrite(ChannelOutboundBuffer)方法会根据变量done的值(即数据是否完全写完)，调用`incompleteWrite(setOpWrite)`方法。假设done为false，即数据不完全写完，则进入`incompleteWrite(setOpWrite)`逻辑：

```java
if (!done) {
    // Did not write all buffers completely.
    incompleteWrite(setOpWrite); // 不完全写
    break;
}
protected final void incompleteWrite(boolean setOpWrite) {
    // Did not write completely. 数据没有完全写完
    if (setOpWrite) {
        setOpWrite(); // SelectionKey interestOps设置OP_WRITE，表示写pending(channel现在不可写出)
    } else {
        // Schedule flush again later so other tasks can be picked up in the meantime
        Runnable flushTask = this.flushTask;
        if (flushTask == null) {
            flushTask = this.flushTask = new Runnable() {
                @Override
                public void run() {
                    flush(); // 再次flush
                }
            };
        }
        eventLoop().execute(flushTask);
    }
}
```

incompleteWrite()会调用setOpWrite()方法将SelectionKey interestOps包含OP_WRITE标记，表示写pending(channel现在不可写出)。

至此，writeAndFlush写数据流程分析完毕。

### 五、面试问题

##### 如何把对象变成字节流，最终写道socket底层？

​	用户需要调用`ctx.channel.writeAndFlush()`代码写出Java对象，Java对象经过pipeline，会被MessageToByteEncoder根据自定义协议编码为ByteBuf，然后该ByteBuf传递到HeadContext。对于write事件，HeadContext将ByteBuf缓存到ChannelOutboundBuffer；对于flush事件，HeadContext将ChannelOutboundBuffer中的ByteBuf缓存写出到底层JDK channel。

### 参考文章

- [netty源码分析之writeAndFlush全解析](https://www.jianshu.com/p/feaeaab2ce56)
- [深入浅出Netty write](https://www.jianshu.com/p/1ad424c53e80)





