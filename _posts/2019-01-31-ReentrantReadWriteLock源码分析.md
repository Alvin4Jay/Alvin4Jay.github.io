---
layout:     post
title:      ReentrantReadWriteLock源码分析
subtitle:   ReentrantReadWriteLock
date:       2019-01-31
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# ReentrantReadWriteLock源码分析

### 一、ReentrantReadWriteLock简介

重入锁`ReentrantLock`是排他锁，排他锁在同一时刻仅有一个线程可以进行访问，但是在大多数场景下，大部分时间都是提供读服务，而写服务占有的时间较少。然而读服务不存在数据竞争问题，如果一个线程在读时，禁止其他线程读势必会导致性能降低。所以JDK就提供了读写锁(`ReadWriteLock/ReentrantReadWriteLock`)。

读写锁维护着一对锁，一个读锁和一个写锁。通过分离读锁和写锁，使得并发性比一般的排他锁有了较大的提升：在同一时间可以允许多个读线程同时访问，但是在写线程访问时，所有读线程和写线程都会被阻塞。

读写锁的主要特性：

- 公平性：支持公平性和非公平性；

- 重入性：支持重入。读写锁最多支持65535个递归写入锁和65535个递归读取锁；

- 锁降级：遵循获取写锁、获取读锁、再释放写锁的次序，写锁能够降级成为读锁。

读写锁`ReentrantReadWriteLock`实现接口`ReadWriteLock`，该接口维护了一对相关的锁，一个用于只读操作，另一个用于写入操作。只要没有 `writer`(没有线程持有写锁)，读锁可以由多个 `reader` 线程同时保持。写入锁是独占的。

```java
// 读写锁接口
public interface ReadWriteLock {
    Lock readLock(); // 获取读锁
    Lock writeLock(); // 获取写锁
}
```

`ReadWriteLock`定义了两个方法。`readLock()`返回用于读操作的锁，`writeLock()`返回用于写操作的锁。

`ReentrantReadWriteLock`定义如下：

```java
/** 内部类  读锁 */
private final ReentrantReadWriteLock.ReadLock readerLock;
/** 内部类  写锁 */
private final ReentrantReadWriteLock.WriteLock writerLock;

// ReentrantReadWriteLock同步机制的实现
final Sync sync;

/** 使用默认（非公平）的排序策略创建一个新的 ReentrantReadWriteLock */
public ReentrantReadWriteLock() {
    this(false);
}

/** 使用给定的公平策略创建一个新的 ReentrantReadWriteLock */
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

/** 返回用于写入操作的锁 */
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
/** 返回用于读取操作的锁 */
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

// ReentrantReadWriteLock同步机制的实现
abstract static class Sync extends AbstractQueuedSynchronizer {
    /** 省略其余源代码 */
}

// 写锁定义
public static class WriteLock implements Lock, java.io.Serializable{
    /** 省略其余源代码 */
}

// 读锁定义
public static class ReadLock implements Lock, java.io.Serializable {
    /** 省略其余源代码 */
}
```

`ReentrantReadWriteLock`与`ReentrantLock`一样，其锁主体依然是`Sync`，它的读锁、写锁都是依靠`Sync`来实现的。所以`ReentrantReadWriteLock`实际上只有一个锁，只是在获取读取锁和写入锁的方式上不一样而已，它的读写锁其实就是两个类：`ReadLock、writeLock`，这两个类都是`Lock`实现。

在`ReentrantLock`中使用一个`int`类型的`state`来表示同步状态，该值表示锁被一个线程重复获取的次数。但是读写锁`ReentrantReadWriteLock`内部维护着一对锁，需要用一个变量维护多种状态。所以读写锁采用“按位切割使用”的方式来维护这个变量，将其切分为两部分，高16为表示读锁获取次数，低16为表示写锁获取次数。分割之后，读写锁是如何迅速确定读锁和写锁的状态呢？通过**位运算**。假如当前同步状态为S，那么写状态等于 `S & 0x0000FFFF`（将高16位全部抹去），读状态等于`S >>> 16`(无符号补0右移16位)。代码如下：

```java
// 读锁使用state高2字节，写锁使用state低2字节
static final int SHARED_SHIFT   = 16;
// 操作读锁次数时使用的常量 00000000 00000001 00000000 00000000
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 读锁或写锁的最大重入次数 00000000 00000000 11111111 11111111
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
// 用于计算写锁的持有数量   00000000 00000000 11111111 11111111
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

static int sharedCount(int c)    { return c >>> SHARED_SHIFT; } // 计算读锁的获取次数
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; } // 计算写锁的获取次数
```

### 二、写锁的获取与释放

写锁是一个支持重入的独占锁。

##### 写锁的获取

