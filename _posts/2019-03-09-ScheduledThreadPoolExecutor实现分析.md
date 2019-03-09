---
layout:     post
title:      ScheduledThreadPoolExecutor实现分析
subtitle:   线程池延迟执行和周期性执行任务
date:       2019-03-09
author:     Jay
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# ScheduledThreadPoolExecutor实现分析

上篇文章[线程池ThreadPoolExecutor实现分析](https://xuanjian1992.top/2019/03/08/%E7%BA%BF%E7%A8%8B%E6%B1%A0ThreadPoolExecutor%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90/)已经分析了ThreadPoolExecutor的实现，本篇详细分析ScheduledThreadPoolExecutor的实现原理。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/ScheduledThreadPoolExecutor%E7%B1%BB%E5%9B%BE.png?x-oss-process=style/markdown-pic)

ScheduledThreadPoolExecutor，继承ThreadPoolExecutor且实现了ScheduledExecutorService接口，它就相当于提供了“延迟执行”和“周期执行”功能的ThreadPoolExecutor。在JDK API中是这样定义它的：一种ThreadPoolExecutor，它可另行安排在给定的延迟后运行命令，或者定期执行命令。与Timer相比，它可运行多个工作线程；或者在要求ThreadPoolExecutor具有额外的灵活性或功能时，可以使用它。

在分析ScheduledThreadPoolExecutor之前，首先介绍它的两个内部类ScheduledFutureTask以及DelayedWorkQueue。这两个类是ScheduledThreadPoolExecutor实现的基础。

### 一、ScheduledFutureTask分析

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/ScheduledFutureTask%E7%B1%BB%E5%9B%BE.png?x-oss-process=style/markdown-pic)

ScheduledFutureTask继承于FutureTask，实现了RunnableScheduledFuture接口。首先它是一个任务，能够被执行，执行完之后能获取执行结果；其次，实现了RunnableScheduledFuture接口，即实现了ScheduledFuture、Delayed接口。Delayed接口用来标记延迟动作的对象，`Delayed.getDelay(TimeUnit unit)`方法能够获取动作执行之前的剩余时间。同时，RunnableScheduledFuture接口定义了`isPeriodic()`方法，可表示该接口的实现是否是周期性的。综上，可见ScheduledFutureTask也是能够延迟和周期执行的任务实现。

下面分析该类的实现:

