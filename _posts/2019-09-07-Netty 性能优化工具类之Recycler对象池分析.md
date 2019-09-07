---
layout:     post
title:      Netty 轻量级对象池Recycler分析
subtitle:   Recycler
date:       2019-09-07
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Netty
---

# Netty 轻量级对象池Recycler分析

Recycler对象池在Netty中使用广泛，主要用来完成对象的管理，如获取对象、对象用完之后回收对象，这样下次再获取该类型对象时可以直接从对象池中获取，减少对象的创建性能消耗。如PooledUnsafeDirectByteBuf中对象池的使用:

```java
private static final Recycler<PooledUnsafeDirectByteBuf> RECYCLER = new Recycler<PooledUnsafeDirectByteBuf>() {
    @Override
    protected PooledUnsafeDirectByteBuf newObject(Handle<PooledUnsafeDirectByteBuf> handle) {
        return new PooledUnsafeDirectByteBuf(handle, 0);
    }
};

static PooledUnsafeDirectByteBuf newInstance(int maxCapacity) {
    PooledUnsafeDirectByteBuf buf = RECYCLER.get();
    buf.reuse(maxCapacity);
    return buf;
}
```

使用对象池有如下的好处:

- 对象复用——因为对象都缓存在对象池中，因此对象得以复用，不用每次都创建对象，减少new的性能消耗；
- 减少young GC——因为减少了不必要的对象创建，因此JVM年轻代垃圾减少，因此young  GC减少。

下面详细分析轻量级对象池Recycler的原理。

### 一、Recycler的使用

```java
public class RecyclerTest {

    private static final Recycler<User> RECYCLER = new Recycler<User>() {
        @Override
        protected User newObject(Handle<User> handle) {
            return new User(handle);
        }
    };

    public static class User {
        private Recycler.Handle<User> handle;

        public User(Recycler.Handle<User> handle) {
            this.handle = handle;
        }

        public void recycle() {
            handle.recycle(this);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        User user = RECYCLER.get();

        user.recycle(); // 同线程回收对象

//        new FastThreadLocalThread(() -> {
//            user.recycle();
//        }).start(); // 异线程回收对象

        System.out.println(user == RECYCLER.get()); // true
    }

}
```

如上是Recycler使用的一个例子，通过`RECYCLER.get()`获取User对象，对象用完之后调用`user.recycle()`回收对象。回收对象有两种方式，一种是同线程回收对象，即对象在一个线程中分配，同时在该线程中回收对象；另一种是异线程回收对象，即对象在一个线程中分配，但在另一个线程中回收对象。

### 二、Recycler的创建

从上面的例子可以看到，创建Recycler对象池时需要实现newObject()抽象方法。下面看其构造器:

```java
private final int maxCapacityPerThread; // 32768
private final int maxSharedCapacityFactor; // 2
private final int ratioMask; // 7
private final int maxDelayedQueuesPerThread; // 2倍的CPU核数

protected Recycler() {
    this(DEFAULT_MAX_CAPACITY_PER_THREAD);
}

protected Recycler(int maxCapacityPerThread) {
    this(maxCapacityPerThread, MAX_SHARED_CAPACITY_FACTOR);
}

protected Recycler(int maxCapacityPerThread, int maxSharedCapacityFactor) {
    this(maxCapacityPerThread, maxSharedCapacityFactor, RATIO, MAX_DELAYED_QUEUES_PER_THREAD);
}

protected Recycler(int maxCapacityPerThread, int maxSharedCapacityFactor,
                   int ratio, int maxDelayedQueuesPerThread) {
    ratioMask = safeFindNextPositivePowerOfTwo(ratio) - 1; // 7
    if (maxCapacityPerThread <= 0) {
        this.maxCapacityPerThread = 0;
        this.maxSharedCapacityFactor = 1;
        this.maxDelayedQueuesPerThread = 0;
    } else {
        this.maxCapacityPerThread = maxCapacityPerThread; // 32768
        this.maxSharedCapacityFactor = max(1, maxSharedCapacityFactor); // 2
        this.maxDelayedQueuesPerThread = max(0, maxDelayedQueuesPerThread); // 2倍CPU核数
    }
}
```