写锁的获取通过`WriteLock`的`lock()`方法完成:

```java
// 获取写锁
public void lock() {
    sync.acquire(1);
}

// AQS方法acquire
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

最终会调用`tryAcquire(int arg)`，该方法在内部类`Sync`中实现：

```java
// 获取写锁
protected final boolean tryAcquire(int acquires) {
  
     // Walkthrough:
     // 1. If read count nonzero or write count nonzero
     //   and owner is a different thread, fail.
     // 2. If count would saturate, fail. (This can only
     //    happen if count is already nonzero.)
     // 3. Otherwise, this thread is eligible for lock if
     //    it is either a reentrant acquire or
     //    queue policy allows it. If so, update state
     //    and set owner.
     //
    Thread current = Thread.currentThread();
    int c = getState(); // 同步器状态state，包含读锁和写锁持有数(获取次数)
    // 写锁持有数
    int w = exclusiveCount(c);
    if (c != 0) {
        // (Note: if c != 0 and w == 0 then shared count != 0) 说明读锁持有数不为0
        // 1.写锁持有数为0，读锁持有数不为0，直接返回失败，为了保证写操作的可见性 2.写锁持有数不为0，但持有线程不是当前线程，直接锁获取失败
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 写锁重入
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // Reentrant acquire
        setState(c + acquires);
        return true;
    }
    // 判断是否需要排队等待
    if (writerShouldBlock() ||
            // 不需要，则CAS尝试(存在并发失败的可能)
        !compareAndSetState(c, c + acquires))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

该方法和`ReentrantLock`的`tryAcquire(int arg)`大致一样，在判断重入时增加了一项条件：读锁是否存在。因为要确保写锁的操作对读锁是可见的，如果在存在读锁的情况下允许获取写锁，那么那些已经获取读锁的其他线程可能就无法感知当前写线程的操作。因此只有等读锁完全释放后，写锁才能够被当前线程所获取，一旦写锁获取了，所有其他读、写线程均会被阻塞。

##### 写锁的尝试获取

尝试获取写锁通过`WriteLock.tryLock()`完成:

```java
// 尝试获取写锁，不考虑公平策略
public boolean tryLock( ) {
    return sync.tryWriteLock();
}

// Sync内定义的tryWriteLock
final boolean tryWriteLock() {
    Thread current = Thread.currentThread();
    int c = getState(); // 同步器状态
    if (c != 0) {
        int w = exclusiveCount(c); // 写锁持有数
        // 存在线程持有读锁，无线程持有写锁或者当前线程不是已持有写锁的线程，返回失败
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        // 已达到最大重入数
        if (w == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
    }
    // c==0 直接CAS，或者C!=0，判断是否已达到最大重入数，否则进行CAS操作
    if (!compareAndSetState(c, c + 1))
        return false;
    setExclusiveOwnerThread(current);
    return true;
}
```

尝试获取写锁，不会考虑公平性(`writerShouldBlock()`)，只要写锁能够获取，则直接获取。

##### 写锁的释放

获取的写锁用完了则需要释放，`WriteLock`提供了`unlock()`方法释放写锁：

```java
// 释放写锁
public void unlock() {
    sync.release(1);
}

// AQS方法release
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

写锁的释放最终还是会调用AQS的模板方法`release(int arg)`方法，该方法首先调用`tryRelease(int arg)`方法尝试释放锁，`tryRelease(int arg)`方法定义在读写锁内部类`Sync`中，如下所示：

```java
// 释放写锁
protected final boolean tryRelease(int releases) {
    // 检查持有锁的线程是否是当前线程
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    int nextc = getState() - releases; // 释放写锁后的state
    // 判断写锁持有数是否为0
    boolean free = exclusiveCount(nextc) == 0;
    if (free)
        setExclusiveOwnerThread(null);
    setState(nextc);
    return free;
}
```

写锁释放锁的整个过程和独占锁`ReentrantLock`相似，每次释放均是减少写状态，当写状态为0时表示写锁已经完全释放了，从而等待的其他线程可以继续访问读写锁，获取同步状态，同时此次写线程的修改对后续的线程可见。

### 三、读锁的获取与释放

读锁是一个可重入的共享锁，它能够被多个读线程同时持有，在没有其他写线程访问时，读锁总是能获取成功。

##### 读锁的获取

读锁的获取可以通过`ReadLock`的`lock()`方法完成：

```java
// 获取读锁
public void lock() {
    sync.acquireShared(1);
}

```

`Sync`的`acquireShared(int arg)`定义在AQS中：

```java
// AQS方法acquireShared
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

`tryAcqurireShared(int arg)`尝试获取读同步状态(读锁)，该方法主要用于获取共享式同步状态，获取成功返回 >= 0的结果，否则返回 < 0 的结果。

```java
// 获取读锁，参数unused未使用
protected final int tryAcquireShared(int unused) {

    // Walkthrough:
    // 1. If write lock held by another thread, fail.
    // 2. Otherwise, this thread is eligible for
    //    lock wrt state, so ask if it should block
    //    because of queue policy. If not, try
    //    to grant by CASing state and updating count.
    //    Note that step does not check for reentrant
    //    acquires, which is postponed to full version
    //    to avoid having to check hold count in
    //    the more typical non-reentrant case.
    // 3. If step 2 fails either because thread
    //    apparently not eligible or CAS fails or count
    //    saturated, chain to version with full retry loop.
    
    Thread current = Thread.currentThread();
    int c = getState(); // 同步器状态state
    // 判断是否锁降级
    if (exclusiveCount(c) != 0 &&
        getExclusiveOwnerThread() != current)
        // 存在线程持有写锁，但该线程不是当前线程，返回-1，排队等待
        return -1;
    // 读锁持有数
    int r = sharedCount(c);
    // 允许获取读锁(不阻塞)，且CAS成功(获取读锁成功)
    if (!readerShouldBlock() &&
        r < MAX_COUNT &&
        // 读锁持有数加1
        compareAndSetState(c, c + SHARED_UNIT)) {
        if (r == 0) { // 之前无读线程获取读锁，则第一个获取读锁的读线程记录在firstReader变量上
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) { // firstReader重入
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter; // 上一个获取读锁的线程对应的Counter，不包含第一个firstReader
            if (rh == null || rh.tid != getThreadId(current))  // rh为null或上一个获取读锁的线程不是当前线程
                cachedHoldCounter = rh = readHolds.get(); // 从ThreadLocal查找，可能初始化；更新cachedHoldCounter
            // 为什么要 count == 0 时进行 ThreadLocal.set? 因为tryReleaseShared方法中，
            // 当 count = 0 时, 进行了ThreadLocal.remove操作
            else if (rh.count == 0) // 上一个获取读锁的线程就是当前线程，并且count=0。
                readHolds.set(rh); // 如果rh.count == 0，需要重新ThreadLocal.set
            rh.count++;
        }
        return 1; // 这里返回1，表示共享读锁可以再次获取
    }
    return fullTryAcquireShared(current);
}
```

读锁获取的过程相对于写锁而言会稍微复杂，整个过程如下：

1. 因为存在锁降级情况，如果存在写锁且锁的持有者不是当前线程则直接返回失败，否则继续；
2. 依据公平性原则，判断读锁是否需要阻塞，读锁持有线程数小于最大值(65535)，且设置锁状态成功，执行设置锁持有数的代码(对于`HoldCounter`下面再阐述)，并返回1。如果不满足改条件，执行`fullTryAcquireShared()`。

```java
// 获取读锁的完整版方法。1.处理tryAcquireShared() CAS失败 
// 2.处理读锁重入的情况(根据公平策略，包括阻塞和不阻塞时的重入判断)
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    // 自旋，是防止CAS失败
    for (;;) {
        int c = getState();
        // 存在线程持有写锁，如持有线程不是当前线程，返回失败(锁降级判断)
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
            // else we hold the exclusive lock; blocking here
            // would cause deadlock. 写锁允许降级为读锁，先获取写锁，再获取读锁，再释放写锁
        } else if (readerShouldBlock()) {
            // 不存在线程持有写锁，若需要阻塞当前获取读锁的线程，先判断是否重入
            // Make sure we're not acquiring read lock reentrantly
            if (firstReader == current) { // 若是firstReader读锁的重入获取，则直接进行下面的 CAS 操作
                // assert firstReaderHoldCount > 0;
                // 重入，直接进行下面的CAS操作
            } else {
                // 判断当前线程是否已持有读锁
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        // 从ThreadLocal查找，可能之前未获取读锁
                        rh = readHolds.get();
                        // 当前线程未持有读锁，则直接通过ThreadLocal移除与该线程绑定的HoldCounter
                        if (rh.count == 0)
                            readHolds.remove();
                    }
                }
                // 判断重入时，当前线程未持有读锁，则直接返回失败-1(阻塞排队)，不进行下面CAS操作
                if (rh.count == 0)
                    return -1;
            }
        }

        // 1.写锁持有数不为0，为锁降级
        // 2.写锁持有数为0，当前线程不需要阻塞
        // 3.写锁持有数为0，当前线程需要阻塞，判断是重入的情况
        // 判断重入数是否达到最大值
        if (sharedCount(c) == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        // CAS update state 更新读锁持有数
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            // 逻辑同tryAcquireShared
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
                // 更新cachedHoldCounter
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

`fullTryAcquireShared(Thread current)`会根据“是否需要阻塞等待”，“读取锁的共享计数是否超过限制”等等进行处理。如果不需要阻塞等待，并且锁的共享计数没有超过限制，则通过CAS尝试获取锁，并返回1。

##### 读锁的尝试获取

尝试获取读锁，通过`ReadLock.tryLock()`完成:

```java
// 尝试获取读锁
public boolean tryLock() {
    return sync.tryReadLock();
}

// Sync定义的方法tryReadLock
final boolean tryReadLock() {
    Thread current = Thread.currentThread();
    // 自旋，防止CAS失败
    for (;;) {
        int c = getState();
        // 不是锁降级，返回失败
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return false;
        // 读锁持有数
        int r = sharedCount(c);
        if (r == MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            // 同tryAcquireShared、fullTryAcquireShared逻辑
            if (r == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return true;
        }
    }
}
```

尝试获取读锁的过程，不会考虑公平策略(`readerShouldBlock()`)，只要读锁可以获取，直接CAS获取。

##### 读锁的释放

与写锁相同，读锁也提供了`unlock()`释放读锁：

```java
// 释放读锁
public void unlock() {
    sync.releaseShared(1);
}
```

`unlcok()`方法内部使用`Sync`的`releaseShared(int arg)`方法，该方法定义在AQS中：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

调用`tryReleaseShared(int arg)`尝试释放读锁，该方法定义在读写锁的`Sync`内部类中：

```java
// 释放读锁，参数unused未使用
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    // 当前线程是firstReader
    if (firstReader == current) {
        // assert firstReaderHoldCount > 0;
        // 如果持有数为1，设置firstReader为null；否则firstReaderHoldCount--
        if (firstReaderHoldCount == 1) 
            firstReader = null;
        else
            firstReaderHoldCount--;
    } else {
        // 获取与该线程绑定的HoldCounter，判断该线程读锁持有数
        HoldCounter rh = cachedHoldCounter;
        // 上一个获取读锁的线程不是当前线程
        if (rh == null || rh.tid != getThreadId(current))
            // 通过ThreadLocal查找，可能初始化
            rh = readHolds.get();
        int count = rh.count;
        if (count <= 1) {
            // 持有数变为0，则删除与当前线程绑定的HoldCounter变量副本
            readHolds.remove();
            if (count <= 0) // 释放读锁时，若线程原本未持有读锁，直接抛出该异常
                throw unmatchedUnlockException();
        }
        --rh.count; // 持有数减1
    }
    // 自旋CAS
    for (;;) {
        int c = getState();
        // 读锁持有数减1
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc)) // CAS
            // nextc为0，说明读锁、写锁持有数都为0，则等待获取写锁的线程可以继续获取写锁
            return nextc == 0; 
    }
}

