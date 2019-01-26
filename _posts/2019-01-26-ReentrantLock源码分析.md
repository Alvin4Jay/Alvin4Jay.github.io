---
layout:     post
title:      ReentrantLock源码分析
subtitle:   ReentrantLock
date:       2019-01-26
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# ReentrantLock源码分析

### 一、ReentrantLock简介

`java.util.concurrent.locks.ReentrantLock`是JDK1.5后提供的锁，可以提供与`synchronized`监视器锁相同的独占锁、可重入互斥功能和内存语义，并且还具有非阻塞的尝试加锁、定时等待的加锁等功能，提供了和`wait()`、`notify()`相同功能的`Condition`类，并且一个锁可以绑定多个条件队列。

[Lock与Condition接口功能分析](https://xuanjian1992.top/2019/01/17/Lock%E4%B8%8ECondition%E6%8E%A5%E5%8F%A3%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90/)这篇文章介绍了`Lock`接口，这篇文章以`ReentrantLock`为例，来分析`Lock`锁接口的基本实现。观察下面`Lock`接口中的方法与`ReentrantLock`类各方法实现的对照表：

| Lock 接口                         | ReentrantLock 类实现                           |
| --------------------------------- | ---------------------------------------------- |
| lock()                            | sync.lock()                                    |
| lockInterruptibly()               | sync.acquireInterruptibly(1)                   |
| tryLock()                         | sync.nonfairTryAcquire(1)                      |
| tryLock(long time, TimeUnit unit) | sync.tryAcquireNanos(1, unit.toNanos(timeout)) |
| unlock()                          | sync.release(1)                                |
| newCondition()                    | sync.newCondition()                            |

可知`ReentrantLock`类对于`Lock`接口的实现都是直接“转交”给`sync`对象的。

### 二、核心属性

`ReentrantLock`只有一个`sync`属性，这个属性提供了所有的实现。上面介绍`ReentrantLock`对`Lock`接口的实现的时候就说到，它对所有的`Lock`接口方法的实现都调用了`sync`的方法，这个`sync`就是`ReentrantLock`的属性，所声明的类型`Sync`继承自`AQS`。

```java
// Sync属性
private final Sync sync;
// 同步器(内部类)
abstract static class Sync extends AbstractQueuedSynchronizer {
    abstract void lock();
    // ...
}
```

在`Sync`类中，定义了一个抽象方法`lock`，该方法应当由继承它的子类来实现，关于继承它的子类，在下一节分析构造函数时再看。

### 三、构造函数

`ReentrantLock`共有两个构造函数：

```java
// 构造器
public ReentrantLock() {
    sync = new NonfairSync();
}
// boolean fair，使用公平锁或非公平锁
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

默认的构造函数使用了非公平锁，另外一个构造函数通过传入一个`boolean`类型的`fair`变量来决定使用公平锁还是非公平锁。其中，`FairSync`和`NonfairSync`的定义如下：

```java
// 公平锁实现
static final class FairSync extends Sync {
    
    final void lock() {//省略实现}

    protected final boolean tryAcquire(int acquires) {//省略实现}
}

// 非公平锁实现
static final class NonfairSync extends Sync {
    
    final void lock() {//省略实现}

    protected final boolean tryAcquire(int acquires) {//省略实现}
}
```

这里默认创建的是非公平锁，原因是非公平锁的效率高，当一个线程请求非公平锁时，如果在**发出请求的同时**该锁变成可用状态，那么这个线程会跳过队列中所有的等待线程而获得锁，这类似于现实中的"插队"现象，这也就是它被称作非公平锁的原因。
之所以使用这种方式是因为：

<font color="red">在恢复一个被挂起的线程(未获取锁)与该线程真正运行(获取锁)之间存在着严重的延迟。</font>

在公平锁模式下，大家讲究先来后到，如果当前线程A在请求锁，即使现在锁处于可用状态，它也得在队列的末尾排队，这时需要唤醒排在等待队列队首的线程H(在AQS中其实是次头节点)，由于恢复一个被挂起的线程并且让它真正运行起来需要较长时间，那么**这段时间锁就处于空闲状态，时间和资源就白白浪费了**，非公平锁的设计思想就是将这段白白浪费的时间利用起来——由于线程A在请求锁的时候本身就处于运行状态，因此如果此时把锁给它，它就会立即执行自己的任务，因此线程A有机会在线程H完全唤醒之前获得、使用以及释放锁。这样就可以把线程H恢复运行的这段时间给利用起来，结果就是线程A更早的获取了锁，线程H获取锁的时刻也没有推迟。因此提高了吞吐量。

当然，非公平锁仅仅是在**当前线程请求锁并且锁处于可用状态时**有效，当请求锁时如果锁已经被其他线程占有，当前线程就只能还是老老实实的去排队。

无论是非公平锁的实现`NonfairSync`，还是公平锁的实现`FairSync`，它们都实现、重写了`lock`方法和`tryAcquire`方法，这两个方法都将用于获取一个锁。

### 四、ReentrantLock对于Lock接口方法的实现

##### 1.lock()

###### 公平锁实现

关于`ReentrantLock`对于`lock`方法的公平锁的实现逻辑，在[AQS源码分析1——独占锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901-%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)中已经讲过，这里不再介绍。

###### 非公平锁实现

接下来看看非公平锁的实现逻辑：

```java
// NonfairSync中的lock方法
final void lock() {
    // 先尝试直接获取锁
    if (compareAndSetState(0, 1))
        // 成功，则设置当前线程为持有锁的线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        // 失败，则转为正常的获取锁流程
        acquire(1);
}
```

对比公平锁的`lock`方法：

```java
// FairSync中的lock方法
final void lock() {
	// 获取锁
    acquire(1);
}
```

可见，相比公平锁，非公平锁在当前锁没有被占用时，可以直接尝试去获取锁，而不用排队，所以线程在一开始就尝试使用CAS操作去抢锁，只有在该操作失败后，才会调用AQS的`acquire`方法。

由于`acquire`方法中除了`tryAcquire`由子类实现外，其余都由AQS实现，在[AQS源码分析1——独占锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901-%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)中已经介绍过，这里不再介绍。仅仅看一下非公平锁`tryAcquire`方法的实现：

```java
// NonfairSync中的tryAcquire方法实现
protected final boolean tryAcquire(int acquires) {
    // 非公平的方式
    return nonfairTryAcquire(acquires);
}
```

它调用了`Sync`类的`nonfairTryAcquire`方法：

```java
// 非公平的尝试获取锁
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 只有这一处和公平锁的实现不同，其它的完全一样。
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

可以拿它和公平锁的`tryAcquire`对比一下：

```java
// FairSync中的tryAcquire方法实现
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        // 判断是否有其他线程排队获取锁
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这两个方法几乎一模一样，唯一的区别是非公平锁在抢锁时不需要调用`hasQueuedPredecessors`方法先去判断是否有线程排在自己前面，而是直接争锁，其它的完全和公平锁一致。

```java
// 判断是否有任何线程排队获取锁的时间比当前线程长
public final boolean hasQueuedPredecessors() {
    Node t = tail; // 尾节点
    Node h = head; // 头结点
    Node s; 
    // 存在线程排队获取锁的时间比当前线程长的条件
    // 1.h!=t，表示存在线程在等待锁
    // 2.1 (s = h.next) == null，表示当前有节点正在入队列，完成了enq()方法入队列的前两步，第三步
    // 还没完成，所以h.next=null。并且正在入队列的线程肯定不是当前线程(在获取锁)。 或者
    // 2.2 s.thread != Thread.currentThread())，表示等待队列中第一个等待锁的节点对应的线程不是当
    // 前线程。
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

##### 2.lockInterruptibly()

上述的`lock`方法是阻塞式的，抢到锁就返回，抢不到锁就将线程挂起，并且在抢锁的过程中是不响应中断的。`lockInterruptibly`提供了一种响应中断的锁获取方式，在`ReentrantLock`中，**无论是公平锁还是非公平锁，这个方法的实现都是一样的**：

```java
// 响应中断的互斥锁获取
public void lockInterruptibly() throws InterruptedException {
    sync.acquireInterruptibly(1);
}
```

他们都调用了AQS的`acquireInterruptibly`方法：

```java
// 响应中断的互斥锁获取
public final void acquireInterruptibly(int arg) throws InterruptedException {
    // 如果调用该方法时已经被中断，抛出InterruptedException异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 尝试获取(公平锁和非公平锁，不同实现逻辑)
    if (!tryAcquire(arg))
        // 排队获取
        doAcquireInterruptibly(arg);
}
```

该方法首先检查当前线程是否已经被中断过，如果已经被中断，则立即抛出`InterruptedException`(这一点是`lock`接口的`lockInterruptibly`方法要求的。

如果调用这个方法时，当前线程还没有被中断过，则接下来先尝试用普通的方法来获取锁（`tryAcquire`）。如果获取成功，则直接返回；否则，与前面的`lock`方法一样，需要将当前线程包装成`Node`扔进等待队列。不同的是，这次，在队列中尝试获取锁时，如果发生中断，需要对中断做出响应，并抛出异常。

```java
// 响应中断的获取互斥锁
private void doAcquireInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE); // 添加节点
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return; // 与acquireQueued方法的不同之处
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException(); // 与acquireQueued方法的不同之处
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

如果在上面分析`lock`方法的时候已经理解了[acquireQueued方法](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901-%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)，那么再看这个方法就很轻松，这里把`lock`方法中的`acquireQueued`拿出来和上面对比一下：

```java
acquireQueued(addWaiter(Node.EXCLUSIVE), arg)) // 添加节点入队列
// 返回结果代表是否发生了中断
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false; //不同之处
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted; //不同之处
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true; //不同之处
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

通过代码对比可以看出，`doAcquireInterruptibly`和`acquireQueued(addWaiter(Node.EXCLUSIVE), arg))`的调用本质上并无区别。只不过对于`addWaiter(Node.EXCLUSIVE)`，一个是外部调用，通过参数传进来；一个是直接在方法内部调用。所以这两个方法的逻辑几乎是一样的，唯一的不同是在`doAcquireInterruptibly`中，当检测到中断后，不再是简单的记录中断状态，而是直接抛出`InterruptedException`。

当抛出中断异常后，在返回前，将进入`finally`代码块进行善后工作。此时`failed`为`true`，则将调用`cancelAcquire`方法：

```java
// 取消获取锁
private void cancelAcquire(Node node) {
    // node节点不存在，直接忽略
    if (node == null)
        return;
	// node节点thread置为null
    node.thread = null;

    // 由当前节点向前遍历，跳过那些已经被cancel的节点，找到第一个正常的节点pred
    Node pred = node.prev;
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;
    
    // 从当前节点向前开始查找，找到第一个waitStatus<=0的Node, 该节点为pred
    // predNext即是pred节点的下一个节点
    // 到这里可知，pred节点是没有被cancel的节点，但是pred节点往后，一直到当前节点Node都处于被Cancel的状态
    Node predNext = pred.next;

    // 将当前节点的waitStatus状态设为Node.CANCELLED
    node.waitStatus = Node.CANCELLED;

    // 如果当前节点是尾节点，则将之前找到的节点pred重新设置成尾节点，并将pred节点的next属性由predNext修改成null
    // 这一段本质上是将pred节点后面的节点全部移出队列，因为它们都被cancel掉了
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {
        // 到这里说明当前节点已经不是尾节点，或者设置新的尾节点失败。在并发条件下，什么都有可能发生。
        // 即在当前线程运行这段代码的过程中，其他线程可能已经入队，成为了新的尾节点。由在分析
        // lock方法的实现时可知，新的节点入队后会设置闹钟，将找一个没有CANCEL的前驱节点，将它的
        // waitStatus设置成SIGNAL以唤醒自己。所以，在当前节点的后继节点入队后，可能将当前节点的
        // waitStatus修改成了SIGNAL。而在这时，当前节点对应的线程发起了中断，又将这个waitStatus
        // 修改成CANCELLED，所以在当前节点出队前，需要根据条件按需负责唤醒后继节点。
        int ws;
        // pred为正常等待的节点，且不为头结点，如果为SIGNAL或者设置为SIGNAL成功，则移除predNext
        // 到node之间的节点
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            // pred为头结点，说明node之前没有正常等待锁的节点了，直接唤醒node的后继节点
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

这个`cancelAcquire`方法不仅是取消了当前节点的排队，还会同时将当前节点之前(`pred`之后)的那些已经`CANCEL`掉的节点移出队列。这里需要注意，这里是在并发条件下，现在新的节点可能已经入队，成为了新的尾节点，这将会导致`node == tail && compareAndSetTail(node, pred)`这一条件失败。

这个方法的前半部分是基于当前节点是队列的尾节点这一情况，即在执行这个方法时，没有新的节点入队，这部分的逻辑比较简单，直接看代码中的注释即可理解。

而后半部分是基于有新的节点加进来，当前节点已经不再是尾节点的情况，详细看看这个else部分：

```java
if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
} else { // 看这里
    int ws;
    if (pred != head &&
        ((ws = pred.waitStatus) == Node.SIGNAL ||
         (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
        pred.thread != null) {
        Node next = node.next;
        if (next != null && next.waitStatus <= 0)
            compareAndSetNext(pred, predNext, next); // 将pred节点的后继节点改为当前节点的后继节点
    } else {
        unparkSuccessor(node);
    }

    node.next = node; // help GC
}
```

（这里再说明一下`pred`变量所代表的含义：表示从当前节点向前遍历所找到的第一个没有被`cancel`的节点）

执行到else代码块，目前的状况如下：

1. 当前线程被中断了，已经将它的`Node`的`waitStatus`属性设为`CANCELLED`，`thread`属性置为`null`；
2. 在执行这个方法期间，又有其他线程加入到队列中来，成为了新的尾节点，使得当前线程已经不是队尾了。

在这种情况下，将执行else语句，将`pred`节点的后继节点改为当前节点的后继节点(`compareAndSetNext(pred, predNext, next)`)，即将从`pred`节点开始（不包含`pred`节点）一直到当前节点（包括当前节点）之间的所有节点全部移出队列，因为他们都是被`cancel`的节点。当然这是基于一定条件的，条件为：

1. `pred`节点不是头节点；
2. `pred`节点的`thread`不为`null`；
3. `pred`节点的`waitStatus`属性是`SIGNAL`或者是小于等于0但是被成功的设置成`SIGNAL`。

上面这三个条件保证了`pred`节点确实是一个正在正常等待锁的线程，并且它的`waitStatus`属性为`SIGNAL`。
如果这一条件无法被满足，那么将直接通过`unparkSuccessor`唤醒`node`的后继节点。

到这里，总结一下`cancelAcquire`方法：

1. 如果要`cancel`的节点`node`已经是尾节点，则在`node`后面并没有节点需要唤醒，只需要从当前节点`node`(即尾节点)开始向前遍历，找到第一个正常等待的节点`pred`，并将两者之前的已经`cancel`的节点移出队列即可；
2. 如果要`cancel`的节点`node`后面还有别的节点，并且找到的`pred`节点处于正常等待状态，则还是直接将从当前节点开始，到`pred`节点之间的所有节点移出队列，这里并不需要唤醒当前节点的后继节点，因为它已经接在了`pred`的后面，`pred`的`waitStatus`已经被置为`SIGNAL`，它会负责唤醒后继节点；
3. 如果上面的条件不满足，则说明当前节点往前已经没有在等待中的线程了，那么就直接将后继节点唤醒。

第3条只是把当前节点的后继节点唤醒，并没有将当前节点移出队列，但是当前节点已经取消排队，不是应该移出队列吗？其实在后继节点被唤醒后，它会在抢锁时调用`shouldParkAfterFailedAcquire`方法，调用过程中将跳过已经`CANCEL`的节点，那个时候，当前节点就会被移出队列。

##### 3.tryLock()

由于`tryLock`方法仅仅是用于检查锁在当前调用的时候是不是可获得的，所以即使现在使用的是公平锁，在调用这个方法时，当前线程也会直接尝试去获取锁，哪怕这个时候队列中还有在等待中的线程。所以这一方法对于公平锁和非公平锁的实现是一样的，它被定义在`Sync`类中，由`FairSync`和`NonfairSync`直接继承使用：

```java
// 尝试获取锁
public boolean tryLock() {
    return sync.nonfairTryAcquire(1);
}
// 非公平的尝试获取锁
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
    	// 直接获取锁
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

这个`nonfairTryAcquire`在上面分析非公平锁的`lock`方法时已经讲过，这里只是简单的方法复用。**该方法不存在任何和队列相关的操作**，仅仅就是直接尝试去获锁，成功就返回`true`，失败就返回`false`。

可能大家会觉得公平锁也使用这种方式去`tryLock`就丧失了公平性，但是这种方式在某些情况下是非常有用的，如果还是想维持公平性，那应该使用**带超时机制**的`tryLock`。

##### 4.tryLock(long timeout, TimeUnit unit)

与立即返回的`tryLock()`不同，`tryLock(long timeout, TimeUnit unit)`带了超时时间，所以是阻塞式的，并且在获取锁的过程中可以响应中断异常：

```java
// 在给定的时间内尝试获取锁，可响应中断
public boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
// 在给定的时间内尝试获取锁，可响应中断(纳秒ns)
public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    // 在调用该方法时已经中断，直接抛出异常
    if (Thread.interrupted())
        throw new InterruptedException();
    // 这里公平锁和非公平锁tryAcquire(arg)逻辑不一样
    return tryAcquire(arg) || doAcquireNanos(arg, nanosTimeout);
}
```

与`lockInterruptibly`方法一样，该方法首先检查当前线程是否已经被中断过，如果已经被中断，则立即抛出`InterruptedException`。

随后通过调用`tryAcquire`和`doAcquireNanos(arg, nanosTimeout)`方法来尝试获取锁，注意，这时公平锁和非公平锁对于`tryAcquire`方法就有不同的实现，公平锁首先会检查当前有没有别的线程在队列中排队，关于公平锁和非公平锁对`tryAcquire`的不同实现上面已经讲过，这里不再说明。直接来看`doAcquireNanos`，这个方法其实和前面说的`doAcquireInterruptibly`方法很像，将相同的部分注释掉，直接看不同的部分：

```java
// 在给定时间内获取锁，true表示获取锁成功；false表示超时、失败
private boolean doAcquireNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout; // 截止时间
    /*final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;*/
                return true; // doAcquireInterruptibly中为 return
            /*}*/
            nanosTimeout = deadline - System.nanoTime(); // 还剩的时间
            if (nanosTimeout <= 0L) // 等到时间已到，失败
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout); // 阻塞nanosTimeout时间
            if (Thread.interrupted()) // 如果中断，直接抛出异常
                throw new InterruptedException();
       /* }
    } finally {
        if (failed)
            cancelAcquire(node);
    }*/
}
```

可以看出，这两个方法的逻辑相差不差，只是`doAcquireNanos`多了对于截止时间的检查。

不过这里有两点需要注意，一个是`doAcquireInterruptibly`是没有返回值的，而`doAcquireNanos`是有返回值的。这是因为`doAcquireNanos`有可能因为获取到锁而返回，也有可能因为超时时间到了而返回，为了区分这两种情况，因为超时时间而返回时，将返回`false`，代表并没有获取到锁。

另外一点值得注意的是，上面有一个`nanosTimeout > spinForTimeoutThreshold`的条件，在它满足的时候才会将当前线程挂起指定的时间，这个`spinForTimeoutThreshold`的定义如下：

```java
/**
 * The number of nanoseconds for which it is faster to spin
 * rather than to use timed park. A rough estimate suffices
 * to improve responsiveness with very short timeouts.
 */
static final long spinForTimeoutThreshold = 1000L;
```

它是个阈值，是为了提升性能用的。如果当前剩下的等待时间已经很短了，就直接使用自旋的形式等待，而不是将线程挂起，可见作者为了尽可能地优化AQS锁的性能费足了心思。

##### 5.unlock()

`unlock`操作用于释放当前线程所占用的锁，这一点对于公平锁和非公平锁的实现是一样的，所以该方法被定义在`Sync`类中，由`FairSync`和`NonfairSync`直接继承使用：

```java
// 释放锁
public void unlock() {
    sync.release(1);
}
```

关于`ReentrantLock`释放锁的操作，在[AQS源码分析1——独占锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901-%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)中已经详细介绍，这里不再说明。

##### 6.newCondition()

`ReentrantLock`本身并没有实现`newCondition`方法，它是直接调用了AQS的`newCondition`方法

```java
// 创建一个Condition实例
public Condition newCondition() {
    return sync.newCondition();
}
```

而AQS的`newCondtion`方法是简单地创建了一个`ConditionObject`对象：

```java
final ConditionObject newCondition() {
    return new ConditionObject();
}
```

关于`ConditionObject`对象的源码分析，请参见[AQS源码分析3——Condition接口实现之ConditionObject分析](https://xuanjian1992.top/2019/01/20/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%903-Condition%E6%8E%A5%E5%8F%A3%E5%AE%9E%E7%8E%B0%E4%B9%8BConditionObject%E5%88%86%E6%9E%90/)

### 五、总结

`ReentrantLock`对于`Lock`接口方法的实现大多数是直接调用AQS的方法，AQS中已经完成了大多数逻辑的实现，子类只需要直接继承使用即可，这足见AQS在并发编程中的地位。当然，有一些逻辑还是需要`ReentrantLock`自己去实现的，例如`tryAcquire`的逻辑。

### 参考文章

- [Class ReentrantLock](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantLock.html)
- [线程间的同步与通信(5)——ReentrantLock源码分析](https://segmentfault.com/a/1190000016503518)
- [ReentrantLock使用和源码分析](https://liuzhengyang.github.io/2017/05/26/reentrantlock/)