在默认情况下，Recycler实例的maxCapacityPerThread变量为32768，maxSharedCapacityFactor为2，ratioMask为7，maxDelayedQueuesPerThread为2倍的CPU核数。同时可以看到Recycler实例包含的FastThreadLocal threadLocal变量：

```java
private final FastThreadLocal<Stack<T>> threadLocal = new FastThreadLocal<Stack<T>>() {
    @Override
    protected Stack<T> initialValue() {
        return new Stack<T>(Recycler.this, Thread.currentThread(), maxCapacityPerThread, maxSharedCapacityFactor,
               ratioMask, maxDelayedQueuesPerThread);
    }
};
```

由于threadLocal变量是FastThreadLocal类型，因此不同线程访问threadLocal时，将获得线程本地变量Stack实例，因此线程和Stack实例是一一对应的。下面看Stack的结构:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Recycler-1.jpg)

```java
// Stack成员变量和构造器
final Recycler<T> parent;
final Thread thread;
final AtomicInteger availableSharedCapacity; // 16384
final int maxDelayedQueues; // WeakOrderQueue最大个数

private final int maxCapacity; // elements数组最大大小
private final int ratioMask; // 控制对象的回收比率
private DefaultHandle<?>[] elements;
private int size; // Stack缓存的DefaultHandle对象个数
private int handleRecycleCount = -1; // Start with -1 so the first one will be recycled.
private WeakOrderQueue cursor, prev;
private volatile WeakOrderQueue head;

Stack(Recycler<T> parent, Thread thread, int maxCapacity, int maxSharedCapacityFactor,
      int ratioMask, int maxDelayedQueues) {
    this.parent = parent; // Recycler
    this.thread = thread; // Reactor线程，创建该Stack的线程
    this.maxCapacity = maxCapacity; // 32768
  	// maxSharedCapacityFactor: 2, availableSharedCapacity: 16384
    availableSharedCapacity = new AtomicInteger(max(maxCapacity / maxSharedCapacityFactor, LINK_CAPACITY));
    elements = new DefaultHandle[min(INITIAL_CAPACITY, maxCapacity)]; // 256
    this.ratioMask = ratioMask; // 7
    this.maxDelayedQueues = maxDelayedQueues; // 2*CPU核数
}
```

从上图可以看到，Stack中包含一个elements数组，该数组用来保存DefaultHandle实例，而DefaultHandle实例中保存了实际回收的对象，如User。thread表示实际创建该Stack的线程，即Stack所属线程(如Thread1)。ratioMask用来控制对象回收的比率，默认只回收1/8的对象。maxCapacity表示elements数组的最大大小。maxDelayedQueues表示在异线程回收对象时，由其他线程(如Thread2)回收对象时创建的WeakOrderQueue最大个数，默认为2倍CPU核数。availableSharedCapacity表示异线程回收对象时，其他线程能保存的被回收对象的最大个数。指针head、prev、cursor指向异线程回收对象的情况下，由其他线程(如Thread2)回收对象时创建的WeakOrderQueue串联起来的链表。

### 三、从Recycler获取对象

##### 1.获取当前线程的Stack

从上面的例子可以看到，对象是通过`Recycler.get()`获取的:

```java
public final T get() {
    if (maxCapacityPerThread == 0) {
        return newObject((Handle<T>) NOOP_HANDLE); // 对象用完不会回收
    }
    Stack<T> stack = threadLocal.get();
    DefaultHandle<T> handle = stack.pop(); // 从缓存中弹出一个DefaultHandle对象
    if (handle == null) {
        handle = stack.newHandle();
        handle.value = newObject(handle); // 创建的对象，保存到DefaultHandle
    }
    return (T) handle.value; // T类型的对象
}
```

如果maxCapacityPerThread为0，则直接通过newObject()创建对象，且对象使用完之后并不会缓存到对象池中。如果maxCapacityPerThread大于0，则线程通过threadLocal获取Stack实例，上面已经分析过threadLocal的作用和Stack的初始化过程，这里不再详述。