```java
private class ScheduledFutureTask<V>
            extends FutureTask<V> implements RunnableScheduledFuture<V> {

        // 序号，表示ScheduledFutureTask入任务队列的顺序。在各个延迟任务比较执行顺序时(见compareTo
        // 方法)，如果任务执行时间time相同，则用该序号确定小序号(先进队列)对应的任务会先执行，即FIFO。
        private final long sequenceNumber;

        // 任务执行时间(纳秒为单位，绝对时间)
        private long time;

        // 任务执行周期。正数:fixed-rate execution；负数: fixed-delay execution；0: 非周期任务
        private final long period;

        // 周期性任务执行完后，需重新入任务队列等待执行。由于ScheduledFutureTask任务在执行时，可能会
        // 被修改或装饰为其他对象，因此该变量用于记住这个修改或装饰后的对象，因为入队列的是修改或装饰后的对
        // 象，见schedule、decorateTask方法说明。
        RunnableScheduledFuture<V> outerTask = this;

        // 延迟的任务队列是基于数组的堆实现(小顶堆)，该变量表示堆中该任务的索引，用于取消任务时快速索
        // 引该任务。
        int heapIndex;

         // 创建一个只执行一次的任务，ns表示触发时间(纳秒)
        ScheduledFutureTask(Runnable r, V result, long ns) {
            super(r, result);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

         // 创建一个周期性任务，ns表示触发时间，period表示周期
        ScheduledFutureTask(Runnable r, V result, long ns, long period) {
            super(r, result);
            this.time = ns;
            this.period = period;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

         // 创建一个只执行一次的任务，ns表示触发时间
        ScheduledFutureTask(Callable<V> callable, long ns) {
            super(callable);
            this.time = ns;
            this.period = 0;
            this.sequenceNumber = sequencer.getAndIncrement();
        }

		// 任务还剩多少时间会执行
        public long getDelay(TimeUnit unit) {
            return unit.convert(time - now(), NANOSECONDS);
        }

		// 任务之间执行顺序(优先级)的比较，该方法在任务队列中任务上浮和下沉时用到(小顶堆)
        public int compareTo(Delayed other) {
            if (other == this) // 同一对象，返回0
                return 0;
            if (other instanceof ScheduledFutureTask) { // other也是ScheduledFutureTask实例
                ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
                long diff = time - x.time; // 直接比较执行时间
                if (diff < 0)
                    return -1;
                else if (diff > 0)
                    return 1;
                // 执行时间相同，则比较任务进入队列的顺序
                else if (sequenceNumber < x.sequenceNumber) 
                    return -1;
                else
                    return 1;
            }
            // 类型不同，根据getDelay结果比较
            long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
            return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
        }

        // 是否周期执行，不为0即为周期执行任务
        public boolean isPeriodic() {
            return period != 0;
        }

        // 周期任务设置下次执行的时间time
        private void setNextRunTime() {
            long p = period; 
            if (p > 0) // fix-rate，直接time = time + p
                time += p;
            else // fix-delay，调用triggerTime计算
                time = triggerTime(-p); // p < 0
        }
        ///----------ScheduledThreadPoolExecutor方法-----------///
        // 计算触发时间
        long triggerTime(long delay) {
            return now() +
                ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
    	}
        private long overflowFree(long delay) {
            Delayed head = (Delayed) super.getQueue().peek();
            if (head != null) {
                long headDelay = head.getDelay(NANOSECONDS);
                // headDelay < 0表示任务过期了，但还没移出队列
                if (headDelay < 0 && (delay - headDelay < 0)) 
                	// 表示任务之间比较优先级时可能溢出，因此将delay限制最大为Long.MAX_VALUE
                    delay = Long.MAX_VALUE + headDelay; // 调整delay
            }
            return delay;
        }
        ///----------ScheduledThreadPoolExecutor方法-----------///

		// 取消任务执行
        public boolean cancel(boolean mayInterruptIfRunning) {
            boolean cancelled = super.cancel(mayInterruptIfRunning);
            // 根据removeOnCancel参数决定在取消任务后，是否从队列移除该任务，默认不移除，等到任务过
            // 期后移除
            if (cancelled && removeOnCancel && heapIndex >= 0)
                remove(this);
            return cancelled;
        }

        // 覆盖FutureTask.run方法，为了重置任务和重新入队列(如果是周期性任务)
        public void run() {
            boolean periodic = isPeriodic(); // 是否周期性任务
            // 在当前运行状态和periodic参数下，是否能运行任务
            if (!canRunInCurrentRunState(periodic)) 
                cancel(false); // 不能，则取消任务的执行
            else if (!periodic) 
                ScheduledFutureTask.super.run(); // 非周期任务，执行一次
            else if (ScheduledFutureTask.super.runAndReset()) { // 周期任务执行完后重置
                setNextRunTime(); // 设置下次执行时间
                reExecutePeriodic(outerTask); // 重新将任务放入队列
            }
        }
        ///----------ScheduledThreadPoolExecutor方法和属性-----------///
        // 线程池关闭时，能否继续执行周期性任务，默认false
        private volatile boolean continueExistingPeriodicTasksAfterShutdown;
        // 线程池关闭时，能否继续执行延迟任务，默认true
        private volatile boolean executeExistingDelayedTasksAfterShutdown = true;
        
        // 在当前运行状态和periodic参数下，是否能运行任务
        boolean canRunInCurrentRunState(boolean periodic) {
            return isRunningOrShutdown(periodic ?
                                       continueExistingPeriodicTasksAfterShutdown :
                                       executeExistingDelayedTasksAfterShutdown);
    	}
    	///----------ScheduledThreadPoolExecutor方法-----------///
    	
    	///----------ThreadPoolExecutor方法-----------///
    	// 线程池为RUNNING状态或SHUTDOWN状态且允许true，则返回true
        final boolean isRunningOrShutdown(boolean shutdownOK) {
            int rs = runStateOf(ctl.get());
            return rs == RUNNING || (rs == SHUTDOWN && shutdownOK);
        }
        ///----------ThreadPoolExecutor方法-----------///
    }
```