// 释放读锁时，若线程原本未持有读锁，直接抛出该异常
private IllegalMonitorStateException unmatchedUnlockException() {
    return new IllegalMonitorStateException(
        "attempt to unlock read lock, not locked by current thread");
}
```

##### HoldCounter

在读线程获取锁和释放锁的过程中，一直都可以看到一个变量`rh (HoldCounter)`，该变量在读锁中扮演着非常重要的作用。

读锁的内在机制其实就是一个共享锁，为了更好理解`HoldCounter `，暂且认为它不是一个锁的概率，而相当于一个计数器。一次共享锁的操作就相当于在该计数器的操作。获取共享锁，则该计数器 + 1，释放共享锁，该计数器 – 1。只有当线程获取共享锁后才能对共享锁进行释放、重入操作。所以`HoldCounter`的作用就是当前线程持有共享锁(读锁)的数量，这个数量必须要与线程绑定在一起，否则操作其他线程锁就会抛出异常。先看`HoldCounter`的定义：

```java
// 与线程绑定的读锁持有计数
static final class HoldCounter {
    int count = 0;
    // Use id, not reference, to avoid garbage retention
    // 线程会引用其HoldCounter，如果该HoldCounter再引用线程，就会造成循环引用、难以GC
    final long tid = getThreadId(Thread.currentThread());
}
```

`HoldCounter`定义非常简单，就是一个计数器`count`和线程`id--tid`两个变量。按照这个意思，`HoldCounter`是需要和某个线程进行绑定了。如果要将一个对象和线程绑定仅仅有`tid`是不够的，而且从上面的代码可以看到，`HoldCounter`仅仅只是记录了`tid`，根本起不到绑定线程的作用。那么怎么实现呢？答案是`ThreadLocal`，定义如下：

```java
static final class ThreadLocalHoldCounter
    extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

