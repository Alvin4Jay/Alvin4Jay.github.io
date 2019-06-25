---
layout:     post
title:      DelayQueue源码分析
subtitle:   DelayQueue/BlockingQueue
date:       2019-06-25
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JCF
---

# DelayQueue源码分析

DelayQueue是一个包含Delayed类型元素的无界阻塞队列，只有在元素延迟时间过期后才能获取这些元素。DelayQueue的特性基本上是由BlockingQueue、PriorityQueue和Delayed的特性来决定的。DelayQueue通过Delayed接口，使得不同元素之间能按照剩余的延迟时间进行排序(`Delayed.getDelay()`)，然后通过PriorityQueue(优先队列，基于小顶堆)，使得`Delayed.getDelay()`最小的元素或者最早过期的元素能最先被处理，然后利用BlockingQueue，将元素处理的操作阻塞住。

### 一、Delayed接口分析

```java
public interface Delayed extends Comparable<Delayed> {

  	// 返回剩余的延迟时间。value<=0，表示延迟已过期
    long getDelay(TimeUnit unit);
}
```

> An implementation of this interface must define a {@code compareTo} method that provides an ordering consistent with its {@code getDelay} method.

Delayed接口继承自Comparable接口，并且有一个getDelay(TimeUnit)方法用于获取剩余延迟时间。同时该接口定义中指出，**Delayed接口的实现类必须实现compareTo()方法，并且compareTo()方法必须提供与getDelay()方法一致的排序规则和顺序**。下面DelayQueue中的元素实现了Delayed接口，并保存在优先队列(小顶堆)PriorityQueue中，排序规则便是getDelay()方法的值。

### 二、DelayQueue分析

##### 1.重要属性

```java
// 重入锁，用于线程同步
private final transient ReentrantLock lock = new ReentrantLock(); 
// 优先队列(小顶堆)，存放Delayed类型元素。Delayed.getDelay()值小的元素在堆顶。
private final PriorityQueue<E> q = new PriorityQueue<E>(); 

// 当前等待获取过期元素的leader线程
// 1.当有线程已成为leader时，该线程是限时等待元素，其他follower线程是无限期的等待元素。
// 2.当leader获取到元素并从take()、poll()返回时，它必须唤醒其他等待的线程，
// 除非其他线程成为了新leader(队列头部元素被具有更早过期时间的元素替换)
// 3.当堆顶元素(队列头部元素)被具有更早过期时间的元素替换时，leader属性置为null，
// 唤醒任何等待的线程(原leader或followers)
// 4.任何等待的线程都有可能获取和失去leader资格
private Thread leader = null;

// 1.当堆顶元素(队列头部元素)被具有更早过期时间的元素替换时，调用available.signal()；
// 2.当leader线程获取到过期元素，在返回之前，调用available.signal()唤醒某个线程，使之能成为leader
private final Condition available = lock.newCondition();
```

##### 2.构造函数

```java
public DelayQueue() {} // 构造空的DelayQueue

public DelayQueue(Collection<? extends E> c) { // 构造一个初始包含c中元素的DelayQueue
    this.addAll(c);
}
```

##### 3.添加元素的方法(非阻塞)

添加元素的方法有add(E e)，offer(E e)，put(E e)和offer(E e, long timeout, TimeUnit unit)。

###### (1) add(E e)

```java
public boolean add(E e) {
    return offer(e);
}
```

###### (2) put(E e)

```java
public void put(E e) {
    offer(e);
}
```

###### (3) offer(E e, long timeout, TimeUnit unit)

```java
public boolean offer(E e, long timeout, TimeUnit unit) {
    return offer(e);
}
```

###### (4) offer(E e)

以上add(E e)，put(E e)和offer(E e, long timeout, TimeUnit unit)最终都调用到了offer(E e)方法。下面分析其实现。

```java
public boolean offer(E e) { // 非阻塞式添加元素
    final ReentrantLock lock = this.lock;
    lock.lock(); // 先获取互斥锁
    try {
        q.offer(e); // e保存到优先队列q中，堆顶元素可能发生了变化
      	// 下面有两种情形:
        // 1.第一次存入元素，所以q.peek() == e，唤醒等待的线程；
        // 2.非第一次添加元素，添加后，该元素上浮到堆顶，此时之前线程等待的队列头部元素发生了变化,
        // 不管之前线程leader与followers，直接将leader置为null，重新选择leader线程(执行
        // 唤醒操作，可能是原leader或follower线程被唤醒，成为新leader)，执行元素的获取。
        if (q.peek() == e) {
            leader = null;
            available.signal();
        }
        return true; // 因为DelayQueue是无界队列，所以添加操作总是返回成功
    } finally {
        lock.unlock(); // 释放锁
    }
}
```

