---
layout:     post
title:      AQS源码分析3——Condition接口实现之ConditionObject分析
subtitle:   Condition/AQS.ConditionObject
date:       2019-01-20
author:     Jay
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# AQS源码分析3——Condition接口实现之ConditionObject分析

[Lock与Condition接口功能分析](https://xuanjian1992.top/2019/01/17/Lock%E4%B8%8ECondition%E6%8E%A5%E5%8F%A3%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90/)这边文章分析了`Lock`与`Condition`接口的方法和功能，下面分析`AbstractQueuedSynchronizer`对于`Condition`接口的实现细节。

## 一、Object类与Condition接口方法的对比

[Lock与Condition接口功能分析](https://xuanjian1992.top/2019/01/17/Lock%E4%B8%8ECondition%E6%8E%A5%E5%8F%A3%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90/)已经阐述，`Condition`接口的`await/signal`机制是设计用来代替监视器锁的`wait/notify`机制的，因此与监视器锁的`wait/notify`机制对比分析，有助于更好地理解`Conditon`接口：

| Object 类方法                      | Condition 接口方法                      | 区别                       |
| ---------------------------------- | --------------------------------------- | -------------------------- |
| void wait()                        | void await()                            |                            |
| void wait(long timeout)            | long awaitNanos(long nanosTimeout)      | 时间单位，返回值           |
| void wait(long timeout, int nanos) | boolean await(long time, TimeUnit unit) | 时间单位，参数类型，返回值 |
| void notify()                      | void signal()                           |                            |
| void notifyAll()                   | void signalAll()                        |                            |
|                                    | void awaitUninterruptibly()             | Condition独有              |
|                                    | boolean awaitUntil(Date deadline)       | Condition独有              |

**这里先做说明**，本文指出`wait`方法时，是泛指`wait()`、`wait(long timeout)`、`wait(long timeout, int nanos)`三个方法，当需要指明某个特定的方法时，会带上相应的参数。同样指出`notify`方法时，也是泛指`notify(`)，`notifyAll()`方法，`await`方法和`signal`方法以此类推。

首先通过`wait/notify`机制来类比`await/signal`机制：

1. 调用`wait`方法的线程首先必须是已经进入了同步代码块，即已经获取了监视器锁；与之类似，调用`await`方法的线程首先必须获得`lock`锁
2. 调用`wait`方法的线程会释放已经获得的监视器锁，进入当前监视器锁的等待队列(`wait set`)中；与之类似，调用`await`方法的线程会释放已经获得的`lock`锁，进入到当前`Condtion`对应的条件队列中。
3. 调用监视器锁的`notify`方法会唤醒等待在该监视器锁上的线程，这些线程将开始参与锁竞争，并在获得锁后，从`wait`方法处恢复执行；与之类似，调用`Condtion`的`signal`方法会唤醒**对应的**条件队列中的线程，这些线程将开始参与锁竞争，并在获得锁后，从`await`方法处开始恢复执行。

## 二、Conditon应用实例

以下是使用`Condition`的实例，理解了`Condition`的作用之后，便于下面的`ConditionObject`源码分析与理解。

```java
// 有界缓冲
class BoundedBuffer {
    final Lock lock = new ReentrantLock(); // 互斥锁
    final Condition notFull = lock.newCondition(); // 条件“非满”
    final Condition notEmpty = lock.newCondition(); // 条件“非空”

    final Object[] items = new Object[100]; // 存放数据的有限大小数组
    int putptr, takeptr, count; // 放入位置，取出位置，总数

    // 生产者方法，往数组里面写数据
    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await(); //数组已满，没有空间时，挂起等待，直到数组“非满”（notFull）
            items[putptr] = x; 
            if (++putptr == items.length) putptr = 0; // 循环数组
            ++count;
            // 因为放入了一个数据，数组肯定不为空
            // 此时唤醒等待在notEmpty条件上的线程
            notEmpty.signal(); 
        } finally {
            lock.unlock();
        }
        // 返回之后才释放锁
    }

    // 消费者方法，从数组里面拿数据
    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await(); // 数组是空的，没有数据可拿时，挂起等待，直到数组非空（notEmpty）
            Object x = items[takeptr];
            if (++takeptr == items.length) takeptr = 0;
            --count;
            // 因为拿出了一个数据，数组肯定不满
            // 此时唤醒等待在notFull条件上的线程
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
        // 返回之后才释放锁
    }
}
```

这是`java`官方文档提供的[例子](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/Condition.html)，是一个典型的生产者-消费者模型。这里在同一个互斥锁上，创建了两个条件队列`notFull`， `notEmpty`。当数组已满、没有存储空间时，`put`方法在`notFull`条件上等待，直到数组“not full”;当数组空了、没有数据可读时，`take`方法在`notEmpty`条件上等待，直到数组“not empty”，而`notEmpty.signal()`和`notFull.signal()`则用来唤醒等待在这个条件上的线程。

注意上面所说的，在`notFull` 、`notEmpty`条件上等待事实上是指线程在条件队列（`condition queue`）上等待，当该线程被相应的`signal`方法唤醒后，将进入到前面两篇AQS源码分析中介绍的`sync queue`(或称为等待队列)中去争锁，争到锁后才能能`await`方法处返回。这里存在两种队列——`condition queue`和`sync queue`，它们都定义在AQS中。

下面先介绍`condition queue`和`sync queue`(等待队列)，理清楚两者的关系。

## 三、同步队列 vs 条件队列

### 1.sync queue

首先，在[AQS源码分析1——独占锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901-%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)这篇文章中提到，所有等待锁的线程都会被包装成`Node`扔到一个同步队列(也称为等待队列，下同)中。该同步队列如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/condition-1.png)

`sync queue`是一个双向链表，使用`prev`、`next`属性来串联节点。但是在这个同步队列中，一直没有用到`nextWaiter`属性，即使是在[共享锁](https://segmentfault.com/a/1190000016447307)模式下，这一属性也只作为一个标记并指向了一个空节点(`Node SHARED = new Node()`)，因此，在`sync queue`中不会用它来串联节点。

### 2.condtion queue

每创建一个`Condtion`对象就会对应一个`Condtion`队列，每一个调用了`Condtion`对象的`await`方法的线程都会被包装成`Node`并扔进一个条件队列中，就像这样：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/condition-2.png)

可见，每一个`Condition`对象对应一个`Conditon`队列，每个`Condtion`队列都是独立的、互相不影响的。在上图中，如果对当前线程调用了`notFull.await()`，则当前线程就会被包装成`Node`加到`notFull`队列的末尾。

值得注意的是，`condition queue`是一个单向链表，在该链表中使用`nextWaiter`属性来串联链表。但是，就像在`sync queue`中不会使用`nextWaiter`属性来串联链表一样，在`condition queue`中也并不会用到`prev`, `next`属性，它们的值都为null。也就是说，在条件队列中，`Node`节点真正用到的属性只有三个：

