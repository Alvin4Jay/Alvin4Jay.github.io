---
layout:     post
title:      Netty NioEventLoop分析
subtitle:   NioEventLoop
date:       2019-08-19
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Netty
---


# Netty NioEventLoop分析

本文分析Netty最核心的Reactor线程(NIO线程)——NioEventLoop的**创建、启动和执行过程**。

### 一、NioEventLoopGroup/NioEventLoop的创建

NioEventLoop的创建，首先需要看NioEventLoopGroup的创建过程，因为NioEventLoop是在NioEventLoopGroup初始化的过程中创建的。

```java
public NioEventLoopGroup() {
    this(0);
}
public NioEventLoopGroup(int nThreads) {
    this(nThreads, (Executor) null);
}
public NioEventLoopGroup(int nThreads, Executor executor) {
    this(nThreads, executor, SelectorProvider.provider());
}
public NioEventLoopGroup(
        int nThreads, Executor executor, final SelectorProvider selectorProvider) {
    this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}
public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,
                         final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
    super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
}
```

NioEventLoopGroup初始化时最终调用到父类MultithreadEventLoopGroup的构造器。这里判断指定的线程个数是否为0，如果为0，则使用默认的线程数，即DEFAULT_EVENT_LOOP_THREADS，该常量定义如下：

```java
private static final int DEFAULT_EVENT_LOOP_THREADS;

static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));

    if (logger.isDebugEnabled()) {
        logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
    }
}
```

默认情况下DEFAULT_EVENT_LOOP_THREADS值为**2倍的CPU核数**。接着继续调用父类MultithreadEventExecutorGroup的构造器：

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
    this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
}
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                        EventExecutorChooserFactory chooserFactory, Object... args) {
    if (nThreads <= 0) {
        throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
    }

    if (executor == null) {
        executor = new ThreadPerTaskExecutor(newDefaultThreadFactory()); // 1.创建 线程创建器
    }

    children = new EventExecutor[nThreads]; // 初始化所有事件执行器

    for (int i = 0; i < nThreads; i ++) {
        boolean success = false;
        try {
            children[i] = newChild(executor, args); // 2.创建NioEventLoop
            success = true;
        } catch (Exception e) {
            // TODO: Think about if this is a good exception type
            throw new IllegalStateException("failed to create a child event loop", e);
        } finally {
            if (!success) { // 处理创建异常
                for (int j = 0; j < i; j ++) {
                    children[j].shutdownGracefully();
                }

                for (int j = 0; j < i; j ++) {
                    EventExecutor e = children[j];
                    try {
                        while (!e.isTerminated()) { // 等待终止
                            e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException interrupted) {
                        // Let the caller handle the interruption.
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            }
        }
    }

    chooser = chooserFactory.newChooser(children); // 3.创建线程选择器

    final FutureListener<Object> terminationListener = new FutureListener<Object>() {
        @Override
        public void operationComplete(Future<Object> future) throws Exception {
            if (terminatedChildren.incrementAndGet() == children.length) {
                terminationFuture.setSuccess(null);
            }
        }
    };

    for (EventExecutor e: children) {
        // 给每个EventExecutor的终止Future添加监听器
        e.terminationFuture().addListener(terminationListener);
    }

    Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
    Collections.addAll(childrenSet, children);
    readonlyChildren = Collections.unmodifiableSet(childrenSet);
}
```

MultithreadEventExecutorGroup构造器中的逻辑主要分为三部分：

- 创建线程创建器ThreadPerTaskExecutor；
- 创建NioEventLoop；
- 创建线程选择器EventExecutorChooser。

##### 1.创建线程创建器ThreadPerTaskExecutor

```java
if (executor == null) {
    executor = new ThreadPerTaskExecutor(newDefaultThreadFactory()); // 创建 线程创建器
}
```

由于构造器传入的executor为null，因此这里会创建ThreadPerTaskExecutor执行器，该执行器是创建NioEventLoop底层线程实体FastThreadLocalThread的创建器。ThreadPerTaskExecutor创建时先new了一个DefaultThreadFactory:

```java
protected ThreadFactory newDefaultThreadFactory() {
    return new DefaultThreadFactory(getClass(), Thread.MAX_PRIORITY);
}
```

跟进去DefaultThreadFactory的构造器：

```java
public DefaultThreadFactory(Class<?> poolType, int priority) { // poolType: NioEventLoopGroup
    this(poolType, false, priority);
}
public DefaultThreadFactory(Class<?> poolType, boolean daemon, int priority) {
    this(toPoolName(poolType), daemon, priority);
}
```

DefaultThreadFactory构造器中先调用了`toPoolName(poolType)`:

```java
public static String toPoolName(Class<?> poolType) {
    if (poolType == null) {
        throw new NullPointerException("poolType");
    }

    String poolName = StringUtil.simpleClassName(poolType); // NioEventLoopGroup
    switch (poolName.length()) {
        case 0:
            return "unknown";
        case 1:
            return poolName.toLowerCase(Locale.US);
        default:
            if (Character.isUpperCase(poolName.charAt(0)) && Character.isLowerCase(poolName.charAt(1))) {
                // nioEventLoopGroup
                return Character.toLowerCase(poolName.charAt(0)) + poolName.substring(1); 
            } else {
                return poolName;
            }
    }
}
```

toPoolName方法的作用是返回**nioEventLoopGroup**这个值。然后继续调用DefaultThreadFactory的构造器:

```java
public DefaultThreadFactory(String poolName, boolean daemon, int priority) {
    this(poolName, daemon, priority, System.getSecurityManager() == null ?
            Thread.currentThread().getThreadGroup() : System.getSecurityManager().getThreadGroup());
}
public DefaultThreadFactory(String poolName, boolean daemon, int priority, ThreadGroup threadGroup) {
    if (poolName == null) {
        throw new NullPointerException("poolName");
    }
    if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
        throw new IllegalArgumentException(
                "priority: " + priority + " (expected: Thread.MIN_PRIORITY <= priority <= Thread.MAX_PRIORITY)");
    }
		// 线程名前缀：nioEventLoopGroup-poolId-
    prefix = poolName + '-' + poolId.incrementAndGet() + '-'; 
    this.daemon = daemon;
    this.priority = priority;
    this.threadGroup = threadGroup;
}
```

在上面的构造器中，主要是构造了DefaultThreadFactory将创建的线程的名字前缀：`nioEventLoopGroup-poolId-`。DefaultThreadFactory初始化完成后，再来看一下它创建线程的方法:

```java
public Thread newThread(Runnable r) {
    // name: nioEventLoopGroup-poolId-nextId
    // 创建的线程实体: FastThreadLocalThread
    Thread t = newThread(new DefaultRunnableDecorator(r), prefix + nextId.incrementAndGet());
    try {
        if (t.isDaemon()) {
            if (!daemon) {
                t.setDaemon(false);
            }
        } else {
            if (daemon) {
                t.setDaemon(true);
            }
        }

        if (t.getPriority() != priority) {
            t.setPriority(priority);
        }
    } catch (Exception ignored) {
        // Doesn't matter even if failed to set.
    }
    return t;
}
```

newThread方法首先将Runnable任务包装成DefaultRunnableDecorator实例:

```java
private static final class DefaultRunnableDecorator implements Runnable {

    private final Runnable r;

    DefaultRunnableDecorator(Runnable r) {
        this.r = r;
    }

    @Override
    public void run() {
        try {
            r.run();
        } finally {
            FastThreadLocal.removeAll(); // 移除线程的所有本地变量副本
        }
    }
}
```

可以看到DefaultRunnableDecorator只是在Runnable任务执行结束后调用了`FastThreadLocal.removeAll()`。然后调用newThread方法创建线程实体:

```java
protected Thread newThread(Runnable r, String name) {
    return new FastThreadLocalThread(threadGroup, r, name);
}
```

可以知道DefaultThreadFactory所创建的线程实体是FastThreadLocalThread，线程名具有`nioEventLoopGroup-poolId-nextId`这样的格式。

以上简要说明了DefaultThreadFactory的初始化过程和创建线程的逻辑，下面继续介绍ThreadPerTaskExecutor的创建过程：

```java
public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory; // DefaultThreadFactory

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        // threadFactory: DefaultThreadFactory，创建线程并启动
        // 每次执行任务，都会创建一个线程实体FastThreadLocalThread
        threadFactory.newThread(command).start();
    }
}
```

可以看到ThreadPerTaskExecutor每次在执行Runnable任务的时候都会使用DefaultThreadFactory创建一个FastThreadLocalThread实体并启动。因此ThreadPerTaskExecutor称为**线程创建器**。

##### 2.创建NioEventLoop

```java
children = new EventExecutor[nThreads]; // 初始化所有事件执行器