##### 2.从Stack弹出对象

得到Stack之后，调用`stack.pop()`获取缓存的DefaultHandle实例，即从Stack弹出DefaultHandle对象。

```java
DefaultHandle<T> pop() {
    int size = this.size;
    if (size == 0) {
        if (!scavenge()) { // 从其他线程的WeakOrderQueue回收DefaultHandle
            return null;
        }
        size = this.size;
    }
    size --;
    DefaultHandle ret = elements[size];
    elements[size] = null;
    if (ret.lastRecycledId != ret.recycleId) {
        throw new IllegalStateException("recycled multiple times");
    }
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    this.size = size;
    return ret;
}
```

pop()方法中如果size不为0，即elements数组中存在缓存的DefaultHandle实例，则直接通过数组索引的方式获取DefaultHandle实例，并更新size大小。如果size为0，则需要先通过scavenge()方法，从其他线程的WeakOrderQueue中回收DefaultHandle对象。这些WeakOrderQueue都是其他线程在异线程回收对象时创建的，用来暂时保存由Stack对应线程创建的DefaultHandle对象。scavenge()方法将在**回收对象到Recycler中的异线程收割对象**部分再详细分析。

假设`stack.pop()`获取到了一个DefaultHandle实例，则直接拿出其中的实际对象并返回。如果未获取到DefaultHandle实例，则创建一个DefaultHandle对象并绑定Stack。

##### 3.创建DefaultHandle对象并绑定Stack

```java
DefaultHandle<T> handle = stack.pop(); // 从缓存中弹出一个DefaultHandle对象
if (handle == null) {
    handle = stack.newHandle();
    handle.value = newObject(handle); // 创建的对象，保存到DefaultHandle
}
```

首先调用`stack.newHandle()`创建DefaultHandle实例：

```java
DefaultHandle<T> newHandle() { // 创建DefaultHandle，与Stack绑定
    return new DefaultHandle<T>(this);
}
static final class DefaultHandle<T> implements Handle<T> {
    private int lastRecycledId;
    private int recycleId;

    boolean hasBeenRecycled; // 是否已经被回收过

    private Stack<?> stack; // 绑定的Stack
    private Object value; // 实际的对象

    DefaultHandle(Stack<?> stack) {
        this.stack = stack;
    }

    @Override
    public void recycle(Object object) {
        if (object != value) {
            throw new IllegalArgumentException("object does not belong to handle");
        }
        stack.push(this); // 对象回收时将对应的DefaultHandle放入Stack中
    }
}
```

DefaultHandle创建过程中将会绑定当前的Stack实例。然后调用newObject(handle)创建实例的对象，例如User，并将引用保存到DefaultHandle的value变量中。至此，DefaultHandle创建完毕。最后将DefaultHandle中保存的对象引用返回。

### 四、回收对象到Recycler

从前面的例子可以看到，回收对象通过以下方式完成：

```java
user.recycle();

public static class User {
    private Recycler.Handle<User> handle;

    public User(Recycler.Handle<User> handle) {
        this.handle = handle;
    }

    public void recycle() {
        handle.recycle(this);
    }
}
```

最终回收对象时，调用的是Handle.recycle(User)方法。

```java
// DefaultHandle
public void recycle(Object object) {
    if (object != value) {
        throw new IllegalArgumentException("object does not belong to handle");
    }
    stack.push(this); // 对象回收时将对应的DefaultHandle放入Stack中
}
```

这里调用Stack.push(DefaultHandle)回收对象:

```java
void push(DefaultHandle<?> item) {
    Thread currentThread = Thread.currentThread();
    if (thread == currentThread) { // 同线程回收对象
        pushNow(item);
    } else { // 异线程回收对象
        pushLater(item, currentThread);
    }
}
```

从该push()方法也可以看出，回收对象有两种情况。如果对象是在获取对象的线程中进行回收，则直接调用pushNow()方法回收对象；如果对象不是在获取对象的线程中进行回收，则调用pushLater()方法进行回收。下面分别进行讨论。