- `thread`：代表当前正在等待某个条件的线程；
- `waitStatus`：线程的等待状态；
- `nextWaiter`：指向条件队列中的下一个节点。

既然这里又提到了`waitStatus`，这里再回顾一下它的取值范围：

```java
volatile int waitStatus;
static final int CANCELLED =  1;
static final int SIGNAL    = -1;
static final int CONDITION = -2;
static final int PROPAGATE = -3;
```

在条件队列中只需要关注一个值即可——`CONDITION`，它表示线程处于正常的等待状态；只要`waitStatus`不是`CONDITION`，就认为线程不再等待了，此时就要从条件队列中出队。

### 3.sync queue 和 conditon queue的联系

一般情况下，等待锁的`sync queue`和条件队列`condition queue`是相互独立的，彼此之间并没有任何关系。但是当调用某个条件队列的`signal`方法时，会将某个或所有等待在这个条件队列中的线程转移到同步队列中，可能会唤醒或继续阻塞。在同步队列中继续争锁，抢不到锁继续阻塞，抢到锁则退出。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/condition-3.png)

但是，`node`是被**一个一个**转移过去的，即使调用的是`signalAll()`方法，也是**一个一个**转移过去的，而不是将整个条件队列直接接在`sync queue`的末尾。

同时要注意，在`sync queue`中只使用`prev`、`next`来串联链表，而不使用`nextWaiter`；在`condition queue`中只使用`nextWaiter`来串联链表，而不使用`prev`、`next`。事实上，它们就是使用了同样的Node数据结构的完全独立的两种链表。因此，将节点从`condition queue`中转移到`sync queue`中时，需要断开原来的链接（`nextWaiter`），建立新的链接（`prev`， `next`），这某种程度上也是需要将节点**一个一个**地转移过去的原因之一。

### 4.入队时和出队时的锁状态

`sync queue`是等待锁的队列，当一个线程被包装成`Node`加到该队列中时，必然是没有获取到锁；当处于该队列中的节点获取到了锁，它将从该队列中移除(事实上移除操作是将获取到锁的节点设为新的`dummy head`，并将`thread`属性置为`null`)。

`condition`队列是等待在特定条件下的队列，因为调用`await`方法时，必然是已经获得了`lock`锁，所以在进入`condtion`队列**前**线程必然是已经获取锁；在被包装成Node扔进条件队列后，线程将释放锁，然后挂起；当处于该队列中的线程被`signal`方法唤醒后，由于队列中的节点在之前挂起的时候已经释放了锁，所以必须先去再次竞争锁，因此该节点会被添加到`sync queue`中。因此，条件队列在出队时，线程并不持有锁。

所以这两个队列的锁状态正好相反：

- `condition queue`：入队时已经持有了锁 -> 在队列中释放锁 -> 离开队列时没有锁 -> 转移到`sync queue`；
- `sync queue`：入队时没有锁 -> 在队列中争锁 -> 离开队列时获得了锁。

通过上面的介绍，应该对条件队列有了清楚的认识，接下来就进入到本篇的重头戏——`ConditionObject`源码分析。

## 四、ConditionObject

AQS对`Condition`接口的实现主要是`ConditionObject`，它的核心实现是一个条件队列，每一个在某个`condition`上等待的线程都会被封装成`Node`对象扔进这个条件队列。

### 1.核心属性

它的核心属性只有两个：

```java
/** First node of condition queue. */
private transient Node firstWaiter;
/** Last node of condition queue. */
private transient Node lastWaiter;
```

这两个属性分别代表了条件队列的队头和队尾，每当新建一个`ConditionObject`对象时，都会对应一个条件队列。

### 2.构造函数

```java
public ConditionObject() { }
```

构造函数体为空，可见条件队列是延时初始化的，在真正用到的时候才会初始化。

## 五、Condition接口方法实现分析

### 1.await()第一部分分析

```java
// 可中断条件等待
public final void await() throws InterruptedException {
    // 如果当前线程在调用await()方法前已经被中断，则直接抛出InterruptedException
    if (Thread.interrupted())
        throw new InterruptedException();
    // 将当前线程封装成Node添加到条件队列，删除cancel的节点
    Node node = addConditionWaiter();
    // 完全释放当前线程所占用的锁，保存当前的锁状态
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    // 如果当前节点不在同步队列中(在条件队列中)，可能是刚刚await, 还没有人调用signal方法，则直接将
    // 当前线程挂起；也可能是在阻塞等待的过程中，线程被假唤醒，由于还在条件队列中，因此该线程继续挂起。
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); // 线程将在这里被挂起，停止运行
        // 能执行到这里，说明要么是signal方法被调用了，要么是线程被中断了(也可能是线程假醒)
        // 所以检查下线程被唤醒的原因，如果是因为中断被唤醒，则跳出while循环
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    // 第一部分就分析到这里，下面的部分到第二部分再看, 先把它注释起来
    /*
    // 同步队列中获取互斥锁
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    */
}
```

上面的代码注释中描述了大体的流程，接下来详细看看`await`方法中所调用方法的具体实现。

首先是将当前线程封装成`Node`扔进条件队列中的`addConditionWaiter`方法：

#### addConditionWaiter

