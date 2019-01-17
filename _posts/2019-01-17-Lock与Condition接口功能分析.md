---
layout:     post
title:      Lock与Condition接口功能分析
subtitle:   接口功能分析
date:       2019-01-17
author:     Jay
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# Lock与Condition接口分析

### 一、Lock接口

`Lock`接口的出现是为了拓展现有的监视器锁(`synchronized`)，实现了可定时、可轮询与可中断的锁获取操作，公平队列等。

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    void unlock();
    
    Condition newCondition();
}
```

典型的使用方式如下：

```java
Lock l = ...;
l.lock();
try {
    // access the resource protected by this lock
} finally {
    l.unlock();
}
```

##### 1.锁的获取

`Lock`接口定义了四种获取锁的方式。

- `lock()`
  - **阻塞式获取**，在没有获取到锁时，当前线程将会休眠，不会参与线程调度，直到获取到锁为止，**获取锁的过程中不响应中断**。
- `lockInterruptibly()`
  - **阻塞式获取**，并且**可中断**，该方法将在以下两种情况之一发生的情况下返回:
    - 获取到锁；
    - 阻塞的线程被其他线程中断。
  - 该方法将在以下两种情况之一发生的情况下返回抛出`InterruptedException`:
    - 在调用该方法时，线程的中断标志位就已经被设为`true`；
    - 在获取锁的过程中，线程被中断，并且锁的获取实现会响应这个中断。
  - 在`InterruptedException`抛出后，当前线程的中断标志位将会被清除。
- `tryLock()`
  - **非阻塞式获取**，`try`就是尝试，无论成功与否，该方法都是立即返回的；
  - 相比前面两种阻塞式获取的方式，该方法是有返回值的，获取锁成功返回`true`，获取锁失败了返回`false`。
- `tryLock(long time, TimeUnit unit)`
  - **带超时机制，并且可中断**；
  - 如果可以获取带锁，则立即返回`true`；
  - 如果获取不到锁，则当前线程将会休眠，不会参与线程调度，直到以下三个条件之一被满足：
    - 当前线程获取到了锁；
    - 其它线程中断了当前线程；
    - 设定的超时时间到了。
  - 该方法将在以下两种情况之一发生的情况下抛出`InterruptedException`:
    - 在调用该方法时，线程的中断标志位已经被设为`true`
    - 在获取锁的过程中，线程被中断，并且锁的获取实现会响应这个中断。
  - 在`InterruptedException`抛出后，当前线程的中断标志位将会被清除。
  - 如果超时时间到了，当前线程还没有获得锁，则会直接返回`false`(注意，**这里并没有抛出超时异常**)。

可见`tryLock(long time, TimeUnit unit)`像是阻塞式与非阻塞式的结合体，即在一定条件下(超时时间内，没有中断发生)阻塞，不满足这个条件则立即返回（非阻塞）。

总结四种锁获取方式的异同点如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/Lock-1.png)

##### 2.锁的释放

锁的释放方法只有一个:

```java
void unlock();
```

值得注意的是，**只有拥有锁的线程才能释放锁，并且必须显式地释放锁**，这一点和离开同步代码块就自动被释放的监视器锁不同。

##### 3.newCondition()方法

`Lock`接口还定义了一个`newCondition`方法:

```java
Condition newCondition();
```

该方法将创建一个绑定在当前`Lock`对象上的`Condition`对象，这说明`Condition`对象和`Lock`对象是对应的，一个`Lock`对象可以创建多个`Condition`对象，它们是一个对多的关系。

### 二、Condition 接口

如上所说，`Lock`接口中定义了`newCondition`方法，它返回一个关联在当前`Lock`对象上的`Condition`对象，下面看这个`Condition`接口的作用。

`Lock`接口的出现是为了拓展现有的监视器锁功能，而`Condition`接口的出现是为了拓展同步代码块中的`wait`，`notify`机制。

##### 1.监视器锁的 wait/notify 机制的弊端

通常情况下，调用`wait`方法，主要是因为一定的条件没有满足，这里把需要满足的事件或条件称作条件谓词。

而另一方面，由`synchronized`的原理可知，所有调用了`wait`方法的线程，都会在同一个监视器锁的`wait set`中等待，这看上去很合理，但是却是该机制的短板所在——所有的线程都等待在同一个`notify`方法上(`notify`方法指`notify()`和`notifyAll()`两个方法，下同)。每一个调用`wait`方法的线程可能等待在不同的条件谓词上，但是有时候即使自己等待的条件并没有满足，线程也有可能被“别的线程的”`notify`方法唤醒，因为大家用的是同一个监视器锁。这就好比一个班上有几个重名的同学(使用相同的监视器锁)，老师喊了这个名字（`notify`方法），结果这几个同学全都站起来了（等待在监视器锁上的线程都被唤醒了）。

这样一来，即使自己被唤醒后抢到了监视器锁，发现其实条件还是不满足，还是得调用`wait`方法挂起，就导致了很多无意义的时间和CPU资源的浪费。

这一切的根源就在于在调用`wait`方法时没有办法来指明究竟是在等待什么样的条件谓词上，因此唤醒时，也不知道该唤醒谁，只能把所有的线程都唤醒了。

因此，最好的方式是，线程在挂起时就指明了在什么样的条件谓词上挂起，同时，在等待的事件发生后，只唤醒等待在这个事件上的线程，而实现了这个思路的就是`Condition`接口。

有了`Condition`接口，就可以在同一个锁上创建不同的唤醒条件，从而在一定条件谓词满足后，有针对性的唤醒特定的线程，而不是将所有等待的线程都唤醒。

##### 2.Condition的 await/signal 机制

`Condition`接口的出现是为了拓展现有的`wait/notify`机制，下面先来看看现有的`wait/notify`机制中的方法：

```java
public class Object {
    public final void wait() throws InterruptedException {
        wait(0);
    }
    public final native void wait(long timeout) throws InterruptedException;
    public final void wait(long timeout, int nanos) throws InterruptedException {
        // 这里省略方法的实现
    }
    public final native void notify();
    public final native void notifyAll();
}
```

`Condition`接口中的方法：

```java
public interface Condition {
    void await() throws InterruptedException;
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;
    