通过上面代码`HoldCounter`就可以与线程进行绑定了。故而，`HoldCounter`应该就是绑定线程上的一个计数器，而`ThradLocalHoldCounter`则是线程绑定的`ThreadLocal`。从上面可以看到`ThreadLocal`将`HoldCounter`绑定到当前线程上，同时`HoldCounter`也持有线程`id`，这样在释放锁的时候才能知道`ReadWriteLock`里面缓存的上一个读取线程（`cachedHoldCounter`）是否是当前线程。这样做的好处是可以减少`ThreadLocal.get()`的次数，因为这也是一个耗时操作。需要说明的是这样`HoldCounter`绑定线程`id`而不绑定线程对象的原因是避免`HoldCounter`和`ThreadLocal`互相绑定而GC难以释放它们（尽管GC能够智能的发现这种引用而回收它们，但是这需要一定的代价），所以其实这样做只是为了帮助GC快速回收对象而已。

明白了`HoldCounter`的作用后，再看获取读锁的代码段:

```java
if (r == 0) { // 之前无读线程获取读锁，则第一个获取读锁的读线程记录在firstReader变量上
    firstReader = current;
    firstReaderHoldCount = 1;
} else if (firstReader == current) { // firstReader重入
    firstReaderHoldCount++;
} else {
    HoldCounter rh = cachedHoldCounter; // 上一个获取读锁的线程对应的Counter，不包含第一个firstReader
    if (rh == null || rh.tid != getThreadId(current))  // rh为null或上一个获取读锁的线程不是当前线程
        cachedHoldCounter = rh = readHolds.get(); // 从ThreadLocal查找，可能初始化；更新cachedHoldCounter
    // 为什么要 count == 0 时进行 ThreadLocal.set? 因为tryReleaseShared方法中，
    // 当 count = 0 时, 进行了ThreadLocal.remove操作
    else if (rh.count == 0) // 上一个获取读锁的线程就是当前线程，并且count=0。
        readHolds.set(rh); // 如果rh.count == 0，需要重新ThreadLocal.set
    rh.count++;
}
```