```java
/* 添加一个等待节点到条件队列，返回新增的节点 */
private Node addConditionWaiter() {
    Node t = lastWaiter;
    // 如果尾节点已经cancel，则先遍历整个链表，清除所有cancel的节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        unlinkCancelledWaiters(); // 删除条件队列中所有cancel的节点
        t = lastWaiter;
    }
    // 将当前线程包装成Node扔进条件队列
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    /*
    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
    */
    // node插入到队尾
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

首先要思考的是，是否存在两个不同的线程同时入队的情况？不存在。因为前面说过，能调用`await`方法的线程必然是已经获得锁，而获得锁的线程只有一个，所以这里不存在并发，不需要`CAS`操作。

这个方法简单的将当前线程封装成`Node`加到条件队列的末尾，这和将一个线程封装成`Node`加入同步(等待)队列略有不同：

1. 节点加入`sync queue`时`waitStatus`的值为0，但节点加入`condition queue`时`waitStatus`的值为`Node.CONDITION`。
2. `sync queue`的头节点为`dummy`节点，如果队列为空，则会先创建一个`dummy`节点，再创建一个代表当前线程的Node添加在`dummy`节点的后面；而`condtion queue` 没有`dummy`节点，初始化时直接将`firstWaiter`和`lastWaiter`直接指向新建的节点。
3. `sync queue`是一个双向链表的队列，在节点入队后，要同时修改**当前节点的前驱**和**前驱节点的后继**；而在`condtion queue`中，只修改了前驱节点的`nextWaiter`，即`condtion queue`是作为**单向链表的队列**来使用的。

如果节点入队时发现尾节点已经取消等待，那么就不应该接在它的后面，此时需要调用`unlinkCancelledWaiters`来剔除那些已经取消等待的线程节点：

```java
/* 条件队列中去除已经cancel的节点 */
private void unlinkCancelledWaiters() {
    // 当前节点
    Node t = firstWaiter;
    // 前一个非cancel(正常等待)的节点
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        // 如果t已经cancel
        if (t.waitStatus != Node.CONDITION) {
            // 节点t断开与next的连接
            t.nextWaiter = null;
            // 还没有找到第一个正常等待的节点，调整firstWaiter为next
            if (trail == null)
                firstWaiter = next;
            else
                // 否则先将前一个正常等待节点的下一个节点指向next
                trail.nextWaiter = next;
            // 如果没有后续节点，则将lastWaiter置为前一个正常等待的节点trail
            if (next == null)
                lastWaiter = trail;
        }
        else
            // t是正常等待的节点，更新trail，将trail置为t
            trail = t;
        // t置为next，检查下一个节点
        t = next;
    }
}
```

`unlinkCancelledWaiters`方法将从头节点开始遍历整个队列，剔除其中`waitStatus`不为`Node.CONDTION`的节点，其中链表操作逻辑如注释所述。

#### fullyRelease

在节点被成功添加到条件队列的末尾后，将调用`fullyRelease`方法来释放当前线程所占用的锁。

```java
/* 完全释放锁状态 */
final int fullyRelease(Node node) {
    boolean failed = true;
    try {
        // 获取state
        int savedState = getState();
        // 释放锁
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else {
            // 如果失败，抛出异常
            throw new IllegalMonitorStateException();
        }
    } finally {
        // 如果失败，则将节点ws置为cancel
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

当调用这个方法时，说明当前线程已经被封装成`Node`扔进条件队列。在该方法中，通过`release`方法释放锁。

需要注意，这是一次性释放了所有的锁，即对于可重入锁而言，无论重入了几次，这里是一次性释放完的，这也就是该方法的名字叫`fullyRelease`的原因。但这里尤其要注意`release(savedState)`方法是有可能抛出`IllegalMonitorStateException`的，这是因为当前线程可能并不是持有锁的线程。但是前面不是说，只有持有锁的线程才能调用`await`方法吗？既然`fullyRelease`方法在`await`方法中，为什么当前线程还有可能并不是持有锁的线程？

虽然逻辑是这样，但是在调用`await`方法时，其实并没有检测`Thread.currentThread() == getExclusiveOwnerThread()`。也就是说，执行到`fullyRelease`这一步才会检测这一点，而这一点检测是由AQS子类实现的`tryRelease`方法来保证的，例如`ReentrantLock`对`tryRelease`方法的实现如下：

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

当发现当前线程不是持有锁的线程时，就会进入`finally`块，将当前`Node`的状态设为`Node.CANCELLED`，这也就是上面`addConditionWaiter`在添加新节点前每次都会检查尾节点是否已经cancel的原因。

在当前线程的锁完全释放之后，就可以调用`LockSupport.park(this)`把当前线程挂起，等待被`signal`。但是在挂起当前线程之前先用`isOnSyncQueue`确保它**不在**`sync queue`中，这是为什么呢？这里的判断条件是为了防止线程休眠时的“**假醒**”现象，下面详述。

```java
// 判断节点node是否在同步队列中
final boolean isOnSyncQueue(Node node) {
    // 节点ws为CONDITION，表明在条件队列
    // node.prev为空，也是在条件队列
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    // node.next不为空，表明在同步队列。因为同步队列利用prev/next串联双向连接的节点
    if (node.next != null) // If has successor, it must be on queue
        return true;
    /*
     * node.prev can be non-null, but not yet on queue because
     * the CAS to place it on queue can fail. So we have to
     * traverse from tail to make sure it actually made it.  It
     * will always be near the tail in calls to this method, and
     * unless the CAS failed (which is unlikely), it will be
     * there, so we hardly ever traverse much.
     */
    // 节点入同步队列过程分为三步(见enq方法)
    // CAS这一步可能失败，因此需要从尾部开始遍历
    // 检查是否已经把当前的node挂到同步队列了
    return findNodeFromTail(node);
}

// 从同步队列尾部开始找节点node
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

为了解释这一问题，先来看看`signal`方法。

### 2.signalAll()

在分析`signalAll`之前，首先要区分调用`signalAll`方法的线程与`signalAll`方法要唤醒的线程(等待在对应的条件队列里的线程)：

- 调用`signalAll`方法的线程本身已经持有锁，现在准备释放锁；
- 在条件队列里的线程已经在对应的条件上挂起，等待被`signal`唤醒，然后去争锁。

首先，与调用`notify`时线程必须已经持有监视器锁类似，在调用`condition`的`signal`方法时，线程也必须已经持有`lock`锁：

```java
// signal对应条件队列上的所有节点
public final void signalAll() {
    if (!isHeldExclusively())
    	throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
    	doSignalAll(first);
}
```

该方法首先检查当前调用`signal`方法的线程是不是持有锁的线程，这是通过`isHeldExclusively`方法来实现的，该方法由继承AQS的子类来实现，例如，`ReentrantLock`对该方法的实现为：

```java
protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}
```

因为`exclusiveOwnerThread`保存了当前持有锁的线程，这里只要检测它是不是等于当前线程就可以。
接下来先通过`firstWaiter`是否为空判断条件队列是否为空，如果条件队列不为空，则调用`doSignalAll`方法：

```java
// Removes and transfers all nodes.	
// first: the first node on condition queue
private void doSignalAll(Node first) {
    lastWaiter = firstWaiter = null;
    do {
        Node next = first.nextWaiter;
        // first断开与条件队列的连接
        first.nextWaiter = null;
        // 转移
        transferForSignal(first);
        first = next;
    } while (first != null);
}
```

首先通过`lastWaiter = firstWaiter = null;`将整个条件队列清空，然后通过一个`do-while`循环，将原先的条件队列里面的节点**一个一个**拿出来(令nextWaiter = null)，再通过`transferForSignal`方法**一个一个**添加到`sync queue`的末尾：

```java
// 将node节点从条件队列转移到同步队列
final boolean transferForSignal(Node node) {
    /*
     * If cannot change waitStatus, the node has been cancelled.
     */
    // 节点在signal之前被取消了，则转移下一个节点
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    /*
     * Splice onto queue and try to set waitStatus of predecessor to
     * indicate that thread is (probably) waiting. If cancelled or
     * attempt to set waitStatus fails, wake up to resync (in which
     * case the waitStatus can be transiently and harmlessly wrong).
     */
    // 如果该节点在条件队列中正常等待，则利用enq方法将该节点添加至同步队列的尾部
    Node p = enq(node);
    int ws = p.waitStatus;
    // 前驱节点已经cancel或者CAS设置前驱节点的状态为signal失败，则唤醒node线程
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL)) 
        LockSupport.unpark(node.thread); 
    return true;
}
```

在`transferForSignal`方法中，先使用`CAS`操作将当前节点的`waitStatus`状态由`CONDTION`设为0，如果修改不成功，则说明该节点已经被`CANCEL`了，则直接返回，操作下一个节点；如果修改成功，则说明已经将该节点从等待的条件队列中成功“唤醒”了，但此时该节点对应的线程并没有真正被唤醒，它还要和其他普通线程一样去争锁，因此它将被添加到`sync queue`的末尾等待获取锁。

这里通过`enq`方法将该节点添加进`sync queue`的末尾。不过这里要注意，`enq`方法将`node`节点添加进同步队列时，返回的是`node`的**前驱节点**。

在将节点成功添加进`sync queue`中后，得到该节点在`sync queue`中的前驱节点。前面说过，在`sync queque`中的节点都要靠前驱节点去唤醒，所以这里要做的就是将前驱节点的`waitStatus`设为`Node.SIGNAL`, 这一点和`shouldParkAfterFailedAcquire`所做的工作类似：

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus;
    if (ws == Node.SIGNAL)
        return true;
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

所不同的是，`shouldParkAfterFailedAcquire`将会向前查找，跳过那些被`cancel`的节点，或者将前驱节点的`waitStatus`设成`SIGNAL`，然后继续尝试获取锁，最后再挂起。而在`transferForSignal`中，当前`Node`所代表的线程本身就已经被挂起，所以这里做的更像是一个复合操作——**只要前驱节点处于被取消的状态或者无法将前驱节点的状态修成Node.SIGNAL，那就将Node所代表的线程唤醒**，但这个条件并不意味着当前lock处于可获取的状态，有可能线程被唤醒但锁还是处于被占有的状态，不过这样做至少是无害的，因为在线程被唤醒后还要去争锁，如果抢不到锁，则大不了再次被挂起。

值得注意的是，`transferForSignal`是有返回值的，但是在这个方法中并没有用到，它将在`signal()`方法中被使用。

在继续往下看`signal()`方法之前，这里再总结一下`signalAll()`方法：

1. 将条件队列清空(只是令`lastWaiter = firstWaiter = null`，队列中的节点和连接关系仍然还存在)；
2. 将条件队列中的头节点取出，使之成为孤立节点(`nextWaiter`，`prev`，`next`属性都为`null`)；
3. 如果该节点处于被`Cancelled`的状态，则直接跳过该节点(由于是孤立节点，则会被GC回收);
4. 如果该节点处于正常状态，则通过`enq`方法将它添加到`sync queue`的末尾；
5. 判断是否需要将该节点唤醒(包括设置该节点的前驱节点的状态为`SIGNAL`)，如有必要，直接唤醒该节点；
6. 重复2-5，直到整个条件队列中的节点都被处理完。

### 3.signal()

与`signalAll()`方法不同，`signal()`方法只会唤醒一个节点，对于AQS的实现来说，就是唤醒条件队列中第一个没有被`Cancel`的节点。弄懂了`signalAll()`方法，`signal()`方法就很容易理解，因为它们大同小异：

```java
// 唤醒一个节点
public final void signal() {
    // 唤醒之前检查互斥锁是否被当前线程占用
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    Node first = firstWaiter;
    if (first != null)
        // 从第一个节点开始唤醒
        doSignal(first);
}
```

首先依然是检查调用该方法的线程(即当前线程)是不是已经持有了锁，这一点和上面的`signalAll()`方法一样，所不一样的是，接下来调用的是`doSignal`方法：

```java
// 唤醒单个节点
private void doSignal(Node first) {
    do {
        // 将firstWaiter指向条件队列队头的下一个节点
        if ( (firstWaiter = first.nextWaiter) == null)
            lastWaiter = null;
        // 将条件队列原来的队头从条件队列中断开，则此时该节点成为一个孤立的节点
        first.nextWaiter = null;
    } while (!transferForSignal(first) &&
             (first = firstWaiter) != null);
}
```

这个方法也是一个`do-while`循环，目的是遍历整个条件队列，找到第一个没有被`cancelled`的节点，并将它添加到同步队列的末尾。如果条件队列里面已经没有节点了，则将条件队列清空（`firstWaiter=lasterWaiter=null`）。

这里用的依然是`transferForSignal`方法，但是用到了它的返回值，只要节点被成功添加到`sync queue`中，`transferForSignal`就返回true，此时while循环的条件就不满足了，整个方法就结束了，即调用`signal()`方法，只会唤醒一个线程。

总结： 调用`signal()`方法会从当前条件队列中取出第一个没有被`cancel`的节点添加到同步队列的末尾。

### 4.await()第二部分分析

前面已经分析了`signal`方法，它会将节点添加进`sync queue`队列中，并要么立即唤醒线程，要么等待前驱节点释放锁后将自己唤醒，无论怎样，被唤醒的线程要从被挂起的地方继续运行(在调用`await()`方法的地方，以`await()`方法为例)：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); // 线程在这里被挂起，被唤醒后，将从这里继续往下运行
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

这里值得注意的是，当线程被唤醒时，其实并不知道是因为什么原因被唤醒，有可能是因为其他线程调用`signal`方法，也有可能是因为当前线程被中断。

但是，无论是因为中断被唤醒还是被`signal`唤醒，被唤醒的线程最后都将离开`condition queue`，进入到`sync queue`中，这一点在下面分析源码的时候详述。

随后线程将在`sync queue`中利用`acquireQueued`方法进行“**阻塞式**”争锁，抢到锁就返回，抢不到锁就继续被挂起。因此当`await()`方法返回时，必然是保证了当前线程已经持有了`lock`锁。

另外有一点需要提前说明，这一点对于下面理解源码很重要，那就是：**如果从线程被唤醒，到线程获取到锁这个过程中发生过中断，该怎么处理？**

一般，中断对于当前线程只是个建议，由当前线程决定怎么对其做出处理。在`acquireQueued`方法中，对中断是不响应的，只是简单的记录抢锁过程中的中断状态，并在抢到锁后将这个中断状态返回，交于上层调用的函数处理。而这里“上层调用的函数”就是`await()`方法。

那么`await()`方法是怎么对待这个中断的？这取决于：

> 中断发生时，线程是否已经被`signal`过。

如果中断发生时，当前线程并没有被`signal`过，则说明当前线程还处于条件队列中，属于正常在等待中的状态，此时中断将导致当前线程的正常等待行为被打断，进入到`sync queue`中抢锁，因此，在线程从`await`方法返回后，需要抛出`InterruptedException`，表示当前线程因为中断而被唤醒。

如果中断发生时，当前线程已经被`signal`过了，则说明**这个中断来的太晚了**，既然当前线程已经被`signal`过了，那么就说明在中断发生前，它就已经正常地从`condition queue`中被唤醒了(转移到同步队列，可能唤醒也可能阻塞)，所以随后即使发生了中断(注意，这个中断可以发生在抢锁之前，也可以发生在抢锁的过程中)，线程都将忽略它，仅仅是在`await()`方法返回后，再自我中断一下，补一下这个中断。就好像这个中断是在`await()`方法调用结束之后才发生的一样。这里之所以要“补一下”这个中断，是因为线程在用`Thread.interrupted()`方法检测是否发生中断的同时，会将中断状态清除，因此如果选择了忽略中断，则应该在`await()`方法退出后将它设成原来的样子。

理清了上面的概念，再来看看`await()`方法是如何做的。它用中断模式`interruptMode`这个变量记录中断事件，该变量有三个值：

1. `0` ： 代表整个过程中一直没有中断发生；
2. `THROW_IE` ： 表示退出`await()`方法时需要抛出`InterruptedException`，这种模式对应于**中断发生在signal之前**；
3. `REINTERRUPT` ： 表示退出`await()`方法时只需要再自我中断以下，这种模式对应于**中断发生在signal之后**，即中断来的太晚了。

```java
/** Mode meaning to reinterrupt on exit from wait */
private static final int REINTERRUPT =  1;
/** Mode meaning to throw InterruptedException on exit from wait */
private static final int THROW_IE    = -1;
```

接下来就从线程被唤醒的地方继续往下走，一步步分析源码：

#### 情况1：中断发生时，线程还没有被signal过

线程被唤醒后，将首先使用`checkInterruptWhileWaiting`方法检测中断的模式:

```java
// 检查中断模式
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ? // 当前线程是否被中断?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