##### 1.同线程回收对象

```java
private void pushNow(DefaultHandle<?> item) {
    if ((item.recycleId | item.lastRecycledId) != 0) { // 防止多次回收
        throw new IllegalStateException("recycled already");
    }
    item.recycleId = item.lastRecycledId = OWN_THREAD_ID;

    int size = this.size;
    if (size >= maxCapacity || dropHandle(item)) {
        // Hit the maximum capacity or should drop - drop the possibly youngest object.
        return;
    }
    if (size == elements.length) { // 扩容
        elements = Arrays.copyOf(elements, min(size << 1, maxCapacity));
    }

    elements[size] = item;
    this.size = size + 1;
}
```

同线程回收对象时，首先判断Stack中缓存的DefaultHandle个数是否已达到maxCapacity，即32768。如果已达到，直接丢弃当前回收的DefaultHandle。之后通过dropHandle()方法判断是否应该丢弃本次回收的DefaultHandle实例:

```java
boolean dropHandle(DefaultHandle<?> handle) { // 判断是否回收handle对象
    if (!handle.hasBeenRecycled) {
        if ((++handleRecycleCount & ratioMask) != 0) { // 默认只回收1/8的对象
            // Drop the object.
            return true;
        }
        handle.hasBeenRecycled = true;
    }
    return false;
}
```

dropHandle()通过handleRecycleCount变量控制是否丢弃本次回收的DefaultHandle实例，可以看到默认情况下，只回收1/8的对象。并且当某对象判断为可以回收之后，会将hasBeenRecycled置为true。这样后面该对象再次被回收时，会被直接判断为**回收**。

假设判断当前Stack缓存的DefaultHandle个数之后，未超过maxCapacity，且`dropHandle(item)`返回false，即不丢弃本次回收的DefaultHandle对象，则将本次回收的DefaultHandle放入elements数组中。放入之前如果需要扩容，则进行扩容。

##### 2.异线程回收对象

```java
// 延迟回收的队列
private static final FastThreadLocal<Map<Stack<?>, WeakOrderQueue>> DELAYED_RECYCLED = 
        new FastThreadLocal<Map<Stack<?>, WeakOrderQueue>>() {
    @Override
    protected Map<Stack<?>, WeakOrderQueue> initialValue() {
        return new WeakHashMap<Stack<?>, WeakOrderQueue>();
    }
};

private void pushLater(DefaultHandle<?> item, Thread thread) {
    Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
    WeakOrderQueue queue = delayedRecycled.get(this);
    if (queue == null) {
        // 达到最大WeakOrderQueue个数，则放弃对象回收
        if (delayedRecycled.size() >= maxDelayedQueues) { 
            delayedRecycled.put(this, WeakOrderQueue.DUMMY);
            return;
        }
        if ((queue = WeakOrderQueue.allocate(this, thread)) == null) { // 创建WeakOrderQueue
            // drop object
            return;
        }
        delayedRecycled.put(this, queue); // 创建的WeakOrderQueue保存到Map
    } else if (queue == WeakOrderQueue.DUMMY) {
        // drop object
        return;
    }

    queue.add(item); // DefaultHandle保存到WeakOrderQueue
}
```

异线程回收对象的过程由pushLater()方法实现，该过程分为3步：获取weakOrderQueue、创建weakOrderQueue和将对象保存到WeakOrderQueue。

> 获取WeakOrderQueue

首先通过FastThreadLocal变量DELAYED_RECYCLED获取`Map<Stack<?>, WeakOrderQueue> delayedRecycled`，可以看到某线程在回收其他线程获取的对象时使用了`Map<Stack<?>, WeakOrderQueue>`数据结构。这个Map中的key是Stack，即创建Stack、获取对象的线程对应的Stack，value为异线程回收对象时，回收由其他线程获取的对象时创建的WeakOrderQueue。由于对某个线程来说，这个异线程回收的对象不只是一个Stack、一个线程获取的对象，因此这里用Map数据结构，分别用于回收保存不同线程获取的不同的对象。