ScheduledFutureTask中包含了周期性任务和延迟执行一次的任务的实现逻辑。同时，在任务运行之前，需要根据

策略`canRunInCurrentRunState(periodic)`确定是否取消任务的执行。

### 二、DelayedWorkQueue分析

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/DelayedWorkQueue%E7%B1%BB%E5%9B%BE.png?x-oss-process=style/markdown-pic)

DelayedWorkQueue是ScheduledThreadPoolExecutor任务队列的实现，它是基于小顶堆实现的，而小顶堆基于数组实现。DelayedWorkQueue是java.util.concurrent.DelayQueue的特殊版本，原理类似。

以下分析其实现:

```java
	static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {

        private static final int INITIAL_CAPACITY = 16; // 堆数组的初始大小
        private RunnableScheduledFuture<?>[] queue =
            new RunnableScheduledFuture<?>[INITIAL_CAPACITY]; // 堆的数组
        private final ReentrantLock lock = new ReentrantLock(); // 互斥锁
        private int size = 0; // 元素个数

        // 当前等待获取过期任务的leader线程
        // 1.当有线程已成为leader时，该线程是限时等待任务，其他follower线程是无限期的等待任务。
        // 2.当leader获取到任务去执行时(从take()、poll()返回)，它必须唤醒其他等待的线程，
        // 除非其他线程成为了新leader(队列头部任务被具有更早过期时间的任务替换)
        // 3.当堆顶任务(队列头部任务)被具有更早过期时间的任务替换时，leader属性置为null，
        // 唤醒任何等待的线程(原leader或followers)
        // 4.任何等待的线程都有可能获取和失去leader资格
        private Thread leader = null;

        // 队列中是否存在任务的Condition对象
        private final Condition available = lock.newCondition();

        // 如果f是ScheduledFutureTask，则设置其heapIndex堆索引
        private void setIndex(RunnableScheduledFuture<?> f, int idx) {
            if (f instanceof ScheduledFutureTask)
                ((ScheduledFutureTask)f).heapIndex = idx;
        }

        // 索引为k位置的堆元素key上浮操作
        private void siftUp(int k, RunnableScheduledFuture<?> key) {
            while (k > 0) {
                int parent = (k - 1) >>> 1; // 父元素索引
                RunnableScheduledFuture<?> e = queue[parent];
                if (key.compareTo(e) >= 0) // 小顶堆，找到合适位置退出
                    break;
                queue[k] = e; // 更换位置
                setIndex(e, k); // 更新索引
                k = parent; // 继续上浮
            }
            queue[k] = key;// 结束
            setIndex(key, k);
        }

         // 索引为k位置的堆元素key下沉操作
        private void siftDown(int k, RunnableScheduledFuture<?> key) {
            int half = size >>> 1; // 中间元素位置
            while (k < half) {
                int child = (k << 1) + 1; // 左子元素
                RunnableScheduledFuture<?> c = queue[child]; // 左子元素值
                int right = child + 1; // 右子元素
                if (right < size && c.compareTo(queue[right]) > 0)
                    c = queue[child = right]; // 如果左子元素大于右边，则取右边较小的元素
                if (key.compareTo(c) <= 0) // 小顶堆，找到合适位置退出
                    break;
                queue[k] = c; // 更换位置
                setIndex(c, k); // 更新索引
                k = child; // 继续下沉
            } 
            queue[k] = key; // 结束
            setIndex(key, k);
        }

        // 数组扩容
        private void grow() {
            int oldCapacity = queue.length;
            int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
            if (newCapacity < 0) // overflow
                newCapacity = Integer.MAX_VALUE;
            queue = Arrays.copyOf(queue, newCapacity);
        }

         // 查找
        private int indexOf(Object x) {
            if (x != null) {
            	// ScheduledFutureTask实例，直接根据heapIndex查找
                if (x instanceof ScheduledFutureTask) { 
                    int i = ((ScheduledFutureTask) x).heapIndex;
                    if (i >= 0 && i < size && queue[i] == x) // 再次检查
                        return i;
                } else {
                    for (int i = 0; i < size; i++) // 遍历
                        if (x.equals(queue[i]))
                            return i;
                }
            }
            return -1;
        }

		// 移除x
        public boolean remove(Object x) {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                int i = indexOf(x);
                if (i < 0)
                    return false; // 不存在x，直接返回false

                setIndex(queue[i], -1); // index置为-1
                int s = --size; // 元素个数-1
                RunnableScheduledFuture<?> replacement = queue[s]; // 取尾部元素
                queue[s] = null;
                if (s != i) {
                    siftDown(i, replacement); // 先尝试下沉
                    if (queue[i] == replacement) // 如果不变，再尝试上浮
                        siftUp(i, replacement);
                }
                return true;
            } finally {
                lock.unlock();
            }
        }

		// 队列顶部任务
        public RunnableScheduledFuture<?> peek() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                return queue[0];
            } finally {
                lock.unlock();
            }
        }

		// 添加任务
        public boolean offer(Runnable x) {
            if (x == null)
                throw new NullPointerException();
            RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                int i = size;
                if (i >= queue.length) // 如果目前容量已经满了，先扩容
                    grow();
                size = i + 1; // size+1
                if (i == 0) { // 第一次存入任务
                    queue[0] = e;
                    setIndex(e, 0);
                } else {
                    siftUp(i, e); // 将任务放在队尾，然后上浮
                }
                // 下面有两种情形:
                // 1.第一次存入任务，所以queue[0] == e，唤醒等待的线程
                // 2.非第一次添加任务，添加后，该任务上浮到堆顶，此时之前线程等待的队列头部任务发生了变化,
                // 不管之前线程leader与followers，直接将leader置为null，重新选择leader线程(执行
                // 唤醒操作，可能是原leader或follower线程被唤醒，成为新leader)，执行任务的获取。
                if (queue[0] == e) {
                    leader = null;
                    available.signal();
                }
            } finally {
                lock.unlock();
            }
            return true; // 因为是无界队列，所有任务总是添加成功
        }

        public void put(Runnable e) {
            offer(e);
        }

        public boolean add(Runnable e) {
            return offer(e);
        }

        public boolean offer(Runnable e, long timeout, TimeUnit unit) {
            return offer(e); // 忽略时间限制，因为添加任务总是能成功
        }

         // 获取任务f返回之前，需要做的动作
        private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
            int s = --size; // size-1
            RunnableScheduledFuture<?> x = queue[s]; // 队尾任务
            queue[s] = null;
            if (s != 0)
                siftDown(0, x); // 队尾任务放到堆顶，然后下沉，使得堆有序
            setIndex(f, -1);
            return f;
        }

		// 如果队列头部任务已过期，返回该任务；否则如果队列头部任务为空，或者未过期，返回null
        public RunnableScheduledFuture<?> poll() {
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                RunnableScheduledFuture<?> first = queue[0];
                if (first == null || first.getDelay(NANOSECONDS) > 0)
                    return null;
                else
                    return finishPoll(first);
            } finally {
                lock.unlock();
            }
        }

		// 阻塞式的获取任务
        public RunnableScheduledFuture<?> take() throws InterruptedException {
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    RunnableScheduledFuture<?> first = queue[0]; // 队列头部任务
                    if (first == null)
                        available.await(); // 队头任务为空，直接等待
                    else {
                        long delay = first.getDelay(NANOSECONDS); // 获取任务执行还需要等待的时间
                        if (delay <= 0)
                            return finishPoll(first); // 任务已过期，可以直接取出、返回
                        first = null; // 线程在等待时不能持有队头任务的引用，防止内存泄漏
                        if (leader != null) // leader线程已存在，该线程直接等待
                         	// 等待线程可能被leader线程唤醒，也可能被添加任务的线程唤醒(选择新leader)
                            available.await();
                        else {
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread; // 当前线程成为leader线程
                            try {
                                available.awaitNanos(delay); // 限时等待
                            } finally {
                            	// 1.该leader线程限时等待完成，可以获取队头过期任务返回了，
                            	// 并将leader置为null
                            	// 2.有可能等待期间队头任务发生变化(新任务添加)，此时
                            	// leader != thisThread(leader重新选择)
                                if (leader == thisThread) 
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
            	// 如果leader线程为null且队头任务不为空，唤醒其中一个等待线程
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }

		// 限时获取任务
        public RunnableScheduledFuture<?> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
            long nanos = unit.toNanos(timeout); // 最多等待的时间(纳秒)
            final ReentrantLock lock = this.lock;
            lock.lockInterruptibly();
            try {
                for (;;) {
                    RunnableScheduledFuture<?> first = queue[0];
                    if (first == null) { 
                        if (nanos <= 0) // 任务为空且已超时，则直接返回null
                            return null;
                        else // 任务为空且未超时，则等待nanos时间
                            nanos = available.awaitNanos(nanos);
                    } else {
                        long delay = first.getDelay(NANOSECONDS);// 获取任务执行还需要等待的时间
                        if (delay <= 0)
                            return finishPoll(first); // 任务已过期，可以直接取出、返回
                        if (nanos <= 0)  // 已超时，则直接返回null
                            return null;
                        first = null; // 线程在等待时不能持有队头任务的引用，防止内存泄漏
                        // leader线程已存在或者最多等待的时间小于delay，该线程直接等待nanos时间
                        if (nanos < delay || leader != null)
                        	// 等待线程可能被leader线程唤醒，也可能被添加任务的线程唤醒(选择新leader)
                            nanos = available.awaitNanos(nanos);
                        else {
                            Thread thisThread = Thread.currentThread();
                            leader = thisThread; // 当前线程成为leader线程
                            try {
                            	// 等待delay时间
                                long timeLeft = available.awaitNanos(delay);
                                nanos -= delay - timeLeft; // 还可以等待的时间
                            } finally {
                           		// 1.该leader线程限时等待完成，可以获取队头过期任务返回了，
                            	// 并将leader置为null
                            	// 2.有可能等待期间队头任务发生变化(新任务添加)，此时
                            	// leader != thisThread(leader重新选择)
                                if (leader == thisThread)
                                    leader = null;
                            }
                        }
                    }
                }
            } finally {
            	// 如果leader线程为null且队头任务不为空，唤醒其中一个等待线程
                if (leader == null && queue[0] != null)
                    available.signal();
                lock.unlock();
            }
        }

       // 如果队头任务已过期，返回队头任务，否则返回null
        private RunnableScheduledFuture<?> peekExpired() {
            // assert lock.isHeldByCurrentThread();
            RunnableScheduledFuture<?> first = queue[0];
            // 如果不存在任务或队头任务还不能执行，返回null
            return (first == null || first.getDelay(NANOSECONDS) > 0) ?
                null : first;
        }

		// 转移任务到集合c
        public int drainTo(Collection<? super Runnable> c) {
            if (c == null)
                throw new NullPointerException();
            if (c == this)
                throw new IllegalArgumentException();
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                RunnableScheduledFuture<?> first;
                int n = 0;
                // 已过期的任务才能转移
                while ((first = peekExpired()) != null) {
                    c.add(first);   // In this order, in case add() throws.
                    finishPoll(first);
                    ++n;
                }
                return n;
            } finally {
                lock.unlock();
            }
        }
		// 转移任务到集合c，最多maxElements个
        public int drainTo(Collection<? super Runnable> c, int maxElements) {
            if (c == null)
                throw new NullPointerException();
            if (c == this)
                throw new IllegalArgumentException();
            if (maxElements <= 0)
                return 0;
            final ReentrantLock lock = this.lock;
            lock.lock();
            try {
                RunnableScheduledFuture<?> first;
                int n = 0;
                while (n < maxElements && (first = peekExpired()) != null) {
                    c.add(first);   // In this order, in case add() throws.
                    finishPoll(first);
                    ++n;
                }
                return n;
            } finally {
                lock.unlock();
            }
        }
    }
```