这里假设已经发生过中断，则`Thread.interrupted()`方法必然返回`true`，接下来就是用`transferAfterCancelledWait`进一步判断是否发生了`signal`：

```java
// 判断中断在signal之前还是之后
final boolean transferAfterCancelledWait(Node node) {
    // 主动取消node的等待，转移到同步队列。CAS成功，说明中断发生在signal()之前。
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
        enq(node);
        // (中断发生在signal()之前)
        return true;
    }
    // (中断发生在signal()之后，该节点可能正在挂到同步队列中(enq))
    /*
     * If we lost out(遇到) to a signal(), then we can't proceed
     * until it finishes its enq().  Cancelling during an
     * incomplete transfer is both rare and transient, so just
     * spin.
     */
    // 节点若还没挂到同步队列，自旋等待
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

上面已经说过，判断一个`node`是否被`signal`过，一个简单有效的方法就是判断它是否离开了`condition queue`，进入到`sync queue`中。换句话说，**只要一个节点的waitStatus还是Node.CONDITION，那就说明它还没有被signal过。**
由于现在分析情况1，则当前节点的`waitStatus`必然是`Node.CONDITION`，则会成功执行`compareAndSetWaitStatus(node, Node.CONDITION, 0)`，将该节点的状态设置成0，然后调用`enq(node)`方法将当前节点添加进`sync queue`中，然后返回`true`。这里值得注意的是，**此时并没有断开node的nextWaiter**，所以最后一定不要忘记将这个链接断开。

再回到`transferAfterCancelledWait`调用处，可知由于`transferAfterCancelledWait`将返回true，现在`checkInterruptWhileWaiting`将返回`THROW_IE`，这表示线程在离开`await`方法时应当要抛出`THROW_IE`异常。

再回到`checkInterruptWhileWaiting`的调用处：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this); 
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) // 现在在这里
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) 
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

`interruptMode`现在为`THROW_IE`，则执行`break`，跳出`while`循环。

接下来将执行`acquireQueued(node, savedState)`进行争锁，注意，这里传入的需要获取锁的重入数量是`savedState`，即之前释放了多少，这里就需要再次获取多少：

```java
// 排队获取锁
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) // 如果线程获取不到锁，则将在这里被阻塞住
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