然后以当前Stack为key，得到与该Stack关联的WeakOrderQueue。

```java
Map<Stack<?>, WeakOrderQueue> delayedRecycled = DELAYED_RECYCLED.get();
WeakOrderQueue queue = delayedRecycled.get(this);
```

> 创建WeakOrderQueue

如果获取到的WeakOrderQueue为null，即表示之前未回收过该Stack的DefaultHandle对象，则先创建WeakOrderQueue实例。

```java
if (delayedRecycled.size() >= maxDelayedQueues) { // 达到最大WeakOrderQueue个数，则放弃对象回收
    // Add a dummy queue so we know we should drop the object
    delayedRecycled.put(this, WeakOrderQueue.DUMMY);
    return;
}
```

创建WeakOrderQueue之前先判断上述的Map容量是否已达到最大值maxDelayedQueues，如果达到最大值，则直接保存一个WeakOrderQueue.DUMMY value。以后只要是该Stack的DefaultHandle对象在该线程回收时直接丢弃掉。

```java
else if (queue == WeakOrderQueue.DUMMY) {
    // drop object
    return;
}
```

如果Map容量未达到最大值maxDelayedQueues，则创建WeakOrderQueue并保存到delayedRecycled中：

```java
if ((queue = WeakOrderQueue.allocate(this, thread)) == null) { // 创建WeakOrderQueue
    // drop object
    return;
}
delayedRecycled.put(this, queue); // 创建的WeakOrderQueue保存到Map

static WeakOrderQueue allocate(Stack<?> stack, Thread thread) {
    // We allocated a Link so reserve the space
    // 创建WeakOrderQueue的时候判断是否还能分配LINK_CAPACITY空间，如果不能，直接返回null
    return reserveSpace(stack.availableSharedCapacity, LINK_CAPACITY)
            ? new WeakOrderQueue(stack, thread) : null;
}
```

调用WeakOrderQueue.allocate()方法创建WeakOrderQueue时，先判断是否还能分配LINK_CAPACITY空间：

```java
private static boolean reserveSpace(AtomicInteger availableSharedCapacity, int space) {
    assert space >= 0;
    for (;;) {
        int available = availableSharedCapacity.get();
        if (available < space) { // 可用空间不够
            return false;
        }
        if (availableSharedCapacity.compareAndSet(available, available - space)) { // CAS
            return true;
        }
    }
}
```

如果可以分配的空间余量availableSharedCapacity小于LINK_CAPACITY(16)，则直接返回null，则本次回收的DefaultHandle直接丢弃。如果可以分配空间，则通过WeakOrderQueue构造器创建对象:

```java
private WeakOrderQueue(Stack<?> stack, Thread thread) {
    head = tail = new Link(); // 分配一个Link
    owner = new WeakReference<Thread>(thread);
    synchronized (stack) { // 头插法插入Stack的WeakOrderQueue列表
        next = stack.head;
        stack.head = this;
    }

    availableSharedCapacity = stack.availableSharedCapacity;
}
```

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Recycler-2.jpg)

WeakOrderQueue是由一个个的Link对象链接而成的，WeakOrderQueue初始化时会创建一个Link对象：

```java
private static final class Link extends AtomicInteger {
    private final DefaultHandle<?>[] elements = new DefaultHandle[LINK_CAPACITY]; // 16

    private int readIndex; // 异线程回收时用
    private Link next; // 指向下一个Link对象
}
```

Link对象默认可以保存LINK_CAPACITY(16)个DefaultHandle实例。在创建WeakOrderQueue的过程中，同时需要将该WeakOrderQueue与Stack绑定。

```java
synchronized (stack) { // 头插法插入Stack的WeakOrderQueue列表
    next = stack.head;
    stack.head = this;
}
```

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Recycler-3.png)

可以看到，Stack通过head指针与异线程的WeakOrderQueue进行绑定。head指向的是一个WeakOrderQueue的链表。这些链表中的WeakOrderQueue，是不同异线程在回收Stack对应线程获取的对象时，用于保存这些对象创建的。这样子的一个链表，便于后续Stack从异线程的WeakOrderQueue收割对象，即DefaultHandle。

