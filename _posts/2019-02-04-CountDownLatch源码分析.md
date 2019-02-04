---
layout:     post
title:      CountDownLatch源码分析
subtitle:   CountDownLatch
date:       2019-02-04
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# CountDownLatch源码分析

`CountDownLatch`是一个很有用的工具，`latch`是门闩的意思，该工具是为了解决某些操作只能在一组操作全部执行完成后才能执行的情景。`CountDown`是倒数计数，所以`CountDownLatch`的用法通常是设定一个大于0的值，该值即代表需要等待的总任务数，每完成一个任务后，将总任务数减一，直到最后该值为0，说明所有等待的任务都执行完了，“门闩”此时就被打开，后面的任务可以继续执行。

`CountDownLatch`是基于**共享锁**实现的，关于共享锁的内容可参考[AQS源码分析2——共享锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902-%E5%85%B1%E4%BA%AB%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)。

### 一、核心属性

`CountDownLatch`主要是通过AQS的共享锁机制实现的，因此它的核心属性只有一个`sync`，它继承自AQS，同时覆写了`tryAcquireShared`和`tryReleaseShared`方法，以实现具体的共享锁的获取与释放的逻辑。

```java
private final Sync sync;
// CountDownLatch的同步控制，用AQS state代表count
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
	// 用AQS state代表count
    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }
	// >=0表示获取共享锁成功，<0表示获取共享锁失败
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    // true代表本次释放共享锁可以唤醒所有等待的线程
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            // count已经为0，直接返回false。表明已经有其他线程在count=0时，在唤醒所有等待的线程了。
            if (c == 0) 
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc)) // CAS设置
                return nextc == 0; // nextc == 0表示count=0, 则可以唤醒所有等待的线程
        }
    }
}
```

### 二、构造函数

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

在构造函数中，传入了一个不小于0的任务数，由上面`Sync`类的构造函数可知，这个任务数就是AQS `state`的初始值。

### 三、核心方法

`CountDownLatch`最核心的方法只有两个，一个是`countDown`方法，每调用一次，就会将当前的`count`减一，当`count`值为0时，就会唤醒所有等待中的线程；另一个是`await`方法，它有两种形式，一种是阻塞式，一种是带超时机制的形式，该方法用于将当前等待“门闩”开启的线程挂起，直到`count`值为0。

##### 1.countDown()

```java
// 将count减1
public void countDown() {
    sync.releaseShared(1);
}
```

前面说过，`countDown()`方法的目的就是将`count`值减一，并且在`count`值为0时，唤醒所有等待的线程，它内部调用的其实是释放共享锁的操作：

```java
// 释放共享锁
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

该方法由AQS实现，但是`tryReleaseShared`方法由`Sync`类自己实现：

```java
// 尝试释放共享锁
// true代表本次释放共享锁可以唤醒所有等待的线程
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        // count已经为0，直接返回false。表明已经有其他线程在count=0时，在唤醒所有等待的线程了。
        if (c == 0) 
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc)) // CAS设置
            return nextc == 0; // nextc == 0表示count=0, 则可以唤醒所有等待的线程
    }
}
```

该方法的实现很简单，就是获取当前的`state`值，如果已经为0了，直接返回`false`；否则通过CAS操作将`state`值减一，之后返回的是`nextc == 0`，由此可见，**该方法只有在`count`值原来不为0，但是调用后变为0时，才会返回`true`，否则返回`false`**，并且也可以看出，该方法在返回`true`之后，后面如果再次调用，还是会返回`false`。也就是说，**调用该方法只有一种情况会返回`true`，那就是`state`值从大于0变为0值时**，这时也是所有在门闩前的任务都完成了。

在`tryReleaseShared`返回`true`以后，将调用`doReleaseShared`方法唤醒所有等待的线程，这个方法在[AQS源码分析2——共享锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902-%E5%85%B1%E4%BA%AB%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)已经分析过。

值得一提的是，其实这里并不关心`releaseShared`的返回值，而只关心`tryReleaseShared`的返回值，或者只关心`count`到0了没有，这里更像是借了共享锁的“壳”来完成目的，事实上完全可以自己设一个全局变量`count`来实现相同的效果，只不过对这个全局变量的操作也必须使用CAS。

##### 2.await()

与[Condition接口](https://xuanjian1992.top/2019/01/17/Lock%E4%B8%8ECondition%E6%8E%A5%E5%8F%A3%E5%8A%9F%E8%83%BD%E5%88%86%E6%9E%90/)的`await()`方法的语义相同，该方法是阻塞式地等待，并且是响应中断的，只不过它不是在等待`signal`操作，而是在等待`count`值为0：

```java
// 等待count为0
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

可见，`await`方法内部调用的是`acquireSharedInterruptibly`方法：

```java
// 获取共享锁，响应中断
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

对比独占模式下相应的方法：

```java
// 获取独占锁，响应中断
public final void acquireInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
        doAcquireInterruptibly(arg);
}
```

可见，两者用的是同一个框架，只是这里：

- `tryAcquire(arg)` 换成了 `tryAcquireShared(arg)` (子类实现)
- `doAcquireInterruptibly(arg)` 换成了 `doAcquireSharedInterruptibly(arg)` （AQS提供）

先来看看`Sync`子类对于`tryAcquireShared`的实现：

```java
// 尝试获取共享锁，>=0表示获取共享锁成功，<0表示获取共享锁失败
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

