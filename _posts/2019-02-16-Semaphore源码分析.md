---
layout:     post
title:      Semaphore源码分析
subtitle:   Semaphore
date:       2019-02-16
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# Semaphore源码分析

`Semaphore`(信号量)是常用的并发工具之一，常常用于流量控制。通常情况下，公共的资源常常是有限的，例如数据库的连接数。使用`Semaphore`可以帮助我们有效管理这些有限资源的使用。

`Semaphore`的结构和`ReentrantLock`以及`CountDownLatch`类似，内部采用了公平锁与非公平锁两种实现，基于AQS共享锁机制。

### 一、核心属性

与[`CountDownLatch`](https://xuanjian1992.top/2019/02/04/CountDownLatch%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)类似，`Semaphore`主要是通过AQS的共享锁机制实现的，它的核心属性只有一个`sync`，它继承自AQS：

```java
// Sync属性
private final Sync sync;

// 同步机制的实现
abstract static class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 1192457210091910933L;
	// 初始许可数
    Sync(int permits) {
        setState(permits);
    }

    final int getPermits() {
        return getState();
    }
	// 非公平的尝试获取许可
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
	// 尝试释放许可
    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }
	// 减少许可，不阻塞
    final void reducePermits(int reductions) {
        for (;;) {
            int current = getState();
            int next = current - reductions;
            if (next > current) // underflow
                throw new Error("Permit count underflow");
            if (compareAndSetState(current, next))
                return;
        }
    }

    // 获取剩余所有许可
    final int drainPermits() {
        for (;;) {
            int current = getState();
            if (current == 0 || compareAndSetState(current, 0))
                return current;
        }
    }
}
```

这里的`permits`和`CountDownLatch`的`count`很像，它们最终都将成为AQS中的`state`属性的初始值。

### 二、构造函数

`Semaphore`有两个构造函数：

```java
// permits--许可
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}
// permits--许可，fair--公平策略
public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

默认的构造函数使用的是非公平锁，另一个构造函数通过传入的`fair`参数来决定使用公平锁还是非公平锁，这一点和`ReentrantLock`用的是同样的套路，都是同样的代码框架。

公平锁和非公平锁的定义如下：

```java
// 公平版本
static final class FairSync extends Sync {
    
   FairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            if (hasQueuedPredecessors())
                return -1;
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }
}
// 非公平版本
static final class NonfairSync extends Sync {
    
   NonfairSync(int permits) {
        super(permits);
    }

    protected int tryAcquireShared(int acquires) {
        return nonfairTryAcquireShared(acquires);
    }
}
```

### 三、获取信号量的许可

获取信号量许可的方法有4个：

| `acquire`方法                         | 本质调用                                   |
| ------------------------------------- | ------------------------------------------ |
| `acquire()`                           | `sync.acquireSharedInterruptibly(1)`       |
| `acquire(int permits)`                | `sync.acquireSharedInterruptibly(permits)` |
| `acquireUninterruptibly()`            | `sync.acquireShared(1)`                    |
| `acquireUninterruptibly(int permits)` | `sync.acquireShared(permits);`             |

可见，`acquire()`方法相当于`acquire(1)`，`acquireUninterruptibly`同理，只不过一种响应中断，一种不响应中断，关于AQS的那四个方法在前面的文章中都已经分析过，除了其中的`tryAcquireShared(arg)`由子类实现外，其他的都由AQS实现。

值得注意的是，在[AQS源码分析2——共享锁的获取与释放](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%902-%E5%85%B1%E4%BA%AB%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)中特别提到过`tryAcquireShared`返回值的含义：

- 如果该值小于0，则代表当前线程获取共享锁失败；
- 如果该值大于0，则代表当前线程获取共享锁成功，并且接下来其他线程尝试获取共享锁的行为很可能成功；
- 如果该值等于0，则代表当前线程获取共享锁成功，但是接下来其他线程尝试获取共享锁的行为会失败。

这里的返回值其实代表的是剩余的信号量许可的数量，如果为负值则说明信号量许可不够了。

接下来看下子类对于`tryAcquireShared(arg)`方法的实现。

##### 1.非公平锁实现

```java
// 尝试获取许可
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

// 非公平的尝试获取许可
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState(); // 当前可用数
        int remaining = available - acquires; // 剩余可用数
        if (remaining < 0 ||  // 当前可用数不够
            compareAndSetState(available, remaining)) // 当前可用数足够，则CAS
            return remaining;
    }
}
```

与一般的`tryAcquireShared`逻辑不同，`Semaphore`的`tryAcquireShared`逻辑是一个**自旋操作**，因为`Semaphore`是共享锁，同一时刻可能有多个线程来修改这个值，所以这里必须使用`自旋 + CAS`来避免线程冲突。

该方法退出的唯一条件是**成功的修改了`state`值，并返回`state`的剩余值。如果剩下的信号量许可不够了，则就不需要进行CAS操作，直接返回剩余值。**所以其实`tryAcquireShared`返回的不是当前剩余的信号量许可的值，而是**如果**扣去`acquires`之后，当前**将要**剩余的信号量许可的值，如果这个“将要”剩余的值比0小，则是不会发生扣除操作的。这就好比我要买10个包子，包子铺现在只剩3个了，则将会返回剩余`3 - 10 = -7`个包子，但是事实上包子店并没有将包子卖出去，实际剩余的包子还是3个；此时如果有另一个人来只要买1个包子，则将会返回剩余`3 - 1 = 2`个包子，并且包子店会将一个包子卖出，实际剩余的包子数也是2个。

非公平锁的这种获取信号量许可的逻辑其实和`CountDownLatch`的`countDown`方法很像：

```java
// CountDownLatch
public void countDown() {
    sync.releaseShared(1);
}
```

在`countDown()`的`releaseShared(1)`方法中将调用`tryReleaseShared`：

```java
// CountDownLatch
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