`acquireQueued`已经在[AQS源码分析1——独占锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901-%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)详细分析过，它是一个**阻塞式**的方法，获取到锁则退出，获取不到锁则会被挂起。该方法只有在最终获取到了锁后才会退出，并且退出时会返回当前线程的中断状态，如果线程在获取锁的过程中又被中断，则会返回true，否则会返回false。但是其实这里返回true还是false已经不重要了，因为前面已经发生过中断了，线程就是因为中断而被唤醒的，所以线程在退出`await()`方法时，必然会抛出`InterruptedException`。

这里假设它获取到了锁，则它将回到上面的调用处，由于这时的`interruptMode = THROW_IE`，则会跳过if语句。接下来将执行

```java
if (node.nextWaiter != null) 
    unlinkCancelledWaiters();
```

上面说过，当前节点的`nextWaiter`是有值的，它并没有和原来的`condition`队列断开，这里线程已经获取到了锁，根据[AQS源码分析1——独占锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901-%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)中的分析，通过`setHead`方法已经将它的`thread`属性置为null，从而将当前线程从`sync queue`"移除"了，接下来应当将它从`condition`队列里面移除。由于`condition`队列是一个单向队列，无法获取到它的前驱节点，所以只能从头开始遍历整个条件队列，然后找到这个节点，再移除它。

然而事实上并没有这么做。因为既然已经必须从头开始遍历链表了，就干脆一次性把链表中所有没有在等待的节点都拿出去，所以这里调用了`unlinkCancelledWaiters`方法，该方法在前面`await()`第一部分分析的时候已经讲过，它就是简单的遍历链表，找到所有`waitStatus`不为`CONDITION的`节点，并把它们从队列中移除。

节点被移除后，接下来就是最后一步——汇报中断状态：

```java
if (interruptMode != 0)
    reportInterruptAfterWait(interruptMode);
```

这里`interruptMode=THROW_IE`，说明发生了中断，则将调用`reportInterruptAfterWait`：

```java
// 根据中断模式，报告中断状态
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    // signal之前被中断
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    // signal之后被中断
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
```

可以看出，在`interruptMode=THROW_IE`时，就是简单的抛出了一个`InterruptedException`。

至此，情况1（中断发生于`signal`之前）就分析完了，这里简单总结一下：

1. 线程因为中断，从挂起的地方被唤醒；
2. 随后通过`transferAfterCancelledWait`确认了线程的`waitStatus`值为`Node.CONDITION`，说明并没有`signal`发生过；
3. 然后修改线程的`waitStatus`为0，并通过`enq(node)`方法将其添加到`sync queue`中；
4. 接下来线程将在`sync queue`中以阻塞的方式获取，如果获取不到锁，将会被再次挂起；
5. 线程在`sync queue`中获取到锁后，将调用`unlinkCancelledWaiters`方法将自己从条件队列中移除，该方法还会顺便移除其他取消等待的锁；
6. 最后通过`reportInterruptAfterWait`抛出了`InterruptedException`。

由此可以看出，一个调用了`await`方法挂起的线程在被中断后不会立即抛出`InterruptedException`，而是会被添加到`sync queue`中去争锁，如果争不到，还是会被挂起；只有争到了锁之后，该线程才得以从`sync queue`和`condition queue`中移除，最后抛出`InterruptedException`。

所以，**一个调用了await方法的线程，即使被中断了，它依旧还是会被阻塞住，直到它获取到锁之后才能返回，并在返回时抛出InterruptedException。中断对它意义更多的是体现在将它从condition queue中移除，加入到sync queue中去争锁，从这个层面上看，中断和signal的效果其实很像，所不同的是，在await()方法返回后，如果是因为中断被唤醒，则await()方法需要抛出InterruptedException异常，表示是它是被非正常唤醒的（正常唤醒是指被signal唤醒）。**

