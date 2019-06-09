---
layout:     post
title:      ArrayBlockingQueue源码分析
subtitle:   BlockingQueue
date:       2019-06-09
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JCF
---

# ArrayBlockingQueue源码分析

本文主要分析BlockingQueue的实现之一——ArrayBlockingQueue的原理与实现，其底层使用重入锁ReentrantLock与条件Condition实现线程的并发控制与阻塞等待。

### 一、Queue/BlockingQueue接口分析

##### 1.Queue接口

> 在Queue接口中，除了继承Collection接口中定义的方法外，它还分别额外地定义**插入、删除、查询**这3个操作，其中每一个操作都以两种不同的形式存在，每一种形式都对应着一个方法。

方法说明: 

| 操作        | 抛出异常  | 返回特殊值 |
| ----------- | --------- | ---------- |
| **Insert**  | add(e)    | offer(e)   |
| **Remove**  | remove()  | poll()     |
| **Examine** | element() | peek()     |

- add方法在将一个元素插入到队列的尾部时，如果出现队列已经满了，那么就会抛出IllegalStateException，而使用offer方法时，如果队列满了，则添加失败，返回false，但并不会引发异常。
- remove方法是获取队列的头部元素并且删除，如果当队列为空时，那么就会抛出NoSuchElementException。而poll在队列为空时，则返回一个null。
- element方法是获取到队列的第一个元素，但不会删除，但是如果队列为空时，那么它就会抛出NoSuchElementException。peek方法与之类似，只是不会抛出异常，而是返回false。

后面在分析ArrayBlockingQueue的方法时，主要也是围绕着这几个方法来进行分析。

##### 2.BlockingQueue接口

> BlockingQueue是JDK1.5出现的接口，它在原来的Queue接口基础上提供了更多的额外功能：当获取队列中的头部元素时，如果队列为空，那么它将会使执行线程处于等待状态(take方法)；当添加一个元素到队列的尾部时，如果队列已经满了，那么它同样会使执行的线程处于等待状态(put方法)。

前面在介绍Queue接口时提到过，它针对于相同的操作提供了2种不同的形式，而BlockingQueue更夸张，针对于相同的操作提供了4种不同的形式。

该四种形式分别为：

- 抛出异常；
- 返回一个特殊值(可能是null或者是false，取决于具体的操作)；
- 阻塞当前执行直到其可以继续；
- 当线程被挂起后，等待最大的时间，如果一旦超时，即使该操作依旧无法继续执行，线程也不会再继续等待下去。

对应的方法说明:

| 操作        | 抛出异常  | 返回特殊值 | 阻塞   | 超时                 |
| ----------- | --------- | ---------- | ------ | -------------------- |
| **Insert**  | add(e)    | offer(e)   | put(e) | offer(e, time, unit) |
| **Remove**  | remove()  | poll()     | take() | poll(time, unit)     |
| **Examine** | element() | peek()     | 无     | 无                   |

BlockingQueue虽然比起Queue在操作上提供了更多的支持，但是它在使用时也应该保证如下的几点:

1.BlockingQueue中是不允许添加null的，该接口在声明的时候就要求所有的实现类在接收到一个null的时候，都应该抛出NullPointerException。

2.BlockingQueue是线程安全的，因此它的所有和队列相关的方法都具有原子性。但是对于那么从Collection接口中继承而来的批量操作方法，比如addAll(Collection e)等方法，BlockingQueue的实现通常没有保证其具有原子性，因此在使用BlockingQueue的时候，应该尽可能地不去使用这些方法。

3.BlockingQueue主要应用于生产者与消费者的模型中，其元素的添加和获取都是极具规律性的。但是对于remove(Object o)这样的方法，虽然BlockingQueue可以保证元素正确的删除，但是这样的操作会非常影响性能，因此在没有特殊的情况下，也应该避免使用这类方法。

### 二、ArrayBlockingQueue分析

有了上面的铺垫，下面开始分析ArrayBlockingQueue。

##### 1.核心属性

```java
// 底层维护队列元素的数组(注意是循环数组)
final Object[] items;

// 当读取元素时数组的下标(这里称为读下标)
int takeIndex;

// 添加元素时数组的下标 (这里称为写小标)
int putIndex;

// 队列中的元素个数
int count;

// 用于并发控制的重入锁(获取元素、添加元素全局使用同一把锁)
final ReentrantLock lock;

// 控制take操作时是否让线程等待的条件(队列非空)
private final Condition notEmpty;

// 控制put操作时是否让线程等待的条件(队列非满)
private final Condition notFull;
```