在获取到或创建WeakOrderQueue之后，就将本次异线程回收的DefaultHandle保存到该queue中。

> 将对象保存到WeakOrderQueue

```java
queue.add(item); // DefaultHandle保存到WeakOrderQueue

void add(DefaultHandle<?> handle) { // handle暂时存放到WeakOrderQueue
    handle.lastRecycledId = id;

    Link tail = this.tail; // 添加到尾部Link
    int writeIndex;
    if ((writeIndex = tail.get()) == LINK_CAPACITY) { // 尾部Link已经没有空间可以存放handle了
        // 无空间可以分配了，直接丢弃handle
        if (!reserveSpace(availableSharedCapacity, LINK_CAPACITY)) { 
            // Drop it.
            return;
        }
        // We allocate a Link so reserve the space
        this.tail = tail = tail.next = new Link(); // 尾部新增Link

        writeIndex = tail.get();
    }
    tail.elements[writeIndex] = handle;
    handle.stack = null; // 回收完毕，stack置为null
    tail.lazySet(writeIndex + 1); // 增加元素计数
}
```

保存对象是直接在WeakOrderQueue尾部Link插入的。保存时，先判断tail Link对象是否已经没有空间可以存放handle了，如果tail Link已满，则先判断是否还有空间可以分配Link对象：

```java
if (!reserveSpace(availableSharedCapacity, LINK_CAPACITY)) { // 无空间可以分配了，直接丢弃handle
    // Drop it.
    return;
}
```

如果没有空间可以分配Link了，直接丢弃本次回收的对象。否则，创建一个Link对象并追加到tail。

如果tail Link有空间或者新分配了一个Link对象，那么保存回收的DefaultHandle对象。

```java
tail.elements[writeIndex] = handle;
handle.stack = null; // 回收完毕，stack置为null
tail.lazySet(writeIndex + 1); // 增加元素计数
```

至此，异线程回收对象的过程分析完毕。

##### 3.异线程收割对象

在分析对象获取的时候，可以知道当Stack没有缓存的DefaultHandle时，会调用`scavenge()`方法，从其他线程的WeakOrderQueue回收DefaultHandle。下面就来分析该异线程收割对象的过程。

```java
DefaultHandle<T> pop() { // 获取对象
    int size = this.size;
    if (size == 0) {
        if (!scavenge()) { // 从其他线程的WeakOrderQueue回收DefaultHandle
            return null;
        }
        size = this.size;
    }
    // ...
}
```

```java
// 从其他线程的WeakOrderQueue转移handle(异线程收割对象)
boolean scavenge() {
    // continue an existing scavenge, if any
    if (scavengeSome()) {
        return true;
    }

    // reset our scavenge cursor 重置指针
    prev = null;
    cursor = head;
    return false;
}
```

可以知道，在调用scavenge()方法时，其最终调用到了scavengeSome()方法从其他线程的WeakOrderQueue中转移DefaultHandle对象。

```java
boolean scavengeSome() {
    WeakOrderQueue cursor = this.cursor;
    if (cursor == null) {
        cursor = head;
        if (cursor == null) {
            return false;
        }
    }

    boolean success = false; // 是否成功从其他线程的WeakOrderQueue回收到对象
    WeakOrderQueue prev = this.prev;
    do {
        // 从cursor指向的WeakOrderQueue收割对象到当前Stack，每次transfer，只从一个Link回收对象
        if (cursor.transfer(this)) {
            success = true;
            break;
        }

        WeakOrderQueue next = cursor.next;
        if (cursor.owner.get() == null) { // 弱引用，表示owner线程已经GC
            if (cursor.hasFinalData()) {
                for (;;) { // 不断transfer Link对象
                    if (cursor.transfer(this)) { // 每次传输一个Link的handle对象到当前Stack
                        success = true;
                    } else { // 返回false，表示传输完毕
                        break;
                    }
                }
            }
            if (prev != null) {
                prev.next = next; // cursor WeakOrderQueue对应的线程已经GC，则跳过该WeakOrderQueue
            }
        } else {
            prev = cursor;
        }

        cursor = next;

    } while (cursor != null && !success);

    this.prev = prev;
    this.cursor = cursor;
    return success;
}
```

