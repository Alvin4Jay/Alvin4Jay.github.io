---
layout:     post
title:      LinkedBlockingQueue源码分析
subtitle:   BlockingQueue
date:       2019-06-09
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JCF
---

# LinkedBlockingQueue源码分析

在前面[ArrayBlockingQueue源码分析](https://xuanjian1992.top/2019/06/09/ArrayBlockingQueue%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)中，已经对JDK中的BlockingQueue做了一个回顾，同时对ArrayBlockingQueue中的核心方法作了说明，而LinkedBlockingQueue作为JDK中BlockingQueue家族系列中的一员，由于其作为固定大小线程池(Executors.newFixedThreadPool())底层所使用的阻塞队列，分析它的目的主要在于2点：
 (1) 与ArrayBlockingQueue进行类比学习，加深各种数据结构的理解；
 (2) 了解底层实现，能够更好地理解每一种阻塞队列对线程池性能的影响。

本文内容如下：

- 分析LinkedBlockingQueue的实现
- 与ArrayBlockingQueue进行比较
- 说明选择LinkedBlockingQueue作为固定大小线程池的阻塞队列的原因。

### 一、LinkedBlockingQueue的实现分析

##### 1.核心属性

LinkedBlockingQueue是一个基于链表的阻塞队列，首先看一下核心属性：

```java
// 所有的元素都通过Node这个静态内部类来进行存储，这与LinkedList的处理方式完全一样
static class Node<E> {
    E item; // 使用item来保存元素本身

  	// 存在三种情况：
  	// 1.真正存在的后继节点 2.指向当前节点本身，说明该节点已经出队 
  	// 3.null，表示无后继节点，该节点是最后的节点
    Node<E> next; // 保存当前节点的后继节点

    Node(E x) { item = x; }
}

// 阻塞队列所能存储的最大容量，用户可以在创建时手动指定最大容量，
// 如果用户没有指定最大容量，那么默认的最大容量为Integer.MAX_VALUE。
private final int capacity;

// 队列中当前元素个数，使用AtomicInteger来保证并发修改的安全性。因为队列中可能同时存在存、取操作。
private final AtomicInteger count = new AtomicInteger();

// 链表头结点，head.item == null。说明真正存储数据的头节点是head.next
transient Node<E> head;

// 链表尾节点，last.next == null
private transient Node<E> last;

// 当执行take、poll等操作时线程需要获取的锁
private final ReentrantLock takeLock = new ReentrantLock();

// 当队列为空时，通过该Condition让从队列中获取元素的线程处于等待状态
private final Condition notEmpty = takeLock.newCondition();

// 当执行add、put、offer等操作时线程需要获取锁
private final ReentrantLock putLock = new ReentrantLock();

// 当队列的元素已经达到capactiy，通过该Condition让元素入队列的线程处于等待状态
private final Condition notFull = putLock.newCondition();
```

通过上面的分析，可以发现LinkedBlockingQueue在元素入队列和出队列时使用的不是同一个Lock，这意味着它们之间的操作不会存在互斥性。**在多个CPU的情况下，它们可以做到真正的在同一时刻既消费、又生产，能够做到并行处理。**

##### 2.构造函数

```java
public LinkedBlockingQueue() { // 如果用户没有显式指定capacity的值，默认使用Integer的最大值
    this(Integer.MAX_VALUE);
}

// 显式指定capacity的值。
// 当队列中没有任何元素的时候，此时队列的头部就等于队列的尾部, 指向的是同一个节点，并且元素的内容为null
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}

// 指定LinkedBlockingQueue初始化时入队列的集合
public LinkedBlockingQueue(Collection<? extends E> c) {
    this(Integer.MAX_VALUE); // 容量为最大值
    final ReentrantLock putLock = this.putLock;
    putLock.lock(); // Never contended, but necessary for visibility 获取putLock
    try {
        int n = 0;
        for (E e : c) {
            if (e == null)
                throw new NullPointerException(); // 元素不能为null
            if (n == capacity)
                throw new IllegalStateException("Queue full");
            enqueue(new Node<E>(e)); // 入队列
            ++n;
        }
        count.set(n); // 更新count值
    } finally {
        putLock.unlock();
    }
}
// 元素入队列(前提是线程持有putLock)
private void enqueue(Node<E> node) {
    // assert putLock.isHeldByCurrentThread();
    // assert last.next == null;
  	// 1.插入节点到尾部 2.更新last变量为刚插入的节点
    last = last.next = node;
}
```

##### 3.入队方法(put/offer)

###### (1) put(e)——阻塞

若执行put操作时队列已满，则阻塞等待队列变为非满时才进行入队操作。

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException(); // 入队元素不能为null
    // Note: convention in all put/take/etc is to preset local var
    // holding count negative to indicate failure unless set.
  	// 约定入队、出队操作时预设置局部变量c为-1，表示操作是否成功。负值表示操作失败，>=0表示成功。
    int c = -1;
    Node<E> node = new Node<E>(e); // 新建节点
    final ReentrantLock putLock = this.putLock; // putLock
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly(); // 可中断的获取锁
    try {
        // 当队列的容量到底最大容量时，此时线程将处于等待状态，直到队列有空闲的位置才继续执行。
      	// 使用while判断是为了防止线程被"伪唤醒”，即当线程被唤醒而队列的大小依旧等于
      	// capacity时，线程应该继续等待。
        while (count.get() == capacity) {
            notFull.await(); // 阻塞等待
        }
        enqueue(node); // 入队队列尾部
        c = count.getAndIncrement(); // count值+1，返回原count值
      	// c+1得到的结果是新元素入队列之后队列元素的总和。当前队列中的总元素个数小于最大容量时，此时唤醒其他
      	// 执行入队列的线程让它们可以放入元素，如果新加入元素之后，队列的大小等于capacity，那么就意味着此时
      	// 队列已经满了，也就没有必要唤醒其他正在等待入队列的线程，因为唤醒它们之后，它们也还是继续等待。
        if (c + 1 < capacity)
            notFull.signal(); // 通知入队列线程
    } finally {
        putLock.unlock(); // 释放putLock
    }
  	// 当c=0时，意味着之前的队列是空队列，出队列的线程都处于等待状态，现在新添加了一个元素，即队列不再为空，
  	// 因此它会唤醒正在等待获取元素的线程。
    if (c == 0)
        signalNotEmpty();
}
// 唤醒正在等待获取元素的线程，告诉它们现在队列中有元素了
private void signalNotEmpty() {
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        notEmpty.signal(); // 通过notEmpty条件唤醒获取元素的线程
    } finally {
        takeLock.unlock();
    }
}
```

###### (2) offer(e)——非阻塞

offer(e)与前面的put(e)不同，若执行该操作时队列已满，则直接返回false，**不会阻塞线程**。

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException(); // 元素不能为null
    final AtomicInteger count = this.count;
    if (count.get() == capacity) // 队列已满，直接返回false
        return false;
    int c = -1;
    Node<E> node = new Node<E>(e);
    final ReentrantLock putLock = this.putLock;
    putLock.lock(); // 获取putLock
    try {
      	// 当获取到锁时，需要进行二次的检查，因为可能当队列的大小为capacity-1时，两个线程同时去抢占锁，
      	// 而只有一个线程抢占成功，那么此时当线程将元素入队列后，释放锁，后面的线程抢占锁之后，此时队列
      	// 大小已经达到capacity，所以它将无法让元素入队列。下面的其余操作和put都一样，此处不再详述。
        if (count.get() < capacity) {
            enqueue(node); // 只有第二次检查，count小于容量时才入队
            c = count.getAndIncrement();
            if (c + 1 < capacity)
                notFull.signal();
        }
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return c >= 0; // c >=0 表示offer操作元素入队成功
}
```