##### 2.构造函数

```java
public ArrayBlockingQueue(int capacity) { // 指定容量，采用默认的非公平锁
    this(capacity, false);
}

// 指定容量，指定使用公平锁或非公平锁
public ArrayBlockingQueue(int capacity, boolean fair) {
    if (capacity <= 0)
        throw new IllegalArgumentException();
    this.items = new Object[capacity];
    lock = new ReentrantLock(fair);
    notEmpty = lock.newCondition();
    notFull =  lock.newCondition();
}

// 指定容量、使用公平锁或非公平锁，以及初始入队列的集合元素
public ArrayBlockingQueue(int capacity, boolean fair, Collection<? extends E> c) {
    this(capacity, fair); // 调用第二个构造函数，先初始化

    final ReentrantLock lock = this.lock;
    lock.lock(); // Lock only for visibility, not mutual exclusion 获得锁
    try {
        int i = 0;
        try {
            for (E e : c) {
                checkNotNull(e);
                items[i++] = e; // 集合c的元素入队列
            }
        } catch (ArrayIndexOutOfBoundsException ex) {
            throw new IllegalArgumentException();
        }
        count = i; // 更新count
        putIndex = (i == capacity) ? 0 : i; // 更新putIndex
    } finally {
        lock.unlock(); // 释放锁
    }
}
```

##### 3.添加元素的方法(入队，add/offer/put)

###### (1) add(e)/offer(e)——非阻塞

add(e)方法底层调用的是offer方法(非阻塞)。

```java
// 添加成功返回true，失败(队列已满)抛出IllegalStateException异常
public boolean add(E e) {
    return super.add(e); // 调用父类AbstractQueue.add(E)方法
}

// 父类AbstractQueue.add(E)
public boolean add(E e) {
    if (offer(e)) // 调用子类ArrayBlockingQueue的offer(e)方法，成功返回true
        return true;
    else // 添加失败(队列已满)，抛出异常
        throw new IllegalStateException("Queue full");
}

// ArrayBlockingQueue.offer(e) 添加成功返回true，若队列已满导致添加失败，返回false
public boolean offer(E e) {
    checkNotNull(e); // 元素不能为null
    final ReentrantLock lock = this.lock;
    lock.lock(); // 获取锁
    try {
        if (count == items.length) // 队列已满，添加失败，返回false
            return false;
        else {
            enqueue(e); // 执行入队操作
            return true; // 添加成功
        }
    } finally {
        lock.unlock(); // 释放锁
    }
}
```

入队操作enqueue(e)方法如下：

```java
// 入队操作必须在获得锁的前提下进行
private void enqueue(E x) {
    // assert lock.getHoldCount() == 1;
    // assert items[putIndex] == null;
    final Object[] items = this.items; // 获取底层数组引用
    items[putIndex] = x; // x放入数组
	  // putIndex递增，若putIndex == items.length，表示putIndex已达到数组末尾。由于是循环数组，
  	// putIndex置0，后续添加操作从index=0开始
    if (++putIndex == items.length) 
        putIndex = 0;
    count++; // 元素个数+1
    notEmpty.signal(); // 由于有元素入队，则通知可能在阻塞等待获取元素的线程
}
```

###### (2) offer(e, timeout, unit)——阻塞

这个方法是offer(e)添加元素的限时阻塞版本。若在添加元素时，队列已满，则线程最多阻塞等待给定的时间。如果阻塞等待给定时间后队列还是没有空间，则返回false失败。

```java
public boolean offer(E e, long timeout, TimeUnit unit) // 等待最多timeout时间
    throws InterruptedException {

    checkNotNull(e); // 元素不能为null
    long nanos = unit.toNanos(timeout); // 转换为纳秒
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 获取锁，并且阻塞时可中断
    try {
        while (count == items.length) { // 检测队列是否已满。这里用循环，防止可能的假醒现象。
            if (nanos <= 0) // 队列已满且剩余的等待时间为0，返回添加失败
                return false;
	          // 队列已满且剩余时间>0，执行等待。awaitNanos在返回后，返回剩余可以继续等待的时间。
            nanos = notFull.awaitNanos(nanos); 
        }
        enqueue(e); // 入队列操作
        return true; // 添加成功
    } finally {
        lock.unlock();
    }
}
```