for (int i = 0; i < nThreads; i ++) {
    boolean success = false;
    try {
        children[i] = newChild(executor, args); // 创建NioEventLoop
        success = true;
    } catch (Exception e) {
        throw new IllegalStateException("failed to create a child event loop", e);
    } finally {
        // 省略部分代码
    }
}
```

通过调用newChild方法，依次创建NioEventLoop:

```java
protected EventLoop newChild(Executor executor, Object... args) throws Exception {
    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2]);
}
```

下面看NioEventLoop的初始化过程:

```java
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
             SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler) {
    super(parent, executor, false, DEFAULT_MAX_PENDING_TASKS, rejectedExecutionHandler);
    if (selectorProvider == null) {
        throw new NullPointerException("selectorProvider");
    }
    if (strategy == null) {
        throw new NullPointerException("selectStrategy");
    }
    provider = selectorProvider;
    selector = openSelector(); // 打开事件轮询器，key set优化
    selectStrategy = strategy;
}
```

NioEventLoop构造器首先调用父类的构造器，然后打开事件轮询器:

```java
private Selector openSelector() {
    final Selector selector;
    try {
        selector = provider.openSelector(); // JDK API打开事件轮询器
    } catch (IOException e) {
        throw new ChannelException("failed to open a new selector", e);
    }

    if (DISABLE_KEYSET_OPTIMIZATION) { // 默认false，key set默认优化
        return selector;
    }

    // 使用数组替换HashSet，使add操作时间复杂度从O(n)降到O(1)
    final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();

    // selectorImplClass -> sun.nio.ch.SelectorImpl
    Object maybeSelectorImplClass = Class.forName("sun.nio.ch.SelectorImpl",false, // 不初始化
                        PlatformDependent.getSystemClassLoader());

    final Class<?> selectorImplClass = (Class<?>) maybeSelectorImplClass;

    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");
    selectedKeysField.setAccessible(true);
    publicSelectedKeysField.setAccessible(true);
		// 反射设置SelectedSelectionKeySet
    selectedKeysField.set(selector, selectedKeySet); 
    publicSelectedKeysField.set(selector, selectedKeySet);

    selectedKeys = selectedKeySet; // selectedKeySet设置到selectedKeys变量
    logger.trace("instrumented a special java.util.Set into: {}", selector);

    return selector;
}
```

openSelector方法中首先调用`provider.openSelector()`获取Selector实例。然后创建了SelectedSelectionKeySet实例(Set)，并通过反射将该实例设置到Selector实现类的`selectedKeys`、`publicSelectedKeys`成员变量上，这样Selector在轮询到IO事件时，就会将对应的`SelectionKey`添加到SelectedSelectionKeySet实例中。为什么要这么做呢？可以从SelectedSelectionKeySet的结果来看：

```java
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {

    private SelectionKey[] keysA; // 一个数组已足够
    private int keysASize;
    private SelectionKey[] keysB;
    private int keysBSize;
    private boolean isA = true;

    SelectedSelectionKeySet() {
        keysA = new SelectionKey[1024];
        keysB = keysA.clone();
    }

    @Override
    public boolean add(SelectionKey o) {
        if (o == null) {
            return false;
        }

        if (isA) {
            int size = keysASize;
            keysA[size ++] = o;
            keysASize = size;
            if (size == keysA.length) {
                doubleCapacityA();
            }
        } else {
            int size = keysBSize;
            keysB[size ++] = o;
            keysBSize = size;
            if (size == keysB.length) {
                doubleCapacityB();
            }
        }

        return true;
    }

    private void doubleCapacityA() {
        SelectionKey[] newKeysA = new SelectionKey[keysA.length << 1];
        System.arraycopy(keysA, 0, newKeysA, 0, keysASize);
        keysA = newKeysA;
    }

    private void doubleCapacityB() {
        SelectionKey[] newKeysB = new SelectionKey[keysB.length << 1];
        System.arraycopy(keysB, 0, newKeysB, 0, keysBSize);
        keysB = newKeysB;
    }

    SelectionKey[] flip() {
        if (isA) {
            isA = false;
            keysA[keysASize] = null;
            keysBSize = 0; // 切换后，B从0位置开始
            return keysA;
        } else {
            isA = true;
            keysB[keysBSize] = null;
            keysASize = 0; // 切换后，A从0位置开始
            return keysB;
        }
    }

    @Override
    public int size() {
        if (isA) {
            return keysASize;
        } else {
            return keysBSize;
        }
    }

}
```

SelectedSelectionKeySet是一个Set，但是其内部实现是数组。因此在添加SelectionKey时，实际上是添加到SelectedSelectionKeySet内部的数组中，相比于JDK默认使用的HashSet高效，并且取出SelectionKey的时候也是一样。openSelector方法中最后将SelectedSelectionKeySet赋值给成员变量`selectedKeys`。

在NioEventLoop的构造器中，将调用其父类SingleThreadEventLoop的构造器:

```java
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                boolean addTaskWakesUp, int maxPendingTasks,
                                RejectedExecutionHandler rejectedExecutionHandler) {
    super(parent, executor, addTaskWakesUp, maxPendingTasks, rejectedExecutionHandler);
    tailTasks = newTaskQueue(maxPendingTasks); // MpscQueue
}
```

SingleThreadEventLoop的构造器先调用父类SingleThreadEventExecutor的构造器，然后构造一个`tailTasks`任务队列，这是一个MpscQueue，即多生产者单消费者队列:

```java
protected Queue<Runnable> newTaskQueue(int maxPendingTasks) {
    // MpscQueue: Multi-Producers(外部线程)-Single-Consumer(Netty NioEventLoop) Queue
    return PlatformDependent.newMpscQueue(maxPendingTasks);
}
```

接着看SingleThreadEventExecutor的构造器逻辑:

```java
protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                    boolean addTaskWakesUp, int maxPendingTasks,
                                    RejectedExecutionHandler rejectedHandler) {
    super(parent);
    this.addTaskWakesUp = addTaskWakesUp; // false
    this.maxPendingTasks = Math.max(16, maxPendingTasks);
    // 保存执行器ThreadPerTaskExecutor，用于创建NioEventLoop底层的线程
    this.executor = ObjectUtil.checkNotNull(executor, "executor"); 
    taskQueue = newTaskQueue(this.maxPendingTasks); // 创建MpscQueue
    rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
}
```

在SingleThreadEventExecutor构造器中，主要是保存了传入的线程创建器ThreadPerTaskExecutor，用于创建NioEventLoop底层的线程。同时创建了一个任务队列`taskQueue`，同样是MpscQueue多生产者单消费者队列。

```java
protected Queue<Runnable> newTaskQueue(int maxPendingTasks) {
    // MpscQueue: Multi-Producers(外部线程)-Single-Consumer(Netty NioEventLoop) Queue
    return PlatformDependent.newMpscQueue(maxPendingTasks);
}
```

因此，NioEventLoop创建过程中主要做了以下事：

- 保存线程创建器ThreadPerTaskExecutor；
- 创建两个MpscQueue——tailTasks、**taskQueue**；
- 创建一个Selector。

##### 3.创建线程选择器EventExecutorChooser

```java
chooser = chooserFactory.newChooser(children); // 创建线程选择器