对比`CountDownLatch`的`tryReleaseShared`方法和`Semaphore`的`tryAcquireShared`方法可知，它们的核心逻辑都是减少`state`的值，只不过`CountDownLatch`借用了共享锁的壳，对它而言，减少`state`的值是一种释放共享锁的行为，因为它的目的是将`state`值降为0；而在`Semaphore`中，减少`state`的值是一种获取共享锁的行为，减少成功了，则获取成功。

##### 2.公平锁实现

```java
// 公平的尝试获取许可
protected int tryAcquireShared(int acquires) {
    for (;;) {
        if (hasQueuedPredecessors()) // 是否有节点排队等待在当前节点前面
            return -1;
        int available = getState(); // 当前可用数 
        int remaining = available - acquires; // 剩余可用数
        if (remaining < 0 || // 当前可用数不够
            compareAndSetState(available, remaining)) // 当前可用数足够，则CAS
            return remaining;
    }
}
```

通过对比可以看出，它和`nonfairTryAcquireShared`的唯一的差别在于：

```java
// 是否有节点排队等待在当前节点前面
if (hasQueuedPredecessors())
    return -1;
```

即在获取共享锁之前，先用`hasQueuedPredecessors`方法判断有没有人排在自己前面。关于`hasQueuedPredecessors`方法，在[前面的文章](https://xuanjian1992.top/2019/01/26/ReentrantLock%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)中已经分析过，它就是判断当前节点是否有前驱节点，有的话直接返回获取失败，因为要让前驱节点先去获取锁。（公平锁遵守FIFO规则）

### 四、释放信号量的许可

释放信号量许可的方法有2个：

```java
// 释放信号量的许可
public void release() {
    sync.releaseShared(1);
}

public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
```

可见，`release()` 相当于调用了 `release(1)`，它们最终都调用了`tryReleaseShared(int releases)`方法：

```java
// 尝试释放许可
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState(); // 当前可用数
        int next = current + releases; // 释放后的可用数
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next)) // CAS
            return true;
    }
}
```

与获取信号量许可的逻辑相反，释放信号量许可的逻辑是将得到的信号量许可再归还回去，因此是增加state值的操作。

### 五、工具方法

除了以上获取和释放信号量许可所用到的方法，`Semaphore`还定义了一些其他方法来帮助用户操作信号量：

##### 1.tryAcquire

注意，**这个`tryAcquire`方法不是给`acquire`方法使用的！**上面分析信号量许可的获取时说过，获取信号量的`acquire`方法调用的是AQS的`acquireShared`和`acquireSharedInterruptibly` ，而这两个方法会调用子类的`tryAcquireShared`方法，子类必须实现这个方法。

而这里的`tryAcquire`方法并没有定义在AQS的子类中，即既不在`NonfairSync`，也不在`FairSync`中，对于使用共享锁的AQS的子类，也不需要定义这个方法。事实上它直接定义在`Semaphore`中的。

所以，虽然它和AQS的独占锁的获取逻辑中的`tryAcquire`重名了，但实际上它和AQS的独占锁是没有关系的。

`tryAcquire`和`tryAcquireShared`方法的不同点有两点：

1. 返回值不同：`tryAcquire`返回`boolean`类型，`tryAcquireShared`返回`int`；
2. 不带超时机制的`tryAcquire`方法一定是采用非公平锁模式，带超时机制的`tryAcquire`方法采用公平锁和非公平锁两种模式，而`tryAcquireShared`有公平和非公平两种实现。

理清楚以上几点之后，再来看`tryAcquire`方法的源码，它有四种重载形式：

两种不带超时机制的形式：

```java
// 不带超时机制的形式
public boolean tryAcquire() {
    return sync.nonfairTryAcquireShared(1) >= 0;
}

public boolean tryAcquire(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.nonfairTryAcquireShared(permits) >= 0;
}
```

两种带超时机制的形式：

```java
// 带超时机制的形式
public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}

public boolean tryAcquire(int permits, long timeout, TimeUnit unit) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    return sync.tryAcquireSharedNanos(permits, unit.toNanos(timeout));
}
```

其中，不带超时机制的`tryAcquire`方法实际上调用的就是`nonfairTryAcquireShared(int acquires)`方法，它和非公平锁的`tryAcquireShared`一样，只是`tryAcquireShared`是直接`return nonfairTryAcquireShared(acquires)`，而`tryAcquire`是`return sync.nonfairTryAcquireShared(1) >= 0;`，即直接返回获取锁的操作是否成功。

而带超时机制的`tryAcquire`方法提供了一种超时等待的方式，它本质上调用了AQS的`tryAcquireSharedNanos(int arg, long nanosTimeout)`方法：

```java
public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquireShared(arg) >= 0 ||
        doAcquireSharedNanos(arg, nanosTimeout);
}
```

这个方法在介绍[CountDownLatch源码分析](https://xuanjian1992.top/2019/02/04/CountDownLatch%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)的`await(long timeout, TimeUnit unit)`方法时已经分析过，这里不再展开。

##### 2.reducePermits

`reducePermits`方法用来减少信号量许可的可用总数，这在debug中是很有用的，它与前面介绍的`acquire`方法的不同点在于，即使当前信号量许可的值不足，它也不会导致调用它的线程阻塞等待。只要需要减少的信号量的数量`reductions`大于0，操作最终就会成功，也就是说，即使当前的`reductions`大于现有的信号量的值也没关系，所以该方法可能会导致剩余信号量许可数为负值。

```java
// 减少信号量许可的可用数
protected void reducePermits(int reduction) {
    if (reduction < 0) throw new IllegalArgumentException();
    sync.reducePermits(reduction);
}

final void reducePermits(int reductions) {
    for (;;) {
        int current = getState(); // 当前可用数
        int next = current - reductions; // 剩余的可用数
        if (next > current) // underflow
            throw new Error("Permit count underflow");
        if (compareAndSetState(current, next)) // CAS
            return;
    }
}
```

将它和`nonfairTryAcquireShared`对比一下：

```java
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

可以看出，两者在CAS前的判断条件并不相同，`reducePermits`只要剩余值不比当前值大就可以，而`nonfairTryAcquireShared`必须要保证剩余值不小于0才会执行CAS操作。

##### 3.drainPermits

相比`reducePermits`，`drainPermits`更简单，它直接将剩下的信号量一次性消耗光，并且返回所消耗的信号量，这个方法在debug中也是很有用的：

```java
// 获取并返回剩余的所有许可
public int drainPermits() {
    return sync.drainPermits();
}
final int drainPermits() {
    for (;;) {
        int current = getState();
        if (current == 0 || compareAndSetState(current, 0))
            return current;
    }
}
```

### 六、实例

以上分析了信号量的源码，接下来看一下官方给的一个使用实例：

```java
/** Semaphore示例 有限资源池的并发获取控制 */
public class Pool {
   private static final int MAX_AVAILABLE = 100;
   /** 信号量，资源数100，公平机制 */
   private final Semaphore available = new Semaphore(MAX_AVAILABLE, true);

   /** 获取item */
   public Object getItem() throws InterruptedException {
      // 先获取信号量的一个许可
      available.acquire();
      return getNextAvailableItem();
   }

   /** 放回item */
   public void putItem(Object x) {
      if (markAsUnused(x)) {
         // 资源释放后，释放信号量的一个许可
         available.release();
      }
   }

   /** 资源与使用情况 */
   private Object[] items = new Object[MAX_AVAILABLE];
   private boolean[] used = new boolean[MAX_AVAILABLE];

   /** 获取下一个可用资源 */
   private synchronized Object getNextAvailableItem() {
      for (int i = 0; i < items.length; i++) {
         if (!used[i]) {
            used[i] = true;
            return items[i];
         }
      }
      return null;
   }

   /** 标记该资源为未使用状态 */
   private synchronized boolean markAsUnused(Object x) {
      for (int i = 0; i < items.length; i++) {
         if (items[i] == x) {
            if (used[i]) {
               used[i] = false;
               return true;
            } else {
               return false;
            }
         }
      }
      return false;
   }
}
```

这个例子用`items`数组代表可用的资源，用`used`数组来标记资源的使用情况，`used[i]`的值为`true`，则代表`items[i]`这个资源已经被使用。

(1) 获取一个可用资源 
调用`getItem()`来获取资源，在该方法中会先调用`available.acquire()`方法请求一个信号量许可，注意，这里如果当前信号量数不够时，是会阻塞等待的；当成功地获取了一个信号量之后，将会调用`getNextAvailableItem`方法，返回一个可用的资源。

(2) 释放一个资源
调用`putItem(Object x)`来释放资源，在该方法中会先调用`markAsUnused(Object item)`将需要释放的资源标记成可用状态（即将`used`数组中对应的位置标记成`false`）, 如果释放成功，就调用`available.release()`来释放一个信号量。

### 总结

`Semaphore`是一个有效的流量控制工具，它基于AQS共享锁实现。工程中常常用它来控制对有限资源的访问。每次使用资源前，先申请一个信号量许可，如果资源数不够，就会阻塞等待；每次释放资源后，就释放一个信号量许可。