###### (3) offer(e, timeout, unit)——阻塞

BlockingQueue定义了一个限时等待插入操作，即在一定的时间内，如果队列有空间可以插入，那么就将元素入队列，然后返回true；如果在过完指定的时间后依旧没有空间可以插入，那么就返回false，下面是限时等待操作的分析:

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {

    if (e == null) throw new NullPointerException();
    long nanos = unit.toNanos(timeout); // 时间转为纳秒形式
    int c = -1;
    final ReentrantLock putLock = this.putLock;
    final AtomicInteger count = this.count;
    putLock.lockInterruptibly(); // 可中断的获取锁
    try { 
        while (count.get() == capacity) { // 判断队列是否已满。这里用循环，防止可能的假醒现象。
            if (nanos <= 0) // 队列已满且剩余的等待时间为0，返回添加失败
                return false;
          	// 队列已满且剩余时间>0，执行等待。awaitNanos在返回后，返回剩余可以继续等待的时间。
            nanos = notFull.awaitNanos(nanos);
        }
        enqueue(new Node<E>(e)); // 执行入队操作
      	// 下面的其余操作和put都一样，此处不再详述。
        c = count.getAndIncrement();
        if (c + 1 < capacity)
            notFull.signal();
    } finally {
        putLock.unlock();
    }
    if (c == 0)
        signalNotEmpty();
    return true;
}
```

通过上面的分析，应该比较清楚地了解了LinkedBlockingQueue的入队列操作，其主要是通过获取putLock锁来完成，当队列中元素的数量达到最大值时，此时会导致线程处于阻塞状态或者返回false(根据具体的方法来看)；如果队列还有剩余的空间，那么此时会将新创建的Node对象，入队列到队列的尾部，作为LinkedBlockingQueue的last元素。

##### 4.出队操作(take/poll)

###### (1) take()——阻塞

若执行take操作时队列为空，则阻塞等待队列变为非空时才进行出队操作。

```java
public E take() throws InterruptedException {
    E x; // 返回值
    int c = -1;
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly(); // 获取takeLock锁，并且阻塞时可中断
    try {
        while (count.get() == 0) { // 检测队列是否为空。这里用循环，防止可能的假醒现象。
            notEmpty.await(); // 为空则阻塞等待
        }
        x = dequeue(); // 出队列
        c = count.getAndDecrement(); // count-1，并返回之前的count值
      	// c>1表明当前出队列操作之后队列中依然存在元素(c-1>0)，当前线程会唤醒其他执行元素出队列的线程，
      	// 让它们也可以执行元素的获取
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock(); // 释放putLock
    }
  	// 当c==capaitcy时，即在获取当前元素之前，队列已经满了，而此时获取元素之后，队列就会空出一个位置，
  	// 故当前线程会唤醒执行插入操作的线程，通知其中的一个可以进行插入操作。
    if (c == capacity)
        signalNotFull();
    return x;
}
// 出队操作(前提是获取takeLock)
// 让头部元素出队列的过程，其最终的目的是让原来的head被GC回收，让其next成为head，并且新的head的
// item为null。
private E dequeue() {
    // assert takeLock.isHeldByCurrentThread();
    // assert head.item == null;
    Node<E> h = head;
    Node<E> first = h.next;
    h.next = h; // help GC
    head = first; 
    E x = first.item;
    first.item = null;
    return x;
}
// 唤醒正在等待添加元素的线程，告诉它们现在队列中有空间了
private void signalNotFull() {
    final ReentrantLock putLock = this.putLock;
    putLock.lock();
    try {
        notFull.signal(); // 通过notFull条件唤醒添加元素的线程
    } finally {
        putLock.unlock(); 
    }
}
```

###### (2) poll()——非阻塞

poll()与前面的take()不同，若执行该操作时队列为空，则直接返回null，**不会阻塞线程**。

```java
public E poll() {
    final AtomicInteger count = this.count;
    if (count.get() == 0) // 队列为空，直接返回null(非阻塞)
        return null;
    E x = null;
    int c = -1;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lock();
    try {
        if (count.get() > 0) { // 再次判断队列元素个数是否>0
				// 以下逻辑take()操作一致，不在详述
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        }
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

###### (3) poll(timeout, unit)——阻塞

限时阻塞版本的poll()方法。

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E x = null;
    int c = -1;
    long nanos = unit.toNanos(timeout); // 等待时间，转为纳秒形式
    final AtomicInteger count = this.count;
    final ReentrantLock takeLock = this.takeLock;
    takeLock.lockInterruptibly();
    try {
        while (count.get() == 0) { // 判断队列是否为空。这里用循环，防止可能的假醒现象。
            if (nanos <= 0) // 队列为空且剩余的等待时间为0，返回添加失败
                return null;
          	// 队列为空且剩余时间>0，执行阻塞等待。awaitNanos在返回后，返回剩余可以继续等待的时间。
            nanos = notEmpty.awaitNanos(nanos);
        }
	      // 以下逻辑take()操作一致，不在详述
        x = dequeue();
        c = count.getAndDecrement();
        if (c > 1)
            notEmpty.signal();
    } finally {
        takeLock.unlock();
    }
    if (c == capacity)
        signalNotFull();
    return x;
}
```

### 二、LinkedBlockingQueue与ArrayBlockingQueue的比较

ArrayBlockingQueue由于其底层基于数组，并且在创建时指定存储的大小，在完成后就会立即在内存分配固定大小的数组容量，因此其存储通常有限，故其是一个**“有界“**的阻塞队列；而LinkedBlockingQueue可以由用户指定最大存储容量，也可以无需指定，如果不指定则最大存储容量将是Integer.MAX_VALUE，即可以看作是一个**“无界”**的阻塞队列，由于其节点的创建都是动态创建，并且在节点出队列后可以被GC所回收，因此其具有灵活的伸缩性。但是由于ArrayBlockingQueue的有界性，因此其能够更好的对于性能进行预测，而LinkedBlockingQueue由于没有限制大小，当任务非常多的时候，不停地向队列中存储，就有可能导致**内存溢出**的情况发生。

其次，ArrayBlockingQueue在入队列和出队列操作的过程中，使用的是同一个lock，所以即使在多核CPU的情况下，其出队和入队操作都无法做到并行，而LinkedBlockingQueue的出队和入队操作所使用的锁是两个不同的lock，它们之间的操作互相不受干扰，因此两种操作可以并行完成，故**LinkedBlockingQueue的吞吐量要高于ArrayBlockingQueue。**

### 三、线程池选择LinkedBlockingQueue的理由

```java
// 下面的代码是Executors创建固定大小线程池的代码，其使用了LinkedBlockingQueue作为任务队列。
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

JDK中固定大小线程池选用LinkedBlockingQueue作为阻塞队列的原因在于其**无界性**。因为线程大小固定的线程池，其线程的数量是不具备伸缩性的，当任务非常繁忙的时候，就势必会导致所有的线程都处于工作状态，如果使用一个有界的阻塞队列来进行处理，那么就非常有可能很快导致队列满的情况发生，从而导致任务无法提交而抛出RejectedExecutionException，而使用无界队列由于其良好的存储容量的伸缩性，可以很好的缓冲任务。繁忙情况下，即使任务非常多，也可以进行动态扩容，当任务被处理完成之后，队列中的节点也会被随之GC回收，非常灵活。

至此，LinkedBlockingQueue分析完毕。

### 参考文章

- [【Java并发编程】—–“J.U.C”：LinkedBlockingQueue](https://www.jianshu.com/p/cc2281b1a6bc)
- [Java阻塞队列ArrayBlockingQueue和LinkedBlockingQueue实现原理分析](http://www.importnew.com/24055.html)