#### 情况2：中断发生时，线程已经被signal过

这种情况对应于“中断来的太晚了”，即`REINTERRUPT`模式，线程在拿到锁退出`await()`方法后，只需要再自我中断一下，不需要抛出`InterruptedException`。

值得注意的是，这种情况其实包含两个子情况：

1. 被唤醒时，已经发生了中断，但此时线程已经被signal过；
2. 被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断。

下面分别来分析：

##### 情况2.1：被唤醒时，已经发生了中断，但此时线程已经被signal过

对于这种情况，与前面中断发生于`signal`之前的主要差别在于`transferAfterCancelledWait`方法：

```java
// 判断中断在signal之前还是之后
final boolean transferAfterCancelledWait(Node node) {
    // 线程A执行到这里，CAS操作将会失败
    if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) { 
        enq(node);
        return true;
    }
    // 由于中断发生前，线程已经被signal，则这里只需要等待线程成功进入sync queue即可
    while (!isOnSyncQueue(node))
        Thread.yield();
    return false;
}
```

在这里，由于`signal`已经发生过，则由之前分析的`signal`方法可知，此时当前节点的`waitStatus`必定不为`Node.CONDITION`，线程将跳过if语句。此时当前线程可能已经在`sync queue`中，**或者正在进入到sync queue**。

为什么这里会出现“正在进入到`sync queue`”的情况？ 下面解释下：

假设当前线程为线程A，它被唤醒之后检测到发生了中断，来到`transferAfterCancelledWait`这里，而另一个线程B在这之前已经调用了`signal`方法，该方法会调用`transferForSignal`将当前线程A添加到`sync queue`的末尾：

```java
// 唤醒、转移节点
final boolean transferForSignal(Node node) {
    // 线程B执行到这里，CAS操作将会成功
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;
        
    // 入队列
    Node p = enq(node);
    int ws = p.waitStatus;
    // 前驱节点已经cancel或者CAS设置前驱节点的状态为signal失败，则唤醒node线程
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

因为线程A和线程B是并发执行的，而这里分析的是“中断发生在`signal`之后”，则此时线程B的`compareAndSetWaitStatus`先于线程A执行。这时可能出现线程B已经成功修改了node的`waitStatus`状态，但是还没来得及调用`enq(node)`方法，线程A就执行到了`transferAfterCancelledWait`方法，此时它发现`waitStatus`已经不是`Condition`，但是其实当前节点还没有被添加到`sync node`队列中，因此，它接下来将通过自旋，等待线程B执行完`transferForSignal`方法。

线程A在自旋过程中会不断地判断节点有没有被成功添加进`sync queue`，判断的方法就是`isOnSyncQueue`：

```java
// 判断节点是否在同步队列中
final boolean isOnSyncQueue(Node node) {
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    if (node.next != null) // If has successor, it must be on queue
        return true;
    return findNodeFromTail(node);
}
```

该方法很好理解，只要`waitStatus`的值还为`Node.CONDITION`，则它一定还在`condtion`队列中，自然不可能在`sync`队列里面；而每一个调用了`enq`方法入队的线程：

```java
// 入队列，返回前驱节点
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) { // 即使这一步失败了，next.prev一定是有值的
                t.next = node; // 如果t.next有值，说明上面的compareAndSetTail方法一定成功了，则当前节点成为了新的尾节点
                return t; // 返回当前节点的前驱节点
            }
        }
    }
}
```

哪怕在设置`compareAndSetTail`这一步失败了，它的`prev`必然也是有值的。因此这两个条件只要有一个满足，就说明节点必然不在`sync queue`队列中。

另一方面，如果`node.next`有值，则说明它不仅在`sync queue`中，并且在它后面还有别的节点，则只要它有值，该节点必然在`sync queue`中。

如果以上都不满足，说明这里出现了尾部分叉的情况，就从尾节点向前寻找这个节点：

```java
// 从同步队列尾部开始寻找节点
private boolean findNodeFromTail(Node node) {
    Node t = tail;
    for (;;) {
        if (t == node)
            return true;
        if (t == null)
            return false;
        t = t.prev;
    }
}
```

这里当然还是有可能出现从尾部反向遍历找不到的情况，但是不用担心，线程A还在自旋`while`循环中，无论如何，节点最后总会入队成功的。最终，`transferAfterCancelledWait`将返回false。

再回到`transferAfterCancelledWait`调用处：

```java
// 检查中断模式
private int checkInterruptWhileWaiting(Node node) {
    return Thread.interrupted() ?
        (transferAfterCancelledWait(node) ? THROW_IE : REINTERRUPT) :
        0;
}
```

由于`transferAfterCancelledWait`返回了false，则`checkInterruptWhileWaiting`方法将返回`REINTERRUPT`，这说明线程在退出该方法时只需要再次中断。

再回到`checkInterruptWhileWaiting`方法的调用处：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0) // 在这里，跳出循环
            break;
    }
    // 当前interruptMode=REINTERRUPT，无论这里是否进入if语句，该值不变
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE) 
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

此时`interruptMode`的值为`REINTERRUPT`，线程将直接跳出`while`循环。接下来就和上面的情况1一样，线程依然还是去争锁，这一步依然是阻塞式的，获取到锁则退出，获取不到锁则会被挂起。

另外由于现在`interruptMode`的值已经为`REINTERRUPT`，因此无论在争锁的过程中是否发生过中断`interruptMode`的值都还是`REINTERRUPT`。

接着就是将节点从`condition queue`中剔除，与情况1不同的是，在`signal`方法成功将`node`加入到`sync queue`时，该节点的`nextWaiter`已经是null了，所以这里这一步不需要执行。

再接下来就是报告中断状态：

```java
// 根据中断模式，汇报中断状态
private void reportInterruptAfterWait(int interruptMode)
    throws InterruptedException {
    // signal之前被中断
    if (interruptMode == THROW_IE)
        throw new InterruptedException();
    // signal之后被中断
    else if (interruptMode == REINTERRUPT)
        selfInterrupt();
}
// 自我中断
static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

注意，这里并没有抛出中断异常，而只是将当前线程再中断一次。

至此，情况2.1(被唤醒时，已经发生了中断，但此时线程已经被`signa`过了)就分析完了，这里简单总结一下：

1. 线程从挂起的地方被唤醒，此时既发生过中断，又发生过`signal`；
2. 随后，通过`transferAfterCancelledWait`确认了线程的`waitStatus`值已经不为`Node.CONDITION`，说明`signal`发生于中断之前；
3. 然后通过自旋的方式，等待`signa`l方法执行完成，确保当前节点已经被成功添加到`sync queue`中；
4. 接下来线程将在`sync queue`中以阻塞的方式获取锁，如果获取不到，将会被再次挂起；
5. 最后通过`reportInterruptAfterWait`将当前线程再次中断，但是不会抛出`InterruptedException`。