这段代码涉及了几个变量：`firstReader、firstReaderHoldCount、cachedHoldCounter、readHolds`。这里先理清楚这几个变量：

```java
private transient Thread firstReader = null;  // 仅表示第一个获取读锁的线程
private transient int firstReaderHoldCount; // 第一个获取读锁的线程的读锁持有数
private transient HoldCounter cachedHoldCounter; // 上一个获取读锁的线程的HoldCounter，不包含firstReader
private transient ThreadLocalHoldCounter readHolds; // ThreadLocal，将HoldCounter绑定到相应的线程，不包含firstReader
```

`firstReader`为第一个获取读锁的线程，`firstReaderHoldCount`为第一个获取读锁的线程的重入数，`cachedHoldCounter`为上一个获取读锁的线程的`HoldCounter`，不包含`firstReader`。

理清楚上面所有的变量后，`HoldCounter`以及上面代码的逻辑也就很好理解了。这里解释下为何要引入`firstRead、firstReaderHoldCount`。这是为了一个**效率问题**，`firstReader`是不会放入到`readHolds(ThreadLocalHoldCounter)`中的，**如果读锁仅有一个的情况下就会避免查找`readHolds`。**

### 四、锁降级

本文开头已阐述了读写锁有一个特性就是**锁降级**，锁降级意味着写锁是可以降级为读锁的，但是需要遵循先获取写锁、获取读锁、再释放写锁的顺序。注意如果当前线程先获取写锁，然后释放写锁，再获取读锁这个过程不能称之为锁降级，锁降级一定要遵循那个顺序。

在获取读锁的方法`tryAcquireShared(int unused)`中，有一段代码就是来判读锁降级的：

```java
Thread current = Thread.currentThread();
int c = getState();
// 判断是否锁降级
if (exclusiveCount(c) != 0 &&
    getExclusiveOwnerThread() != current)
    // 存在线程持有写锁，但该线程不是当前线程，返回-1，排队等待
    return -1;
// 读锁持有数
int r = sharedCount(c);
```

锁降级中读锁的获取是否有必要？肯定是必要的。试想，假如当前线程A不获取读锁而是直接释放了写锁，这个时候另外一个线程B获取了写锁，那么这个线程B对数据的修改是不会对当前线程A可见的(**可见性**)。如果获取了读锁，则线程B在获取写锁过程中判断如果有读锁还没有释放则会被阻塞，只有当前线程A释放读锁后，线程B才会获取写锁成功。

##### 示例1(JDK)