scavengeSome()方法的执行流程可以用下面这幅图表示:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Recycler-4.png)

Stack使用了head、prev、cursor三个指针指向了WeakOrderQueue链表。通过该链表就可以将其他线程保存的对象收割到Stack中。从scavengeSome()方法可以知道，只要从cursor指向的WeakOrderQueue中转移DefaultHandle对象成功，就返回true。下面看下具体的转移过程`cursor.transfer(this)`:

```java
// 将WeakOrderQueue中的handle对象转移至Stack，每次转移一个Link
boolean transfer(Stack<?> dst) {
    Link head = this.head;
    if (head == null) {
        return false;
    }

    if (head.readIndex == LINK_CAPACITY) {
        if (head.next == null) { // 没有Link对象了
            return false;
        }
        this.head = head = head.next; // 更新head
    }

    final int srcStart = head.readIndex;
    int srcEnd = head.get();
    final int srcSize = srcEnd - srcStart;
    if (srcSize == 0) { // Link无元素，返回false
        return false;
    }

    final int dstSize = dst.size;
    final int expectedCapacity = dstSize + srcSize;

    if (expectedCapacity > dst.elements.length) {
        final int actualCapacity = dst.increaseCapacity(expectedCapacity); // 扩容
        // actualCapacity - dstSize: 当前可以传输的对象个数
        srcEnd = min(srcStart + actualCapacity - dstSize, srcEnd);
    }

    if (srcStart != srcEnd) {
        final DefaultHandle[] srcElems = head.elements;
        final DefaultHandle[] dstElems = dst.elements;
        int newDstSize = dstSize;
        for (int i = srcStart; i < srcEnd; i++) {
            DefaultHandle element = srcElems[i];
            if (element.recycleId == 0) { // 未被回收过
                element.recycleId = element.lastRecycledId;
            } else if (element.recycleId != element.lastRecycledId) {
                throw new IllegalStateException("recycled already");
            }
            srcElems[i] = null;

            if (dst.dropHandle(element)) { // 判断是否丢弃该handle
                // Drop the object.
                continue;
            }
            element.stack = dst; // 绑定Stack
            dstElems[newDstSize ++] = element;
        }

        if (srcEnd == LINK_CAPACITY && head.next != null) {
            // Add capacity back as the Link is GCed.
            reclaimSpace(LINK_CAPACITY); // 释放可分配空间

            this.head = head.next;
        }

        head.readIndex = srcEnd; // 读到srcEnd
        if (dst.size == newDstSize) { // 未收割任何DefaultHandle
            return false;
        }
        dst.size = newDstSize;
        return true;
    } else {
        // The destination stack is full already. Stack已满
        return false;
    }
}
```

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/Recycler-5.png)

从transfer()方法可知，每次调用该方法只会转移一个Link节点的对象。transfe r()方法逻辑比较简单，这里不再详述。

如果从其他线程的WeakOrderQueue收割DefaultHandle对象成功，则`scavenge()`方法返回true。此时获取对象的pop()方法中，就可以获取到已回收的对象。

```java
DefaultHandle<T> pop() {
    int size = this.size;
    if (size == 0) {
        if (!scavenge()) { // 从其他线程的WeakOrderQueue回收DefaultHandle
            return null;
        }
        size = this.size;
    }
    size --;
    DefaultHandle ret = elements[size];
    elements[size] = null;
    if (ret.lastRecycledId != ret.recycleId) {
        throw new IllegalStateException("recycled multiple times");
    }
    ret.recycleId = 0;
    ret.lastRecycledId = 0;
    this.size = size;
    return ret;
}
```

至此，异线程收割对象分析完毕。

### 参考文章

- [Netty-对象池(Recycler)](https://www.jianshu.com/p/5072058ba324)