在`take()`和`poll(long timeout, TimeUnit unit)`方法中有如下代码:

> first = null; // 线程在等待时不能持有队头任务的引用，防止内存泄漏

因为存在多线程获取队头任务的情况，如果其中一个线程拿到队头任务返回并执行完任务，该任务对象应该被GC回收；线程若没有将first置为null，则其他线程引用该队头任务，导致该任务无法被回收。因此，这步操作是必要的。

### 三、ScheduledThreadPoolExecutor分析

在分析了以上两个类的实现之后，下面开始分析ScheduledThreadPoolExecutor的实现逻辑。

##### 1.构造器

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}

public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue(), threadFactory);
}

public ScheduledThreadPoolExecutor(int corePoolSize,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue(), handler);
}

public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue(), threadFactory, handler);
}
```

从以上四个构造函数可以看出，ScheduledThreadPoolExecutor使用了无界的DelayedWorkQueue，并且可以指定corePoolSize、threadFactory及handler。由于任务队列无界，因此maximumPoolSize、keepAliveTime参数无效。

##### 2.ScheduledExecutorService接口实现

由于ScheduledThreadPoolExecutor实现了ScheduledExecutorService接口，下面看下ScheduledExecutorService接口的实现。

- schedule(Callable callable, long delay, TimeUnit unit) :创建并执行在给定延迟后启用的 ScheduledFuture。

- schedule(Runnable command, long delay, TimeUnit unit) :创建并执行在给定延迟后启用的一次性操作。

- scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) :创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期；也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推。

- scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) :创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。

```java
	// 延迟执行一个Runnable任务
	public ScheduledFuture<?> schedule(Runnable command,
                                       long delay,
                                       TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        // 新建ScheduledFutureTask实例，装饰或修改为RunnableScheduledFuture实例
        RunnableScheduledFuture<?> t = decorateTask(command,
            new ScheduledFutureTask<Void>(command, null,
                                          triggerTime(delay, unit)));
        delayedExecute(t); // 放入队列执行(RunnableScheduledFuture实例)
        return t;
    }

	// 延迟执行一个Callable任务
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,
                                           long delay,
                                           TimeUnit unit) {
        if (callable == null || unit == null)
            throw new NullPointerException();
        // 新建ScheduledFutureTask实例，装饰或修改为RunnableScheduledFuture实例
        RunnableScheduledFuture<V> t = decorateTask(callable,
            new ScheduledFutureTask<V>(callable,
                                       triggerTime(delay, unit)));
        delayedExecute(t); // 放入队列执行(RunnableScheduledFuture实例)
        return t;
    }

	// 按照计算的时间点执行周期性任务
	// initialDelay，initialDelay+period，initialDelay+2period...之后执行
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,
                                                  long initialDelay,
                                                  long period,
                                                  TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (period <= 0) // 周期必须大于0
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft =  // 新建ScheduledFutureTask实例
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(period));
        // 装饰或修改为RunnableScheduledFuture实例
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t; // outerTask置为t，任务一次执行完之后，重新添加到队列时使用
        delayedExecute(t); // 放入队列执行(RunnableScheduledFuture实例)
        return t;
    }

	// 按照固定的间隔delay执行周期性任务，一次执行完成与下一次执行开始之间的间隔相同
    public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                     long initialDelay,
                                                     long delay,
                                                     TimeUnit unit) {
        if (command == null || unit == null)
            throw new NullPointerException();
        if (delay <= 0) // 周期必须大于0
            throw new IllegalArgumentException();
        ScheduledFutureTask<Void> sft = // 新建ScheduledFutureTask实例
            new ScheduledFutureTask<Void>(command,
                                          null,
                                          triggerTime(initialDelay, unit),
                                          unit.toNanos(-delay));
        // 装饰或修改为RunnableScheduledFuture实例
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t; // outerTask置为t，任务一次执行完之后，重新添加到队列时使用
        delayedExecute(t); // 放入队列执行(RunnableScheduledFuture实例)
        return t;
    }