##### 情况2.2：被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断

这种情况比上面的情况简单一点，既然被唤醒时没有发生中断，那基本可以确信线程是被`signal`唤醒的，但是不要忘记还存在“**假唤醒**”这种情况，因此线程依然还是要检测被唤醒的原因。

那么如何区分到底是假唤醒还是因为被`signal`而唤醒？

如果线程是因为`signal`而被唤醒，则由前面分析的`signal`方法可知，线程最终都会离开`condition queue` 进入`sync queue`中，所以线程只需要判断被唤醒时，线程是否已经在`sync queue`中即可：

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);  // 线程将在这里被唤醒
        // 由于现在没有发生中断，所以interruptMode目前为0
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

线程被唤醒时，暂时还没有发生中断，所以这里`interruptMode = 0`， 表示没有中断发生，所以线程将继续`while`循环，这时将通过`isOnSyncQueue`方法判断当前线程是否已经在`sync queue`中。由于已经发生过`signal`了，则此时`node`必然已经在`sync queue`中，所以`isOnSyncQueue`将返回true，线程将退出`while`循环。

不过这里插一句，如果`isOnSyncQueue`检测到当前节点不在`sync queue`中，则说明既没有发生中断，也没有发生过`signal`，则当前线程是被“**假唤醒**”的，那么线程将再次进入循环体，将线程挂起。

退出`while`循环后接下来还是利用`acquireQueued`争锁，因为前面没有发生中断，则`interruptMode=0`，这时，如果在争锁的过程中发生了中断，则acquireQueued将返回true，则此时interruptMode将变为`REINTERRUPT`。

接下是判断`node.nextWaiter != null`，由于在调用`signal`方法时已经将节点移出了队列，所有这个条件也不成立。

最后就是汇报中断状态，此时`interruptMode`的值为`REINTERRUPT`，说明线程在被`signal`后又发生了中断，这个中断发生在抢锁的过程中，这个中断来的太晚了，因此线程只是再次自我中断一下。

至此，情况2.2(被唤醒时，并没有发生中断，但是在抢锁的过程中发生了中断)就分析完了，这种情况和2.1很像，区别是一个是在唤醒后就被发现已经发生了中断，一个是唤醒后没有发生中断，但是在抢锁的过成中发生了中断，**但无论如何，这两种情况都会被归结为“中断来的太晚了”**，中断模式为`REINTERRUPT`，情况2.2的总结如下：

1. 线程被`signal`方法唤醒，此时并没有发生过中断；
2. 因为没有发生过中断，线程将从`checkInterruptWhileWaiting`处返回，此时`interruptMode=0`；
3. 接下来线程回到`while`循环中，因为`signal`方法保证了将节点添加到`sync queue`中，此时`while`循环条件不成立，循环退出；
4. 接下来线程将`在sync queue`中以阻塞的方式获取，如果获取不到锁，将会被再次挂起；
5. 线程获取到锁返回后，线程检测到在获取锁的过程中发生过中断，并且此时`interruptMode=0`，这时将`interruptMode`修改为`REINTERRUPT`；
6. 最后线程通过`reportInterruptAfterWait`将当前线程再次中断，但是不会抛出`InterruptedException`。

这里再总结一下情况2(中断发生时，线程已经被`signal`过)，这种情况对应于中断发生`signal`之后，**不管这个中断是在抢锁之前就已经发生了还是抢锁的过程中发生了，只要它是在signal之后发生的，就认为它来的太晚了，线程将忽略这个中断。因此，从await()方法返回的时候，只会将当前线程重新中断一下，而不会抛出中断异常。**

#### 情况3： 一直没有中断发生

这种情况就更简单，它的大体流程和上面的情况2.2差不多，只是在抢锁的过程中也没有发生异常，则`interruptMode`为0，没有发生过中断，因此不需要汇报中断。则线程就从`await()`方法处正常返回。

### 5.await()总结

至此，分析完了`await()`方法的执行流程，这里对整个方法做出总结：

1. 进入`await()`时必须是已经持有了锁；
2. 离开`await()`时同样必须是已经持有了锁；
3. 调用`await()`会使得当前线程被封装成Node扔进条件队列，然后释放所持有的锁；
4. 释放锁后，当前线程将在`condition queue`中被挂起，等待`signal`或者中断；
5. 线程被唤醒后会将会离开`condition queue`进入`sync queue`中进行抢锁；
6. 若在线程抢到锁之前发生过中断，则根据中断发生在`signal`之前还是之后记录中断模式；
7. 线程在抢到锁后进行善后工作（离开`condition queue`， 处理中断异常）；
8. 线程已经持有了锁，从`await()`方法返回。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/condition-4.png)

在这一过程中尤其要关注中断，如前面所说，**中断和signal所起到的作用都是将线程从condition queue中移除，加入到sync queue中去争锁，所不同的是，signal方法被认为是正常唤醒线程，中断方法被认为是非正常唤醒线程，如果中断发生在signal之前，则线程在最终返回时，应当抛出InterruptedException；如果中断发生在signal之后，就认为线程本身已经被正常唤醒了，这个中断来的太晚了，线程直接忽略它，并在await()返回时再自我中断一下，这种做法相当于将中断推迟至await()返回时再发生。**

### 6.awaitUninterruptibly()

在前面分析的`await()`方法中，中断起到了和`signal`同样的效果，但是中断属于将一个等待中的线程非正常唤醒，可能即使线程被唤醒后，也抢到了锁，但是却发现当前的等待条件并没有满足，则还是得把线程挂起。因此有时候并不希望`await`方法被中断，`awaitUninterruptibly()`方法即实现了这个功能：

```java
// 不响应中断的条件等待
public final void awaitUninterruptibly() {
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean interrupted = false;
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if (Thread.interrupted())
            interrupted = true; // 发生中断后线程依旧留在condition queue中，将再次被挂起
    }
    if (acquireQueued(node, savedState) || interrupted)
        selfInterrupt();
}
```

首先，从方法签名上就可以看出，这个方法不会抛出中断异常，拿它和`await()`方法对比一下：

```java
// 可中断的条件等待
public final void await() throws InterruptedException {
    if (Thread.interrupted())  // 不同之处
        throw new InterruptedException(); // 不同之处
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    int interruptMode = 0;  // 不同之处
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)  // 不同之处
            break;
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)  // 不同之处
        interruptMode = REINTERRUPT;  // 不同之处
    if (node.nextWaiter != null)  // 不同之处
        unlinkCancelledWaiters(); // 不同之处
    if (interruptMode != 0) // 不同之处
        reportInterruptAfterWait(interruptMode); // 不同之处
}
```

由此可见，`awaitUninterruptibly()`全程忽略中断，即使是当前线程因为中断被唤醒，该方法也只是简单的记录中断状态，然后再次被挂起(因为并没有任何操作将它添加到`sync queue`中)。要使当前线程离开`condition queue`去争锁，则**必须是**发生了`signal`事件。