public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) { // 判断执行器个数，是否为2的幂次
        return new PowerOfTowEventExecutorChooser(executors); // 优化
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}
```

在调用DefaultEventExecutorChooserFactory.newChooser方法创建EventExecutorChooser时，首先判断已创建的NioEventLoop个数，如果为2的幂次，则创建PowerOfTowEventExecutorChooser选择器，否则创建GenericEventExecutorChooser选择器。这里是一个优化的地方。

```java
// PowerOfTowEventExecutorChooser.next()
public EventExecutor next() {
    return executors[idx.getAndIncrement() & (executors.length - 1)]; 
}
// GenericEventExecutorChooser.next()
public EventExecutor next() {
    return executors[Math.abs(idx.getAndIncrement() % executors.length)];
}
```

两种实现的主要区别在于next()方法返回EventExecutor(NioEventLoop)的逻辑，前者通过idx递增并且位运算取余的方式、后者使用idx递增并且%取余的方式，可见位运算相比于%取余性能更优，即如果已创建的NioEventLoop个数为2的幂次，可以使用优化的方式获取NioEventLoop。

### 二、NioEventLoop的启动

##### 1.Boss NioEventLoop

从[Netty服务端启动流程分析](https://xuanjian1992.top/2019/07/28/Netty服务端启动流程分析/)这篇文章可以看出，最早触发Boss NioEventLoop启动的逻辑是服务端Channel注册到Selector以及绑定NioEventLoop的时候，如下代码所示:

```java
// AbstractUnsafe类
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
		// 省略部分代码
    // 绑定服务端Channel到EventLoop
    AbstractChannel.this.eventLoop = eventLoop;

    if (eventLoop.inEventLoop()) { // false
        register0(promise);
    } else {
        try {
            // 将任务放入任务队列后，主线程main返回。此处会创建Nio线程并启动，然后处理该注册任务
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
           // 省略部分代码
        }
    }
}
```

由于`eventLoop.inEventLoop()`返回`false`，因此执行else分支的逻辑。该分支逻辑已在[Netty服务端启动流程分析#注册NioServerSocketChannel到Selector](https://xuanjian1992.top/2019/07/28/Netty服务端启动流程分析/)详细介绍，主要逻辑是创建NioEventLoop底层的FastThreadLocalThread线程并启动，并将该Runnable任务放入`taskQueue`，然后执行该注册任务。

##### 2.Worker NioEventLoop

最早触发Worker NioEventLoop启动的时机是在客户端新连接接入，通过EventExecutorChooser绑定一个NioEventLoop的时候，也就是NioSocketChannel注册到Selector并且绑定NioEventLoop的时候。

```java
// ServerBootstrapAcceptor类
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    final Channel child = (Channel) msg; // NioSocketChannel

    try {
        // NioSocketChannel绑定NioEventLoop
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

详细的逻辑在下篇文章中介绍。

### 三、NioEventLoop的执行

在`SingleThreadEventExecutor.doStartThread`方法首次启动NioEventLoop对应底层线程的时候，将执行以下代码:

```java
SingleThreadEventExecutor.this.run(); // NIO线程核心逻辑在这里
```

以上代码的实现在NioEventLoop中，这是NioEventLoop NIO线程的核心逻辑所在:

```java
protected void run() {
    for (;;) {
        try {
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE: // 重试IO循环
                    continue;
                case SelectStrategy.SELECT:
                    // 1.检查是否有IO事件
                    // select方法传入旧wakeup状态(即调用select方法前是否已将wakeup置为true)，并将wakeup设置为false(进入阻塞式select)
                    select(wakenUp.getAndSet(false));

                    if (wakenUp.get()) { // select(boolean)操作之后是否已将wakeup置为true
                        // 如果wakenUp为true，则调用selector.wakeup()，于是接下来的一次selector.select(...)操作不会阻塞
                        selector.wakeup();
                    }
                default: // >= 0 表示有任务需要处理(hasTask)
                    // fallthrough(落空)
            }

            cancelledKeys = 0; // 已取消注册的channel个数
            needsToSelectAgain = false; // 是否需要再次select
            final int ioRatio = this.ioRatio;
            if (ioRatio == 100) {
                try {
                    processSelectedKeys();
                } finally {
                    // Ensure we always run tasks.
                    runAllTasks();
                }
            } else {
                final long ioStartTime = System.nanoTime();
                try {
                    processSelectedKeys(); // 2.处理IO事件
                } finally {
                    // Ensure we always run tasks.
                    final long ioTime = System.nanoTime() - ioStartTime;
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio); // 3.处理异步任务队列
                }
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
    }
}
```

该run方法是一个无限循环，NIO线程不断循环做了以下事：

- 轮询注册到NIO线程对应的Selector上的所有Channel的IO事件；
- 处理IO事件；
- 处理任务队列(普通任务、定时任务等)。

可以用下面这幅图表示(来源: [netty源码分析之揭开reactor线程的面纱](https://www.jianshu.com/p/0d0eece6d467))：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/NioEventLoop-1.jpg?x-oss-process=style/markdown-pic)

##### 1.轮询IO事件

```java
// 检查是否有IO事件
// select方法传入旧wakeup状态(即调用select方法前是否已将wakenUp置为true)，并将wakeup设置为false(进入阻塞式select)
select(wakenUp.getAndSet(false));
if (wakenUp.get()) { // select(boolean)操作之后是否已将wakenUp置为true
    // 如果wakenUp为true，则调用selector.wakeup()，于是接下来的一次selector.select(...)操作不会阻塞
    selector.wakeup();
}
```

wakenUp变量表示是否应该唤醒阻塞的select操作。Netty在进行一次新的select之前，都会将wakenUp设置成false，标志着新的一轮select的开始。下面具体分析select操作的过程：

```java
// oldWakenUp: 旧的wakeup值，调用select方法前是否已将wakeup置为true
private void select(boolean oldWakenUp) throws IOException {
    Selector selector = this.selector;
    try {
        int selectCnt = 0; // select次数
        long currentTimeNanos = System.nanoTime();
        // delayNanos(currentTimeNanos): 获取将来最近会执行的调度任务的剩余时间(将要执行)
        long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
        for (;;) {
            // 1.如果定时任务时间到了，则中断本次轮询
            long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L; // ms
            if (timeoutMillis <= 0) {
                if (selectCnt == 0) { // 还未select
                    selector.selectNow();
                    selectCnt = 1;
                }
                break;
            }

            // 2.轮询过程中如果发现有任务加入，则中断本次轮询
            if (hasTasks() && wakenUp.compareAndSet(false, true)) {
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            // 阻塞式select操作，可能被wakeup操作唤醒
            int selectedKeys = selector.select(timeoutMillis);
            selectCnt ++;

            
            // 1.轮询到IO事件(selectedKeys != 0)
            // 2.oldWakenUp参数为true(进行select(boolean)之前已将wakenup置为true)
            // 3.用户主动唤醒(wakenUp.get())
            // 4.任务队列里面有任务(hasTasks())
            // 5.第一个定时任务需要被执行(hasScheduledTasks())
            if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
                break;
            }
            if (Thread.interrupted()) { // 线程被中断，直接退出select
                if (logger.isDebugEnabled()) {
                    logger.debug("Selector.select() returned prematurel(过早的) because " +
                            "Thread.currentThread().interrupt() was called. Use " +
                            "NioEventLoop.shutdownGracefully() to shutdown the NioEventLoop.");
                }
                selectCnt = 1;
                break;
            }

            long time = System.nanoTime(); // 当前时间
            if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                // select已超时，一次正常的select
                selectCnt = 1; // select完毕，将要退出循环(下一个循环判断超时)
            } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                    // 如果当前select次数大于512，则重建Selector
                    selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) { 
                logger.warn(
                        "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
                        selectCnt, selector);

                rebuildSelector(); // 重建Selector
                selector = this.selector; // 更新selector局部变量

                // Select again to populate selectedKeys.
                selector.selectNow();
                selectCnt = 1;
                break;
            }

            currentTimeNanos = time; // 更新当前时间
        }

        if (selectCnt > MIN_PREMATURE_SELECTOR_RETURNS) {
            if (logger.isDebugEnabled()) {
                logger.debug("Selector.select() returned prematurely {} times in a row for Selector {}.",
                        selectCnt - 1, selector);
            }
        }
    } catch (CancelledKeyException e) {
        if (logger.isDebugEnabled()) {
            logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                    selector, e);
        }
        // Harmless exception - log anyway
    }
}
```

下面分步骤分析select操作的逻辑：

> 1.如果定时任务时间到了，则中断本次轮询

```java
int selectCnt = 0; // select次数
long currentTimeNanos = System.nanoTime(); // 当前时间
// delayNanos(currentTimeNanos): 获取将来最近会执行的调度任务的剩余时间(将要执行)
long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);
for (;;) {
    // 1.如果定时任务时间到了，则中断本次轮询
    long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L; // ms
    if (timeoutMillis <= 0) {
        if (selectCnt == 0) { // 还未select
            selector.selectNow();
            selectCnt = 1;
        }
        break;
    }
}  
```

NioEventLoop中NIO线程的select操作也是一个for循环，在for循环第一步中，如果最近将要执行的定时任务需要执行时(超出0.5ms)，则中断本次轮询、跳出循环。如果跳出之前还没有进行过select操作，则进行一次非阻塞的selectNow操作。

这里说明一点，Netty里面定时任务队列是按照执行时间点(`deadlineNanos`)由近到远进行排序， `delayNanos(currentTimeNanos)`方法即取出第一个待执行定时任务相对于`currentTimeNanos`的延迟时间(还有多少时间执行任务)。

```java
protected long delayNanos(long currentTimeNanos) {
    ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
    if (scheduledTask == null) {
        return SCHEDULE_PURGE_INTERVAL; // 1s
    }

    return scheduledTask.delayNanos(currentTimeNanos);
}
```

> 2.轮询过程中如果发现有任务加入，则中断本次轮询

```java
if (hasTasks() && wakenUp.compareAndSet(false, true)) {
    selector.selectNow();
    selectCnt = 1;
    break;
}
```

Netty为了保证任务队列能够及时执行，在进行阻塞select操作的时候会判断任务队列是否为空，如果不为空，就执行一次非阻塞select操作，跳出循环。

> 3.阻塞式select操作，可能被wakeup操作唤醒

```java
int selectedKeys = selector.select(timeoutMillis);
selectCnt ++;

// 1.轮询到IO事件(selectedKeys != 0)
// 2.oldWakenUp参数为true(进行select(boolean)之前已将wakenup置为true)
// 3.用户主动唤醒(wakenUp.get())
// 4.任务队列里面有任务(hasTasks())
// 5.第一个定时任务需要被执行(hasScheduledTasks())
if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
    break;
}
```

执行到这一步，说明Netty任务队列里面没有任务，并且所有定时任务执行时间还未到，于是在这里进行一次阻塞式select操作，阻塞`timeoutMillis`时间。在阻塞过程中，有可能被外部线程唤醒。

> 外部线程调用execute方法添加任务

```java
public void execute(Runnable task) {
  	// ...
  
    if (!addTaskWakesUp && wakesUpForTask(task)) { // addTaskWakesUp: false
        wakeup(inEventLoop); // 唤醒selector, inEventLoop: false
    }
}
```

> 调用wakeup方法唤醒阻塞的Selector

```java
protected void wakeup(boolean inEventLoop) {
    if (!inEventLoop && wakenUp.compareAndSet(false, true)) {
        selector.wakeup(); // 如果是外部线程，且设置wakenUp为true，则唤醒select阻塞过程。
    }
}
```

可以看到，在外部线程添加任务的时候，会调用wakeup方法来唤醒 `selector.select(timeoutMillis)`。

阻塞式select操作结束之后，Netty又做了一系列的状态判断来决定是否中断本次轮询，中断本次轮询的条件有

1.轮询到IO事件(selectedKeys != 0)
2.oldWakenUp参数为true(进行select(boolean)之前已被将wakeup置为true)
3.用户主动唤醒(wakenUp.get())
4.任务队列里面有任务(hasTasks())
5.第一个定时任务需要被执行(hasScheduledTasks())

> 4.解决JDK NIO空轮询Bug

关于该Bug的描述见 [http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6595055)](http://bugs.java.com/bugdatabase/view_bug.do?bug_id=6595055)

该Bug会导致Selector一直空轮询，最终导致CPU 100%，NIO Server不可用，Netty的解决办法如下:

```java
long time = System.nanoTime(); // 当前时间
if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
    // select已超时，一次正常的select
    selectCnt = 1; // select完毕，将要退出循环(下一个循环判断超时)
} else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
        // 如果当前select次数大于512，则重建Selectors
        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) { 
    logger.warn(
            "Selector.select() returned prematurely {} times in a row; rebuilding Selector {}.",
            selectCnt, selector);

    rebuildSelector(); // 重建Selector
    selector = this.selector; // 更新selector局部变量

    // Select again to populate selectedKeys.
    selector.selectNow();
    selectCnt = 1;
    break;
}
```

Netty会在每次进行 `selector.select(timeoutMillis)`之前记录一下开始时间`currentTimeNanos`，在select之后记录一下结束时间，判断select操作是否至少持续了`timeoutMillis`时间（这里将`time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos`改成`time - currentTimeNanos >= TimeUnit.MILLISECONDS.toNanos(timeoutMillis)`或许更好理解一些）,
如果持续的时间大于等于`timeoutMillis`，说明这是一次有效的轮询，重置`selectCnt`标志；否则，表明该阻塞方法并没有阻塞这么长时间，可能触发了JDK的空轮询Bug，当空轮询的次数超过一个阀值的时候，默认是512，就开始重建Selector。

空轮询阀值相关的设置代码如下:

```java
int selectorAutoRebuildThreshold = SystemPropertyUtil.getInt("io.netty.selectorAutoRebuildThreshold", 512);
if (selectorAutoRebuildThreshold < MIN_PREMATURE_SELECTOR_RETURNS) {
    selectorAutoRebuildThreshold = 0;
}

SELECTOR_AUTO_REBUILD_THRESHOLD = selectorAutoRebuildThreshold;
```

下面简单描述一下Netty通过`rebuildSelector`方法来Fix空轮询Bug的过程，`rebuildSelector`方法其实很简单：new一个新的Selector，将之前注册到老的Selector上的的Channel重新转移到新的Selector上。主要代码如下:

```java
public void rebuildSelector() {
    final Selector oldSelector = selector;
    final Selector newSelector;
    newSelector = openSelector(); // 创建新Selector

    // 将所有Channel注册到新Selector
    int nChannels = 0;
    for (;;) {
        try {
            for (SelectionKey key: oldSelector.keys()) {
                Object a = key.attachment();
                try {
                    // key无效或者key对应的channel已注册到新Selector，则跳过处理
                    if (!key.isValid() || key.channel().keyFor(newSelector) != null) {
                        continue;
                    }

                    int interestOps = key.interestOps(); // 感兴趣的IO事件
                    key.cancel(); // 取消Channel在旧Selector的注册
                    // 注册到新Selector
                    SelectionKey newKey = key.channel().register(newSelector, interestOps, a); 
                    if (a instanceof AbstractNioChannel) {
                        // 更新Channel的selectionKey变量
                        ((AbstractNioChannel) a).selectionKey = newKey; 
                    }
                    nChannels ++;
                } catch (Exception e) {
                   // ......
                }
            }
        } catch (ConcurrentModificationException e) {
            // 只要执行过程中出现过一次并发修改selectionKeys异常，就重新开始转移
            continue;
        }

        break;
    }

    selector = newSelector; // 更新selector变量
    oldSelector.close();
}
```

首先，通过`openSelector()`方法创建一个新的Selector，然后执行一个死循环，只要执行过程中出现过一次并发修改SelectionKeys异常，就重新开始转移。

具体的转移步骤为

- 拿到有效的SelectionKey；

- 取消Channel在旧Selector的注册；

- 将Channel注册到新Selector；

- 更新Channel的selectionKey变量。

转移完成之后，就可以将原有的Selector废弃，后面所有的轮询都是在新的Selector进行。

总结NIO线程select步骤做的事情：不断地轮询是否有IO事件发生，并且在轮询的过程中不断检查是否有定时任务和普通任务需要处理，保证了Netty的任务队列中的任务得到有效执行，轮询过程顺带用一个计数器避开了JDK空轮询的Bug。

##### 2.处理IO事件

```java
processSelectedKeys(); 

private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized(selectedKeys.flip()); // 优化处理IO事件
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
```

处理IO事件时，Netty有两种选择，一种是处理优化过的selectedKeys，一种是正常的处理。selectedKeys是SelectedSelectionKeySet实例，上面已经介绍过使用SelectedSelectionKeySet的好处，且默认情况下NioEventLoop使用了这种优化，因此`selectedKeys != null`。下面进入processSelectedKeysOptimized方法的逻辑:

```java
// 处理SelectedKeys(优化方式)
private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
    for (int i = 0;; i ++) {
        // 1.取出IO事件SelectionKey以及对应的channel
        final SelectionKey k = selectedKeys[i];
        if (k == null) {
            break;
        }
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys[i] = null;

        final Object a = k.attachment(); // NioServerSocketChannel、NioSocketChannel
        // 2.处理该SelectionKey
        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        // 3.判断是否需要再来次轮询(大量Channel取消注册)
        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            for (;;) {
                i++;
                if (selectedKeys[i] == null) {
                    break;
                }
                selectedKeys[i] = null;
            }

            selectAgain(); // 再次轮询IO事件
            // Need to flip the optimized selectedKeys to get the right reference to the array
            // and reset the index to -1 which will then set to 0 on the for loop
            // to start over again.
            //
            // See https://github.com/netty/netty/issues/1523
            selectedKeys = this.selectedKeys.flip();
            i = -1;
        }
    }
}
```

processSelectedKeysOptimized方法可以分为三个步骤:

> 1.取出IO事件SelectionKey以及对应的channel

因为SelectedSelectionKeySet内部使用的是数组，因此processSelectedKeysOptimized方法中取出SelectionKey时遍历的是数组，相对于JDK原生的`HashSet`效率有所提高。拿到索引i对应的SelectionKey之后，将`selectedKeys[i]`置为null，便于GC回收。之后取出SelectionKey中的attachment，即Channel实例。

> 2,处理该SelectionKey和Channel

```java
if (a instanceof AbstractNioChannel) {
    processSelectedKey(k, (AbstractNioChannel) a);
}
```

SelectionKey中的attachment是AbstractNioChannel实例，这个可以从[Netty服务端启动流程分析](https://xuanjian1992.top/2019/07/28/Netty%E6%9C%8D%E5%8A%A1%E7%AB%AF%E5%90%AF%E5%8A%A8%E6%B5%81%E7%A8%8B%E5%88%86%E6%9E%90/)这篇文章的分析中看出。接下来调用processSelectedKey方法:

```java
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe(); // 与Channel绑定的unsafe
    if (!k.isValid()) { // SelectionKey无效时，关闭Channel
        final EventLoop eventLoop;
        try {
            eventLoop = ch.eventLoop();
        } catch (Throwable ignored) {
            return;
        }
        if (eventLoop != this || eventLoop == null) {
            return;
        }
        unsafe.close(unsafe.voidPromise()); // 关闭Channel
        return;
    }

    try {
        int readyOps = k.readyOps(); // 就绪的事件集
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) { // 处理客户端连接完成事件
            int ops = k.interestOps(); // 感兴趣的事件集
            ops &= ~SelectionKey.OP_CONNECT; // 感兴趣的事件集，去除OP_CONNECT事件
            k.interestOps(ops); // 更新感兴趣的事件集

            unsafe.finishConnect(); // 调用JDK SocketChannel.finishConnect()方法
        }

        if ((readyOps & SelectionKey.OP_WRITE) != 0) { // 处理写事件
            ch.unsafe().forceFlush();
        }

        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            // 对于NioServerSocketChannel，是NioMessageUnsafe.read()
            // 对于NioSocketChannel，是NioByteUnsafe.read()
            unsafe.read();
            if (!ch.isOpen()) {
                // Connection already closed - no need to handle write.
                return;
            }
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise()); // 出现异常，关闭Channel
    }
}
```

processSelectedKey方法主要处理SelectionKey就绪的IO事件，如客户端连接完成事件、读写事件、服务端Accept事件。各事件详细的处理流程将在后面文章中分析，这里简要说明一下:

1.对于Boss NioEventLoop来说，轮询到的是基本上就是Accept事件，后续的事情就是通过它的pipeline将连接扔给一个Worker NioEventLoop处理；
2.对于Worker NioEventLoop来说，轮询到的基本上都是IO读写事件，后续的事情就是通过它的pipeline将读取到的字节流传递给每个ChannelHandler来处理。

> 3.判断是否需要再来次轮询(大量Channel取消注册)

```java
if (needsToSelectAgain) {
    for (;;) {
        i++;
        if (selectedKeys[i] == null) {
            break;
        }
        selectedKeys[i] = null;
    }

    selectAgain(); // 再次轮询IO事件
    selectedKeys = this.selectedKeys.flip();
    i = -1;
}
```

Netty在轮询IO事件时，会将needsToSelectAgain置为false。那么什么时候needsToSelectAgain会重新被置成true呢？

```java
// NioEventLoop类
void cancel(SelectionKey key) {
    key.cancel();
    cancelledKeys ++;
    if (cancelledKeys >= CLEANUP_INTERVAL) {
        cancelledKeys = 0;
        needsToSelectAgain = true;
    }
}
```

继续查看 `cancel` 方法被调用的地方:

```java
// AbstractChannel类
protected void doDeregister() throws Exception {
    eventLoop().cancel(selectionKey());
}
```

不难看出，在Channel从Selector上取消注册的时候，调用cancel方法将key取消，并且当取消的key到达 `CLEANUP_INTERVAL` 的时候，设置needsToSelectAgain为true,`CLEANUP_INTERVAL`默认值为256。

```java
private static final int CLEANUP_INTERVAL = 256;
```

也就是说，对于每个NioEventLoop而言，每隔256个Channel从Selector上移除的时候，就标记needsToSelectAgain为true，我们还是跳回到上面这段代码:

```java
if (needsToSelectAgain) {
    for (;;) {
        i++;
        if (selectedKeys[i] == null) {
            break;
        }
        selectedKeys[i] = null;
    }

    selectAgain(); // 再次轮询IO事件
    selectedKeys = this.selectedKeys.flip();
    i = -1;
}
```

每满256次，就会进入到if的代码块，首先将selectedKeys的内部数组全部清空，方便被jvm垃圾回收，然后重新调用`selectAgain`重新填装一下 `selectionKey`:

```java
private void selectAgain() {
    needsToSelectAgain = false;
    try {
        selector.selectNow();
    } catch (Throwable t) {
        logger.warn("Failed to update SelectionKeys.", t);
    }
}
```

Netty这么做的目的应该是每隔256次Channel断线，重新清理并填充selectionKeys，保证现存的SelectionKey及时有效。

总结NIO线程处理IO事件的过程：Netty使用数组替换掉JDK原生的HashSet来保证IO事件的高效处理，每个SelectionKey上绑定了Netty类`AbstractChannel`对象作为attachment，在处理每个SelectionKey的时候，就可以找到`AbstractChannel`，然后通过pipeline的方式将处理串行到ChannelHandler，回调到用户方法。

##### 3.处理任务队列(普通任务、定时任务等)

关于task的分类和添加，可以参考闪电侠[netty源码分析之揭开reactor线程的面纱（三）](https://www.jianshu.com/p/58fad8e42379)这篇文章，对于任务的使用场景、分类和添加讲解的很清楚。下面看下NIO线程如何处理这些任务。

```java
runAllTasks(ioTime * (100 - ioRatio) / ioRatio);

protected boolean runAllTasks(long timeoutNanos) {
    fetchFromScheduledTaskQueue(); // 普通任务和定时任务聚合
    Runnable task = pollTask(); // 取出任务
    if (task == null) {
        afterRunningAllTasks(); // 处理tailtask queue的任务(执行收尾任务)
        return false;
    }

    final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos; // 执行任务的截止时间点
    long runTasks = 0; // 执行任务个数
    long lastExecutionTime; // 上次执行时间
    for (;;) {
        safeExecute(task); // 执行任务

        runTasks ++;

        // Check timeout every 64 tasks because nanoTime() is relatively expensive.
        // XXX: Hard-coded value - will make it configurable if it is really a problem.
        if ((runTasks & 0x3F) == 0) { // runTasks=64，0x3F: 111111(二进制)
            lastExecutionTime = ScheduledFutureTask.nanoTime(); // 计算当前时间，更新lastExecutionTime
            if (lastExecutionTime >= deadline) {
                break;
            }
        }

        task = pollTask();
        if (task == null) {
            lastExecutionTime = ScheduledFutureTask.nanoTime();
            break;
        }
    }

    afterRunningAllTasks(); // 执行收尾任务
    this.lastExecutionTime = lastExecutionTime; // 记录上次执行的时间
    return true;
}
```

这个方法表示尽量在一定的时间内，将所有的任务都取出来run一遍。`timeoutNanos` 表示该方法最多执行这么长时间。这么做的原因是如果NIO线程在此停留的时间过长，那么将积攒许多的IO事件无法处理(见NIO线程的前面两个步骤)，最终导致大量客户端请求阻塞，因此默认情况下，Netty将控制内部任务队列的执行时间。

该方法可以拆解成下面几个步骤

- 从scheduledTaskQueue(PriorityQueue)转移定时任务到taskQueue(mpsc queue)；

- 计算本次任务循环的执行截止时间；

- 执行任务；

- 收尾。

> 1.从scheduledTaskQueue转移定时任务到taskQueue(mpsc queue)

首先调用 `fetchFromScheduledTaskQueue()`方法，将到期的定时任务转移到普通任务mpsc queue里面:

```java
// 普通任务和定时任务聚合
private boolean fetchFromScheduledTaskQueue() {
    long nanoTime = AbstractScheduledEventExecutor.nanoTime(); // 当前时间
    Runnable scheduledTask  = pollScheduledTask(nanoTime);
    while (scheduledTask != null) {
        if (!taskQueue.offer(scheduledTask)) { // 任务聚合，定时任务添加到普通任务队列
            // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
            scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask); // 定时任务添加失败，放回定时任务队列
            return false;
        }
        scheduledTask  = pollScheduledTask(nanoTime); // 再次获取可以执行的定时任务
    }
    return true;
}
```

可以看到，NIO线程在把任务从scheduledTaskQueue转移到taskQueue的时候还是非常小心的，当taskQueue无法offer的时候，需要把从scheduledTaskQueue里面取出来的任务重新添加回去。

从scheduledTaskQueue拉取一个定时任务的逻辑如下，传入的参数`nanoTime`为当前时间(其实是当前纳秒减去`ScheduledFutureTask`类被加载的纳秒时间):

```java
protected final Runnable pollScheduledTask(long nanoTime) {
    assert inEventLoop();
		// scheduledTaskQueue优先队列
    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
    if (scheduledTask == null) {
        return null;
    }

    if (scheduledTask.deadlineNanos() <= nanoTime) {
        scheduledTaskQueue.remove(); // 可以执行队列头部的定时任务了
        return scheduledTask;
    }
    return null; // 没有可以执行的定时任务
}
```

可以看到，每次 `pollScheduledTask` 的时候，只有在任务的执行时间已经到了，才会取出来。

> 2.计算本次任务循环的执行截止时间

```java
Runnable task = pollTask(); // 取出任务

final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos; // 执行任务的截止时间点
long runTasks = 0; // 执行任务个数
long lastExecutionTime; // 上次执行时间
```

这一步将取出第一个任务，用NIO线程传入的超时时间 `timeoutNanos` 来计算出当前任务循环的deadline，并且使用了`runTasks`，`lastExecutionTime`来时刻记录任务的状态。

> 3.循环执行任务

```java
for (;;) {
    safeExecute(task); // 执行任务

    runTasks ++;

    // Check timeout every 64 tasks because nanoTime() is relatively expensive.
    // XXX: Hard-coded value - will make it configurable if it is really a problem.
    if ((runTasks & 0x3F) == 0) { // runTasks=64，0x3F: 111111(二进制)
        lastExecutionTime = ScheduledFutureTask.nanoTime(); // 计算当前时间，更新lastExecutionTime
        if (lastExecutionTime >= deadline) {
            break;
        }
    }

    task = pollTask();
    if (task == null) {
        lastExecutionTime = ScheduledFutureTask.nanoTime();
        break;
    }
}
```

首先调用`safeExecute`来确保任务安全执行，忽略任何异常:

```java
protected static void safeExecute(Runnable task) {
    try {
        task.run();
    } catch (Throwable t) {
        logger.warn("A task raised an exception. Task: {}", task, t);
    }
}
```

然后将已运行任务 `runTasks`加一，每隔`0x3F`任务，即每执行完64个任务之后，判断当前时间是否已超过本次任务循环的截止时间，如果超过，那就break掉，如果没有超过，那就继续执行。可以看到，Netty对性能的优化考虑地相当的周到，假设任务队列里面有海量小任务，如果每次都要执行完任务都要判断一下是否到截止时间，那么效率是比较低下的。

> 4.收尾

```java
afterRunningAllTasks(); // 执行收尾任务
this.lastExecutionTime = lastExecutionTime; // 记录上次执行的时间

protected void afterRunningAllTasks() {
    runAllTasksFrom(tailTasks);
}
```

`NioEventLoop`可以通过父类`SingleTheadEventLoop`的`executeAfterEventLoopIteration`方法向`tailTasks`中添加收尾任务。

```java
public final void executeAfterEventLoopIteration(Runnable task) {
    ObjectUtil.checkNotNull(task, "task");
    if (isShutdown()) {
        reject();
    }

    if (!tailTasks.offer(task)) {
        reject(task);
    }

    if (wakesUpForTask(task)) {
        wakeup(inEventLoop());
    }
}
```

`this.lastExecutionTime = lastExecutionTime;`简单记录一下任务执行的时间。

总结处理任务队列的过程：

- Netty内部的任务分为普通任务和定时任务，分别落地到MpscQueue和PriorityQueue；
- Netty每次执行任务循环之前，会将已经到期的定时任务从PriorityQueue转移到MpscQueue；
- Netty每执行64个任务检查一下是否该退出任务循环。

### 四、面试点

- Netty服务端默认起多少线程？何时启动？ 

  线程数：2倍的CPU核数，第一次执行任务时启动。

- Netty如何解决空轮训Bug？ 

  判断一定的时间内(timeoutMillis)，空轮训次数如果超过512次，则重建Selector。

- Netty如何实现异步串行无锁化？ 

  执行任务时，根据inEventLoop()方法判断是NIO线程还是外部线程，对于外部线程，将执行逻辑封装成Runnable任务，放到MpscQueue，由NIO线程执行。

### 参考文章

- [netty源码分析之揭开reactor线程的面纱（一）](https://www.jianshu.com/p/0d0eece6d467)
- [netty源码分析之揭开reactor线程的面纱（二）](https://www.jianshu.com/p/467a9b41833e)
- [netty源码分析之揭开reactor线程的面纱（三）](https://www.jianshu.com/p/58fad8e42379)