**该方法似乎有点挂羊头卖狗肉的感觉——所谓的获取共享锁，事实上并不是什么抢锁的行为，没有任何CAS操作，它就是判断当前的state值是不是0，是就返回1，不是就返回-1。**

在[AQS源码分析2——共享锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902-%E5%85%B1%E4%BA%AB%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)中提到过`tryAcquireShared`方法返回值的含义：

- 如果该值小于0，则代表当前线程获取共享锁失败；
- 如果该值大于0，则代表当前线程获取共享锁成功，并且接下来其他线程尝试获取共享锁的行为很可能成功；
- 如果该值等于0，则代表当前线程获取共享锁成功，但是接下来其他线程尝试获取共享锁的行为会失败。

所以，当该方法的返回值不小于0时，就说明抢锁成功，等待的线程可以继续执行，所对应的就是`count`值已经为0，所有等待的事件都满足了。否则，调用`doAcquireSharedInterruptibly(arg)`将当前线程封装成`Node`，丢到`sync queue`中去阻塞等待：

```java
// 获取共享锁，响应中断
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

在[AQS源码分析2——共享锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902-%E5%85%B1%E4%BA%AB%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)中介绍共享锁的获取时，已经分析过`doAcquireShared`方法，只是它是不抛出`InterruptedException`的，`doAcquireSharedInterruptibly(arg)`是它的可中断版本，可以直接对比一下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CounDownLatch-1.jpg)

可见，它们仅仅是在对待中断的处理方式上有所不同，其他部分都是一样的。

##### 3.await(long timeout, TimeUnit unit)

相较于`await()`方法，`await(long timeout, TimeUnit unit)`提供了超时等待机制：

```java
// 提供超时机制的await
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

// AQS tryAcquireSharedNanos
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}

// 获取共享锁，提供超时机制，响应中断
private boolean doAcquireSharedNanos(int arg, long nanosTimeout)
    throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
            }
            nanosTimeout = deadline - System.nanoTime();
            if (nanosTimeout <= 0L)
                return false;
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

注意，在`tryAcquireSharedNanos`方法中，用到了`doAcquireSharedNanos`的返回值，如果该方法因为超时而退出时，则将返回`false`。由于`await()`方法是阻塞式的，也就是说没有获取到锁是不会退出的，因此它没有返回值，换句话说，如果它正常返回了，则一定是因为获取到了锁而返回； 而`await(long timeout, TimeUnit unit)`由于有了超时机制，它是有返回值的，返回值为`true`则表示获取锁成功，为`false`则表示获取锁失败。**`doAcquireSharedNanos`的这个返回值有助于理解该方法究竟是因为获取到了锁而返回，还是因为超时时间到了而返回。**

至于`doAcquireSharedNanos`的实现细节，由于它和`doAcquireSharedInterruptibly`相比只是多了一个超时机制：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CountDownLatch-2.jpg)

代码本身很简单，这里不再解释。

### 四、实例

接下来学习一个使用`CountDownLatch`的实例，Java的官方源码已经为我们提供了一个使用的示例代码：

```java
/** startSignal启动信号，doneSignal任务完成信号 */
public class Driver {

   private static final int WORKER_COUNT = 10;

   void main() throws InterruptedException {
      CountDownLatch startSignal = new CountDownLatch(1);
      CountDownLatch doneSignal = new CountDownLatch(WORKER_COUNT);

      // 创建、启动线程
      for (int i = 0; i < WORKER_COUNT; i++) {
         new Thread(new Worker(startSignal, doneSignal)).start();
      }

      // 先不让workers运行
      doSomethingElse();
      // 让所有的worker运行
      startSignal.countDown();
      doSomethingElse();
      // 等待所有worker执行完成
      doneSignal.await();
   }

   private void doSomethingElse() {}

   private static class Worker implements Runnable {
      private final CountDownLatch startSignal;
      private final CountDownLatch doneSignal;

      public Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
         this.startSignal = startSignal;
         this.doneSignal = doneSignal;
      }

      @Override
      public void run() {
         try {
            startSignal.await();
            doWork();
            // 执行完成
            doneSignal.countDown();
         } catch (InterruptedException e) {
            e.printStackTrace();
         }
      }

      private void doWork() {}
   }
}
```

在这个例子中，有两个“闸门”，一个是`CountDownLatch startSignal = new CountDownLatch(1)`，它开启后，等待在这个“闸门”上的任务才能开始运行；另一个“闸门”是`CountDownLatch doneSignal = new CountDownLatch(N)`, 它表示等待N个任务都执行完成后，才能继续往下。

`Worker`实现了`Runnable`接口，代表了要执行的任务，在它的`run`方法中，先调用了`startSignal.await()`，等待`startSignal`这一“闸门”开启，闸门开启后，就执行自己的任务，任务完成后再执行`doneSignal.countDown()`，将等待的总任务数减一。

### 五、总结

- `CountDownLatch`相当于一个“门栓”，一个“闸门”，只有它开启了，代码才能继续往下执行。通常情况下，如果当前线程需要等其他线程执行完成后才能执行，就可以使用`CountDownLatch`。
- 使用`CountDownLatch.await`方法阻塞线程，等待一个“闸门”开启。
- 使用`CountDownLatch.countDown`方法减少闸门所等待的任务数。
- `CountDownLatch`基于**共享锁**实现。
- `CountDownLatch`是一次性的，“闸门”开启后，无法再重复使用，如果想重复使用，应该用`CyclicBarrier`。