```

以上是四个方法的实现，用到了以下四个方法:

```java
// 装饰或修改为RunnableScheduledFuture实例
protected <V> RunnableScheduledFuture<V> decorateTask(
    Runnable runnable, RunnableScheduledFuture<V> task) {
    return task;
}
// 装饰或修改为RunnableScheduledFuture实例
protected <V> RunnableScheduledFuture<V> decorateTask(
    Callable<V> callable, RunnableScheduledFuture<V> task) {
    return task;
}

// 将延迟任务放入队列执行
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown()) // 添加任务时，线程池已关闭，则拒绝任务
        reject(task);
    else {
        super.getQueue().add(task); // 添加到队列
        if (isShutdown() && // 再次检查线程池状态，若已关闭，且关闭后任务不能继续执行，则从队列移除任务
            !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false); // 取消执行任务
        else
            ensurePrestart(); // 确保有工作线程启动，等待任务
    }
}

// 确保有工作线程启动，等待任务
void ensurePrestart() {
    int wc = workerCountOf(ctl.get()); // 目前的工作线程数
    if (wc < corePoolSize) // 小于核心线程数，则添加一个核心线程
        addWorker(null, true);
    else if (wc == 0) // 这种情况是即使核心线程数为0，也要保证一个线程是启动的
        addWorker(null, false);
}
```

##### 3.AbstractExecutorService.submit方法重写

```java
public Future<?> submit(Runnable task) {
    return schedule(task, 0, NANOSECONDS);
}