    void awaitUninterruptibly();
    boolean awaitUntil(Date deadline) throws InterruptedException;
    
    void signal();
    void signalAll();
}
```

对比发现，这里存在明显的对应关系：

| Object 类方法                      | Condition 接口方法                      | 区别                       |
| ---------------------------------- | --------------------------------------- | -------------------------- |
| void wait()                        | void await()                            |                            |
| void wait(long timeout)            | long awaitNanos(long nanosTimeout)      | 时间单位，返回值           |
| void wait(long timeout, int nanos) | boolean await(long time, TimeUnit unit) | 时间单位，参数类型，返回值 |
| void notify()                      | void signal()                           |                            |
| void notifyAll()                   | void signalAll()                        |                            |
|                                    | void awaitUninterruptibly()             | Condition独有              |
|                                    | boolean awaitUntil(Date deadline)       | Condition独有              |

它们在接口的规范上都是差不多的，只不过`wait/notify`机制针对的是所有在监视器锁的`wait set`中的线程，而`await/signal`机制针对的是所有等待在该`Condition`上的线程。

需要注意，在接口的规范中，`wait(long timeout)`的时间单位是毫秒(`milliseconds`)，而`awaitNanos(long nanosTimeout)`的时间单位是纳秒(`nanoseconds`)，就这一点而言，`awaitNanos`这个方法名语义上更清晰，并且相对于`wait(long timeout, int nanos)`这个略显鸡肋的方法，`await(long time, TimeUnit unit)`这个方法显得更加直观和有效。

另外值得注意的是，`awaitNanos(long nanosTimeout)`是**有返回值的**，它返回了**剩余等待的时间**；`await(long time, TimeUnit unit)`也是有返回值的，**如果该方法是因为超时时间到了而返回的，则该方法返回false，否则返回true。**

同样是带超时时间的等待，为什么`wait`方式没有返回值，`await`方式有返回值？其实，这里`Condition`接口方法返回值的定义是有一定的目的的。

由于当一个线程从带有超时时间的`wait/await`方法返回时，必然是发生了以下4种情况之一：

1. 其他线程调用了`notify/signal`方法，并且当前线程恰好是被选中来唤醒的那一个；
2. 其他线程调用了`notifyAll/signalAll`方法；
3. 其他线程中断了当前线程；
4. 超时时间到了。

其中第三条会抛出`InterruptedException`，比较容易分辨；除去这个，**当`wait`方法返回后，其实无法区分它是因为超时时间到了返回，还是被`notify`返回的**。但是对于`await`方法，由于它是有返回值的，因此就能够通过返回值来区分，即

- 如果`awaitNanos(long nanosTimeout)`的返回值大于0，说明超时时间还没到，则该返回是由`signal`行为导致的；
- 如果`await(long time, TimeUnit unit)`返回`true`, 说明超时时间还没到，则该返回是由`signal`行为导致的。

可见，`await(long time, TimeUnit unit)`相当于调用`awaitNanos(unit.toNanos(time)) > 0`。因此，**它们的返回值能够说明方法返回的原因**。

`Condition`接口中还有两个在`Object`中找不到对应的方法：

```java
void awaitUninterruptibly();
boolean awaitUntil(Date deadline) throws InterruptedException;
```

前面提到的所有的`wait/await`方法，它们方法的签名中都抛出了`InterruptedException`，说明这些方法在等待锁的过程中都是响应中断的。`awaitUninterruptibly`方法从名字中就可以看出，**它在等待锁的过程中是不响应中断的**，所以没有`InterruptedException`抛出。也就是说，它会一直阻塞，直到`signal/signalAll`被调用。如果在这过程中线程被中断了，它并不响应这个中断，只是在该方法返回的时候，该线程的中断标志位将是`true`， 调用者可以检测这个中断标志位以辅助判断在等待过程中是否发生了中断，以此决定要不要做额外的处理。

`boolean awaitUntil(Date deadline)`和`boolean await(long time, TimeUnit unit)` 其实作用差不多，返回值代表的含义也一样，只不过一个是相对时间，一个是绝对时间，`awaitUntil`方法的参数是`Date`，表示了一个绝对的时间，即截止日期，在这个日期之前，该方法会一直等待，除非被`signal`或者被中断。

至此，`Lock`接口和`Condition`接口的功能就介绍完了。