```java
// 锁降级，先获取写锁、获取读锁、再释放写锁
public class CacheData {

    Object data; // 缓存的数据
    volatile boolean cacheValid; // 缓存是否有效
    final ReentrantReadWriteLock rwl = new ReentrantReadWriteLock();

    void processCacheDate(){
        rwl.readLock().lock(); // 1. 先获取 readLock
        if(!cacheValid){       // 2. 发现数据无效
            // Must release read lock before acquiring write lock
            rwl.readLock().unlock(); // 3. 释放 readLock
            rwl.writeLock().lock();  // 4. 获取 writeLock (1)
            try{
                // Recheck state because another thread might have
                // acquired write lock and changed state before we did
                if(!cacheValid){            // 5. 重新确认数据是否真的无效
                    // data = ...           // 6. 进行数据 data 的重新赋值
                    cacheValid = true;      // 7. 重置标签 cacheValid
                }
                // Downgrade by acquiring read lock before releasing write lock
                rwl.readLock().lock();      // 8. 在获取 writeLock 的前提下, 再次获取 readLock (2)
            }finally{
                rwl.writeLock().unlock(); // Unlock write, still hold read // 9. 释放 writeLock, 完成锁的降级 (3)
            }
        }

        try{
            use(data); // 数据处理
        }finally{
            rwl.readLock().unlock(); // 10. 释放 readLock (4)
        }
    }

}
```



##### 示例2

锁降级：从写锁变成读锁；锁升级：从读锁变成写锁。读锁是可以被多线程共享的，写锁是单线程独占的。也就是说写锁的并发限制比读锁高，这可能就是升级/降级名称的来源。

如下代码会产生**死锁**，因为同一个线程中，在没有释放读锁的情况下，就去申请写锁，这属于**锁升级，`ReentrantReadWriteLock`是不支持**的。

```java
ReadWriteLock rtLock = new ReentrantReadWriteLock();
rtLock.readLock().lock();
System.out.println("get readLock.");
rtLock.writeLock().lock(); // 阻塞
System.out.println("blocking");
```

**`ReentrantReadWriteLock`支持锁降级，**如下代码不会产生死锁。

```java
ReadWriteLock rtLock = new ReentrantReadWriteLock();
rtLock.writeLock().lock();
System.out.println("get writeLock");
 
rtLock.readLock().lock();
System.out.println("get readLock"); 

rtLock.writeLock().unlock();
System.out.println("release writeLock"); // 释放写锁，写锁降级为读锁
```

##### 示例3

```java
// 锁降级测试
public class ReadWriteLockTest {
	// 缓存的数据是否有效
    private volatile boolean cacheValid = false;
	// 数据
    private int currentValue = 0;
	// 读写锁
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
	// 读锁
    private Lock readLock = lock.readLock();
	// 写锁
    private Lock writeLock = lock.writeLock();

    /** 测试锁降级 */
    @Test
    public void testLockDowngrading() throws InterruptedException {
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch end = new CountDownLatch(2);
        ThreadPoolExecutor executor = new ThreadPoolExecutor(10, 10, 100, TimeUnit.SECONDS, new ArrayBlockingQueue
                <Runnable>(10));
        for (int i = 0; i < 2; i ++){
            int finalI = i;
            executor.execute(new Thread(new Runnable() {
                @Override
                public void run() {
                    Thread.currentThread().setName("thread-" + finalI);
                    try {
                        start.await();
                        TimeUnit.SECONDS.sleep(finalI);
                        System.out.println("after sleep " + finalI + " seconds, excute " + Thread.currentThread().getName());
                        cacheValid = false;
                        processCachedData(finalI); // 无锁降级
                       	// processCachedDataDownGrading(finalI); // 锁降级
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }finally {
                        end.countDown();
                    }

                }
            }));
        }
        start.countDown();
        end.await();
    }

    /** 锁降级过程 */
    private void processCachedDataDownGrading(int num){
        readLock.lock();
        if(!cacheValid){ // 发现数据无效
            // 必须先释放读锁，再获取写锁 (1)
            readLock.unlock();
            writeLock.lock();
            try{
                // 在更新数据之前做二次检查
                if(!cacheValid){
                    System.out.println(Thread.currentThread().getName() + " has updated!");
                    // 将数据更新为和线程值相同，以便验证数据
                    currentValue = num;
                    cacheValid = true;
                    readLock.lock(); // 获取读锁 (2)
                }
            }finally {
                writeLock.unlock(); // 释放写锁 (3)
            }

        }

        try{
            // 模拟5秒的处理时间，并打印出当前值，在这个过程中cacheValid可能被其他线程修改，
            // 锁降级保证其他线程获取写锁会被阻塞，数据不被改变。
            TimeUnit.SECONDS.sleep(5);
            System.out.println(Thread.currentThread().getName() + ": " +  currentValue);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            if(lock.getReadHoldCount() > 0){
                readLock.unlock();
            }
        }
    }

    /** 无锁降级的过程 */
    private void processCachedData(int num){
        readLock.lock();
        if(!cacheValid){ // 发现数据无效
            readLock.unlock(); // 必须先释放读锁，再获取写锁 (1)
            writeLock.lock();
            try{
                if(!cacheValid){ // 在更新数据之前做二次检查
                    System.out.println(Thread.currentThread().getName() + " has updated!");
                    // 将数据更新为和线程值相同，以便验证数据
                    currentValue = num;
                    cacheValid = true;
                }
            }finally {
                writeLock.unlock(); // 释放写锁 (2)
            }
        }
        try{
            // 模拟5秒的处理时间，并打印出当前值，在这个过程中cacheValid可能被其他线程修改，
            // 无锁降级过程，其他线程此时可能获取写锁，并更改缓存的数据，导致该线程处理数据时
            // 无法看到数据的变更，或者无法保证原子性。
            TimeUnit.SECONDS.sleep(5);
            System.out.println(Thread.currentThread().getName() + ": " +  currentValue);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            if(lock.getReadHoldCount() > 0){
                readLock.unlock();
            }
        }
    }
}
```