public <T> Future<T> submit(Runnable task, T result) {
    return schedule(Executors.callable(task, result), 0, NANOSECONDS);
}

public <T> Future<T> submit(Callable<T> task) {
    return schedule(task, 0, NANOSECONDS);
}
```

从以上方法的实现可以看出，ScheduledThreadPoolExecutor实现时是直接调用`schedule(Runnable command, long delay, TimeUnit unit)`和`schedule(Callable<V> callable, long delay, TimeUnit unit)`两个方法，延迟设置为0，即立即执行。

##### 4.Executor接口实现

```java
public void execute(Runnable command) {
    schedule(command, 0, NANOSECONDS);
}
```

该方法也是调用了`schedule(Runnable command, long delay, TimeUnit unit)`方法，延迟为0。

### 四、总结

本文详细分析了ScheduledThreadPoolExecutor的实现，在此之前分析了ScheduledFutureTask、DelayedWorkQueue两个类的实现机制。由于ScheduledThreadPoolExecutor提供了“延迟执行”和“周期执行”的功能，这在定时任务的执行等方面有很大的用处。

### 参考文献

- [JDK ScheduledThreadPoolExecutor](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ScheduledThreadPoolExecutor.html)
- [【死磕Java并发】—–J.U.C之线程池：ScheduledThreadPoolExecutor](http://cmsblogs.com/?p=2451)