###### (3) put(e)——阻塞

```java
// 添加元素，若队列已满，则阻塞等待，直到被唤醒且有空间可以添加元素。
public void put(E e) throws InterruptedException {
    checkNotNull(e); // 元素不能为null
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 获取锁，并且阻塞时可中断
    try {
        while (count == items.length) // 检测队列是否已满。这里用循环，防止可能的假醒现象。
            notFull.await(); // 队列已满，阻塞等待。
        enqueue(e); // 入队列
    } finally {
        lock.unlock();
    }
}
```

##### 4.获取元素的方法(出队，remove/poll/take)

###### (1) remove()/poll()——非阻塞

remove()方法定义在父类AbstractQueue中，如下所示，底层调用是的poll()方法：

```java
// AbstractQueue.remove() 
public E remove() {
    E x = poll(); // 调用poll()获取元素
    if (x != null) // 队列不为空，获取到元素后返回
        return x;
    else // 否则抛出异常
        throw new NoSuchElementException();
}
// ArrayBlockingQueue.poll() 队列中无元素，返回null；否则返回队列头部元素
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 获取锁
    try {
        return (count == 0) ? null : dequeue(); // 存在元素时，调用dequeue()方法
    } finally {
        lock.unlock();
    }
}
// ArrayBlockingQueue.dequeue() 出队操作是在获得锁的前提下进行的
private E dequeue() {
    // assert lock.getHoldCount() == 1;
    // assert items[takeIndex] != null;
    final Object[] items = this.items; // 获取底层数组的引用
    @SuppressWarnings("unchecked")
    E x = (E) items[takeIndex]; // 获取出队元素
    items[takeIndex] = null; // 已出队元素位置置null
    if (++takeIndex == items.length) // takeIndex+1，达到尾部，则takeIndex置0(循环数组)
        takeIndex = 0;
    count--; // 总数-1
    if (itrs != null)
        itrs.elementDequeued(); // 元素出队，更新迭代器状态
    notFull.signal(); // 通知可能阻塞等待添加元素的线程(可能因为之前队列满了)
    return x;
}
```

###### (2) poll(timeout, unit)——阻塞

这个方法是poll()获取元素的限时阻塞版本。若在获取元素时，队列为空，则线程最多阻塞等待给定的时间。如果阻塞等待给定时间后队列还是为空，则返回null。

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException { // 限时获取元素
    long nanos = unit.toNanos(timeout); // 转换为纳秒
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 获取锁，并且阻塞时可中断
    try {
        while (count == 0) { // 判断队列是否为空。这里用循环，防止可能的假醒现象。
            if (nanos <= 0) // 队列为空且剩余的等待时间为0，返回null
                return null;
          	// 队列为空且剩余时间>0，执行等待。awaitNanos在返回后，返回剩余可以继续等待的时间。
            nanos = notEmpty.awaitNanos(nanos);
        }
        return dequeue(); // 出队操作
    } finally {
        lock.unlock();
    }
}
```

###### (3) take()——阻塞

```java
// 获取元素，若队列为空，则阻塞等待，直到被唤醒且队列非空可以获取元素。
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 获取锁，并且阻塞时可中断
    try {
        while (count == 0) // 检测队列是否为空。这里用循环，防止可能的假醒现象。
            notEmpty.await(); // 为空则等待
        return dequeue(); // 出队列
    } finally {
        lock.unlock();
    }
}
```

至此，ArrayBlockingQueue分析完毕。

### 参考文章

- [【Java并发编程】—–“J.U.C”：ArrayBlockingQueue](https://www.jianshu.com/p/9a652250e0d1)
- [Java阻塞队列ArrayBlockingQueue和LinkedBlockingQueue实现原理分析](http://www.importnew.com/24055.html)
- [JDK1.8 ArrayBlockingQueue源码分析](https://blog.csdn.net/qq_22929803/article/details/52347071)
- [深入Java并发之阻塞队列-迭代器（一）](http://www.manongjc.com/article/63920.html)
- [深入Java并发之阻塞队列-迭代器（二）](http://www.ishenping.com/ArtInfo/274420.html)