##### 4.获取元素的操作

获取元素的方法有poll()，take()和poll(long timeout, TimeUnit unit)。

###### (1) poll()(非阻塞)

获取并删除队列的头部元素，如果还未过期，则返回null。

```java
// 获取并删除队列的头部元素，如果还未过期，则返回null
public E poll() {
    final ReentrantLock lock = this.lock;
    lock.lock(); // 获取互斥锁
    try {
        E first = q.peek(); // 获取堆顶元素(优先队列头部元素)
        if (first == null || first.getDelay(NANOSECONDS) > 0)// 如果为null或者还未过期，返回null
            return null;
        else
            return q.poll(); // 否则，返回已过期的头部元素
    } finally {
        lock.unlock(); // 释放锁
    }
}
```

###### (2) take()(阻塞)

阻塞式地获取过期的元素。

```java
// 阻塞式地获取过期的元素
public E take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 获取互斥锁，可被中断
    try {
        for (;;) {
            E first = q.peek(); // 队列头部元素
            if (first == null)
                available.await(); // 队头元素为空(无元素)，直接等待
            else {
                long delay = first.getDelay(NANOSECONDS); // 获取元素过期还需要等待的时间
                if (delay <= 0)
                    return q.poll(); // 元素已过期，可以直接取出、返回
                first = null; // 线程在等待时不能持有队头元素的引用，防止内存泄漏
                if (leader != null) // leader线程已存在，该线程直接等待
                  	// 等待线程可能被leader线程唤醒，也可能被添加元素的线程唤醒(选择新leader)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread; // 当前线程成为leader线程
                    try {
                        available.awaitNanos(delay); // 限时等待
                    } finally {
                      	// 1.该leader线程限时等待完成，可以获取队头过期元素返回了，
                        // 并将leader置为null
                        // 2.有可能等待期间队头元素发生变化(新元素添加)，此时
                        // leader != thisThread(leader重新选择)
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
      	// 如果leader线程为null且队头任务不为空，唤醒其中一个等待线程，使之能成为新leader
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock(); // 释放锁
    }
}
```

###### (3)poll(long timeout, TimeUnit unit)(阻塞)

限时获取队头过期的元素，如果在指定时间内没有元素过期，则返回null。

```java
// 限时获取队头过期的元素，如果在指定时间内没有元素过期，则返回null
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    long nanos = unit.toNanos(timeout); // 最多等待的时间(纳秒)
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 获取互斥锁，可被中断
    try {
        for (;;) {
            E first = q.peek(); // 队列头部元素
            if (first == null) {
                if (nanos <= 0) // 元素为null(队列为空)且已超时，则直接返回null
                    return null;
                else // 元素为null(队列为空)且未超时，则等待nanos时间
                    nanos = available.awaitNanos(nanos);
            } else {
                long delay = first.getDelay(NANOSECONDS); // 获取元素过期还需要等待的时间
                if (delay <= 0) // 元素已过期，可以直接取出、返回
                    return q.poll();
                if (nanos <= 0) // 已超时，则直接返回null
                    return null;
                first = null; // 线程在等待时不能持有队头任务的引用，防止内存泄漏
              	// leader线程已存在或者最多等待的时间小于delay，该线程直接等待nanos时间
                if (nanos < delay || leader != null)
                  	// 等待线程可能被leader线程唤醒，也可能被添加元素的线程唤醒(选择新leader)
                    nanos = available.awaitNanos(nanos);
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread; // 当前线程成为leader线程
                    try {
                        long timeLeft = available.awaitNanos(delay); // 等待delay时间
                        nanos -= delay - timeLeft; // 还可以等待的时间
                    } finally {
                      // 1.该leader线程限时等待完成，可以获取队头过期元素返回了，
                      // 并将leader置为null
                      // 2.有可能等待期间队头元素发生变化(新元素添加，并成为堆顶)，此时
                      // leader != thisThread(leader重新选择)
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
      	// 如果leader线程为null且队头任务不为空，唤醒其中一个等待线程，使之能成为新leader
        if (leader == null && q.peek() != null)
            available.signal();
        lock.unlock(); // 释放锁
    }
}
```

##### 5.DelayQueue总结

- DelayQueue内部使用PriorityQueue来保存元素和维护元素顺序；
- DelayQueue存储的元素必须实现Delayed接口，通过实现Delayed接口，可以获取到元素的剩余延迟时间，以及可以比较元素大小(Delayed 继承Comparable)；
- DelayQueue通过一个重入锁来控制元素的入队出队行为；
- PriorityQueue只是负责存储数据以及维护元素的顺序，对于元素是否过期以及取数据则是在DelayQueue中进行判断控制的。

### 三、参考文献

- [DelayQueue JDK8](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/DelayQueue.html)