在上述的测试代码中，有两个方法，一个是`processCachedDataDownGrading`，该方法模拟锁降级的过程，另一个是`processCachedData`，模拟无锁降级的过程。

在主测试代码中，起了两个线程，每个线程执行前先将`cacheValid`置为`false`，同时为了模拟处理数据(5秒)过程中**外部线程想要执行的数据变更**，两个线程的实际执行行为相差1秒。

(1) 如果两个线程中执行的方法是`processCachedData`(无锁降级过程)，那么输出为：

```java
after sleep 0 seconds, excute thread-0
thread-0 has updated!
after sleep 1 seconds, excute thread-1
thread-1 has updated!
thread-0: 1
thread-1: 1
```

从输出看，线程0马上执行（不是指`run`前，指`run`中想测试的内容），线程1过了1秒执行，两个线程中的数据均被改变，但是最终两个线程中的值均为1（实际期望线程0中的数据应该为0），导致这个结果的原因就是在线程0处理数据（`sleep`5秒）的过程中，线程1获取了写锁并更新了数据，从而线程0的数据被更新。

(2) 如果两个线程中执行的方法是`processCachedDataDownGrading`(锁降级过程)，那么输出为：

```java
after sleep 0 seconds, excute thread-0
thread-0 has updated!
after sleep 1 seconds, excute thread-1
thread-0: 0
thread-1 has updated!
thread-1: 1
```

从输出看，线程0马上执行，线程1过了1秒执行，两个线程中的数据均被改变，但是最终两个线程中的值分别为0和1，符合期望，因为这个过程是**锁降级**的过程，线程0在更新数据之后，释放写锁之前，获取了读锁，并在处理完数据之后才将其释放，因此线程1在线程0处理数据时获取写锁被阻塞，从thread-1 has updated! 语句输出在thread-0: 0 之后也可看出，从而保证了数据0的数据没有问题。

### 五、其他

##### 内部类 Sync类其他属性和方法

```java
/// 构造器
Sync() {
    readHolds = new ThreadLocalHoldCounter();
    // readHolds为非volatile变量，state为volatile变量，
    // 通过setState()操作可以保证，setState()之前的操作都强制刷新到主内存，保证了readHolds的内存可见性
    setState(getState()); // ensures visibility of readHolds
}

/// 公平性策略
// 获取读锁的线程是否阻塞
abstract boolean readerShouldBlock();
// 获取写锁的线程是否阻塞
abstract boolean writerShouldBlock();

// 判断写锁是否由当前线程持有
protected final boolean isHeldExclusively() {
    return getExclusiveOwnerThread() == Thread.currentThread();
}

// 创建Condition对象
final ConditionObject newCondition() {
    return new ConditionObject();
}

/// 监控方法
// 获取写锁持有线程
final Thread getOwner() {
    return ((exclusiveCount(getState()) == 0) ? null : getExclusiveOwnerThread());
}

// 读锁持有数
final int getReadLockCount() {
    return sharedCount(getState());
}

// 判断写锁是否被任何线程持有
final boolean isWriteLocked() {
    return exclusiveCount(getState()) != 0;
}

// 获取当前线程写锁的持有数
final int getWriteHoldCount() {
    return isHeldExclusively() ? exclusiveCount(getState()) : 0;
}

// 获取当前线程读锁的持有数
final int getReadHoldCount() {
    if (getReadLockCount() == 0)
        return 0;

    Thread current = Thread.currentThread();
    // firstReader是当前线程
    if (firstReader == current)
        return firstReaderHoldCount;
    // cachedHoldCounter对应的线程是当前线程
    HoldCounter rh = cachedHoldCounter;
    if (rh != null && rh.tid == getThreadId(current))
        return rh.count;
    // 通过ThreadLocal查找当前线程绑定的Counter
    int count = readHolds.get().count;
    if (count == 0) readHolds.remove();
    return count;
}

/**  反序列化 */
private void readObject(java.io.ObjectInputStream s)
    throws java.io.IOException, ClassNotFoundException {
    s.defaultReadObject();
    readHolds = new ThreadLocalHoldCounter();
    setState(0); // reset to unlocked state
}

// 获取锁状态
final int getCount() { return getState(); }
```