最后，当线程在获取锁的过程中发生了中断，该方法也是不响应，只是在最终获取到锁返回时，再自我中断一下。可以看出，该方法和“中断发生于`signal`之后的”`REINTERRUPT`模式的`await()`方法很像。

至此该方法就分析完了，它的核心思想是：

1. 中断虽然会唤醒线程，但是不会导致线程离开`condition queue`，如果线程只是因为中断而被唤醒，则将再次被挂起；
2. 只有`signal`方法会使得线程离开`condition queue`；
3. 调用该方法时或者调用过程中如果发生了中断，仅仅会在该方法结束时再自我中断一下，不会抛出`InterruptedException`。

### 7.awaitNanos(long nanosTimeout)

前面分析的方法，无论是`await()`还是`awaitUninterruptibly()`，它们在抢锁的过程中都是阻塞式的，即一直等到抢到了锁才能返回，否则线程还是会被挂起，这样带来一个问题就是线程如果长时间抢不到锁，就会一直被阻塞，因此有时候更需要带超时机制的抢锁，这一点和带超时机制的`wait(long timeout)`很像：

```java
// 超时等待
public final long awaitNanos(long nanosTimeout) throws InterruptedException {
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);*/
    // 绝对截止日期，单位ns
    final long deadline = System.nanoTime() + nanosTimeout;
    /*int interruptMode = 0;
    while (!isOnSyncQueue(node)) */{
        // 时间到了，直接转移到同步队列
        if (nanosTimeout <= 0L) {
            transferAfterCancelledWait(node);
            break;
        }
        // 阻塞等待nanosTimeout(ns)
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        // 由于中断被唤醒
        /*if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;*/
        // signal或者假醒
        nanosTimeout = deadline - System.nanoTime(); // 剩余时间
    }
    /*if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);*/
    // 截止时间-获取锁之后的最终时间
    return deadline - System.nanoTime();
}
```

该方法几乎和`await()`方法一样，只是多了超时时间的处理，上面已经把和`await()`方法相同的部分注释起来了，只留下了不同的部分，这样它们的区别就变得更明显了。

该方法的主要设计思想是，如果设定的超时时间还没到，就将线程挂起；超过等待的时间了，就将线程从`condtion queue`转移到`sync queue`中。注意这里对于超时时间有一个小小的优化——当设定的超时时间很短时(小于`spinForTimeoutThreshold`的值)，线程就是简单的自旋，而不是将线程挂起，以减少挂起线程和唤醒线程所带来的时间消耗。

这里需要注意`awaitNanos(0)`的意义，`Object.wait(0)`的含义是无限期等待，而在`awaitNanos(long nanosTimeout)`方法中是这样处理`awaitNanos(0)`的：

```java
if (nanosTimeout <= 0L) {
    transferAfterCancelledWait(node);
    break;
}
```

从这里可以看出，如果设置的等待时间本身就小于等于0，当前线程是会直接从`condition queue`中转移到`sync queue`中的，并不会被挂起，也不需要等待`signal`，这一点确实是更复合逻辑。如果需要线程只有在`signal`发生的条件下才会被唤醒，则应该用上面的`awaitUninterruptibly()`方法。

### 8.await(long time, TimeUnit unit)

分析完`awaitNanos(long nanosTimeout)`，再分析`await(long time, TimeUnit unit)`方法比较简单，它就是在`awaitNanos(long nanosTimeout)`的基础上多了对于超时时间的时间单位的设置，但是在内部实现上还是会把时间转成纳秒去执行，这里直接拿它和上面的`awaitNanos(long nanosTimeout)`方法进行对比，只给出不同的部分：

```java
// 超时等待
public final boolean await(long time, TimeUnit unit) throws InterruptedException {
    // 转为纳秒单位
    long nanosTimeout = unit.toNanos(time);
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    final long deadline = System.nanoTime() + nanosTimeout;*/
    // 是否超时
    /*boolean timedout = false;
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) {
        if (nanosTimeout <= 0L) {*/
            timedout = transferAfterCancelledWait(node);
            /*break;
        }
        if (nanosTimeout >= spinForTimeoutThreshold)
            LockSupport.parkNanos(this, nanosTimeout);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
        nanosTimeout = deadline - System.nanoTime();
    }
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);*/
    return !timedout;
}
```

可以看出，这两个方法主要的差别就体现在返回值上面，`awaitNanos(long nanosTimeout)`的返回值是剩余的超时时间，如果该值大于0，说明超时时间还没到，则说明该返回是由`signal`行为导致的，而`await(long time, TimeUnit unit)`的返回值就是`transferAfterCancelledWait(node)`的值，如果调用该方法时，`node`还没有被`signal`过则返回true，`node`已经被`signal`过，则返回false。因此当`await(long time, TimeUnit unit)`方法返回true，则说明在超时时间到之前就已经发生过`signal`，该方法的返回是由`signal`方法导致的而不是超时时间。

综上，调用`await(long time, TimeUnit unit)`其实就等价于调用`awaitNanos(unit.toNanos(time)) > 0 `方法。关于这一点，在[Lock与Condition接口功能分析](https://xuanjian1992.top/2019/01/17/Lock%E4%B8%8ECondition%E6%8E%A5%E5%8F%A3%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90/)的时候已经提过。

### 9.awaitUntil(Date deadline)

`awaitUntil(Date deadline)`方法与上面的几种带超时的方法也基本类似，所不同的是它的超时时间是一个绝对时间，直接拿它来和上面的`await(long time, TimeUnit unit)`方法对比：

```java
public final boolean awaitUntil(Date deadline) throws InterruptedException {
    long abstime = deadline.getTime(); // 绝对时间(ms)
    /*if (Thread.interrupted())
        throw new InterruptedException();
    Node node = addConditionWaiter();
    int savedState = fullyRelease(node);
    boolean timedout = false;  // 是否超时
    int interruptMode = 0;
    while (!isOnSyncQueue(node)) */{
        if (System.currentTimeMillis() > abstime) {
            /*timedout = transferAfterCancelledWait(node);
            break;
        }*/
        LockSupport.parkUntil(this, abstime);  // 绝对时间的阻塞
        /*if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;*/
    }
    /*
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
    return !timedout;*/
}
```

可见，这里大段的代码都是重复的，区别就是在超时时间的判断上使用了绝对时间，其实这里的`deadline`就和`awaitNanos(long nanosTimeout)`以及`await(long time, TimeUnit unit)`内部的`deadline`变量是等价的，另外在这个方法中，没有使用`spinForTimeoutThreshold`进行自旋优化，因为一般调用这个方法，目的就是设定一个较长的等待时间，否则使用上面的相对时间会更方便一点。

至此，AQS对于`Condition`接口的实现源码分析全部结束。

## 参考

- [线程间的同步与通信(4)——Lock 和 Condtion](https://segmentfault.com/a/1190000016449988)
- [逐行分析AQS源码(4)——Condition接口实现](https://segmentfault.com/a/1190000016462281)