##### 内部类 NonfairSync

```java
// Sync的非公平实现版本
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = -8159625535654395037L;
    // 非公平策略下，在获取写锁时，获取线程总是可以尝试抢占写锁
    final boolean writerShouldBlock() {
        return false; // writers can always barge
    }
    // 非公平策略下，如果等待队列中第一个节点线程是获取写锁的线程，则此次获取读锁的线程必须等待
    final boolean readerShouldBlock() {
        return apparentlyFirstQueuedIsExclusive();
    }
}
```

##### 内部类 FairSync

```java
// Sync的公平实现版本
static final class FairSync extends Sync {
    private static final long serialVersionUID = -2274990926593161451L;
    // 公平策略下，无论是获取写锁还是读锁，都必须判断当前获取锁的线程前面是否有线程在等待
    final boolean writerShouldBlock() {
        return hasQueuedPredecessors();
    }
    final boolean readerShouldBlock() {
        return hasQueuedPredecessors();
    }
}
```

##### 内部类 ReadLock

```java
// 读锁
public static class ReadLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -5992448646407690164L;
    private final Sync sync;

    // 构造器
    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    // 获取读锁，不响应中断
    public void lock() {
        sync.acquireShared(1);
    }

    // 响应中断的获取读锁
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

   // 尝试获取读锁
    public boolean tryLock() {
        return sync.tryReadLock();
    }

   // 响应中断、限时的尝试获取读锁
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    // 释放读锁
    public void unlock() {
        sync.releaseShared(1);
    }

    // 读锁不支持Condition
    public Condition newCondition() {
        throw new UnsupportedOperationException();
    }

   // 读锁的字符串表示
    public String toString() {
        int r = sync.getReadLockCount();
        return super.toString() +
            "[Read locks = " + r + "]";
    }
}
```

##### 内部类 WriteLock

```java
// 写锁
public static class WriteLock implements Lock, java.io.Serializable {
    private static final long serialVersionUID = -4992448646407690164L;
    private final Sync sync;

   	// 构造器
    protected WriteLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    // 获取写锁，不响应中断
    public void lock() {
        sync.acquire(1);
    }

    // 响应中断的获取写锁
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireInterruptibly(1);
    }

   // 尝试获取写锁
    public boolean tryLock( ) {
        return sync.tryWriteLock();
    }

    // 响应中断、限时的尝试获取写锁
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }

    // 释放写锁
    public void unlock() {
        sync.release(1);
    }

    // 创建与该写锁关联的Condition实例
    public Condition newCondition() {
        return sync.newCondition();
    }

    // 写锁字符串表示
    public String toString() {
        Thread o = sync.getOwner();
        return super.toString() + ((o == null) ?
                                   "[Unlocked]" :
                                   "[Locked by thread " + o.getName() + "]");
    }

    // 写锁是否由当前线程持有
    public boolean isHeldByCurrentThread() {
        return sync.isHeldExclusively();
    }

    // 写锁的持有数
    public int getHoldCount() {
        return sync.getWriteHoldCount();
    }
}
```

### 六、总结

`ReeantrantReadWriteLock`是一个设计非常优秀的类，支持设置公平策略，`read/write Lock`，锁的降级(锁升级会导致死锁)；而本文只是分析了`ReeantrantReadWriteLock`里面的代码，而要真正掌握 `ReeantrantReadWriteLock`，还需要掌握`Condition` 与 `AbstractQueuedSynchronizer`，可以查看之前[博客](https://xuanjian1992.top/tags/#JUC)的源码分析。

### 参考文献

- [JDK ReentrantReadWriteLock](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/ReentrantReadWriteLock.html)
- [【死磕Java并发】—–J.U.C之读写锁：ReentrantReadWriteLock](http://cmsblogs.com/?p=2213)
- [ReentrantReadWriteLock 源码分析(基于Java 8)](https://www.jianshu.com/p/6923c126e762)
- [读写锁ReentrantReadWriteLock之锁降级](https://www.jianshu.com/p/0f4a1995f57d)
- [JDK读写锁ReadWriteLock的升级和降级问题](https://blog.csdn.net/aitangyong/article/details/38315885)

