---
layout:     post
title:      线程池ThreadPoolExecutor实现分析
subtitle:   Java线程池核心实现机制解密
date:       2019-03-08
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# 线程池ThreadPoolExecutor实现分析 #

合理使用线程池对线程进行管理的好处

- 降低资源消耗
- 提高响应速度
- 提高线程的可管理性

### 一、Executor执行框架

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/ThreadPoolExecutor%E7%BB%A7%E6%89%BF%E5%85%B3%E7%B3%BB.png)

**1.Executor接口**

Executor存在的目的是提供一种将"任务提交"与"任务如何运行"分离开来的机制。

**2.ExecutorService接口**

扩展了Exector接口，添加了一些用来管理执行器生命周期和任务生命周期的方法，如submit等。

**3.AbstractExecutorService抽象类**

AbstractExecutorService存在的目的是为ExecutorService中的函数接口提供了默认实现。

**4.ThreadPoolExecutor线程池**

**5.ScheduledExecutorService接口**

继承于于ExecutorService，相当于是提供了"延时"和"周期执行"功能的ExecutorService。

**6.ScheduledThreadPoolExecutor**

ScheduledThreadPoolExecutor继承于ThreadPoolExecutor，并且实现了ScheduledExecutorService接口，相当于是提供了"延时"和"周期执行"功能的ScheduledExecutorService。

**7.Executors**

Executors是个静态工厂类,通过静态工厂方法返回ExecutorService、ScheduledExecutorService、ThreadFactory 和 Callable 等类的对象。
（1）初始化一个指定线程数的线程池，其中corePoolSize = maximumPoolSize，使用LinkedBlockingQuene作为阻塞队列，不过当线程池没有可执行任务时，也不会释放线程资源。

```java
// Executor的静态工厂方法
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

（2）1、初始化一个可以缓存线程的线程池，默认缓存60s，线程池的线程数可达到Integer.MAX_VALUE，即2147483647，内部使用**SynchronousQueue**作为阻塞队列；
          2、和newFixedThreadPool创建的线程池不同，newCachedThreadPool在没有任务执行时，当线程的空闲时间超过keepAliveTime，会自动释放线程资源，当提交新任务时，**如果没有空闲线程，则创建新线程执行任务**，会**导致一定的系统开销**；

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

（3）初始化的线程池中只有一个线程，如果该线程异常结束，会重新创建一个新的线程继续执行任务，唯一的线程可以**保证所提交任务的顺序执行**，内部使用LinkedBlockingQueue作为阻塞队列。

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

（4）初始化的线程池可以在指定的时间内周期性的执行所提交的任务，在实际的业务场景中可以使用该线程池定期的同步数据。

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

### 二、ThreadPoolExecutor重要字段说明

**1.corePoolSize**

线程池中的核心线程数，当提交一个任务时，线程池创建一个新线程执行任务，直到当前线程数等于corePoolSize；如果当前线程数为corePoolSize，继续提交的任务被保存到阻塞队列中，等待被执行；如果执行了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有核心线程。

	public int prestartAllCoreThreads() {
	    int n = 0;
	    while (addWorker(null, true))
	        ++n;
	    return n;
	}

**2.maximumPoolSize**

线程池中允许的最大线程数。如果当前阻塞队列满了，且继续提交任务，则创建新的线程执行任务，前提是当前线程数小于maximumPoolSize。

**3.keepAliveTime**

线程空闲时的存活时间，即当线程没有任务执行时，继续存活的时间；默认情况下，该参数只在线程数大于corePoolSize时才有用；如果设置allowCoreThreadTimeOut为true，则对核心线程也有效。

**4.TimeUnit unit**

keepAliveTime的单位；

**5.workQueue**

常用的有ArrayBlockingQueue、LinkedBlockingQueue、SynchronousQuene、DelayedWorkQueue。

**6.threadFactory**

创建线程的工厂，通过自定义的线程工厂可以给每个新建的线程设置一个具有识别度的线程名。

```java
DefaultThreadFactory() {
    SecurityManager s = System.getSecurityManager();
    group = (s != null) ? s.getThreadGroup() :
                          Thread.currentThread().getThreadGroup();
    namePrefix = "pool-" +
                  poolNumber.getAndIncrement() +
                 "-thread-";
}
```


**7.RejectedExecutionHandler handler**

线程池的饱和策略，当阻塞队列满了，且没有空闲的工作线程，如果继续提交任务，必须采取一种策略处理该任务，线程池提供了4种策略：

1、AbortPolicy：直接抛出异常，默认策略；
2、CallerRunsPolicy：用调用者所在的线程来执行任务；
3、DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并重新尝试执行当前任务；
4、DiscardPolicy：直接丢弃任务；
当然也可以根据应用场景实现RejectedExecutionHandler接口，自定义饱和策略，如记录日志或持久化存储不能处理的任务。

### 三、线程池状态及线程数
```java
// 表示线程池状态和工作线程数量的字段
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3; // 29
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits ctl高3位存储运行状态
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; } // 得到线程池状态
private static int workerCountOf(int c)  { return c & CAPACITY; } // 得到工作线程数
private static int ctlOf(int rs, int wc) { return rs | wc; } // 得到ctl

private static boolean runStateLessThan(int c, int s) {
    return c < s;
}

private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}

private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
```

其中AtomicInteger变量ctl的功能非常强大：**利用低29位表示线程池中线程数workCount，通过高3位表示线程池的运行状态runState**：
1、RUNNING：-1 << COUNT_BITS，即高3位为111，**该状态的线程池会接收新任务，并处理阻塞队列中的任务**；
2、SHUTDOWN： 0 << COUNT_BITS，即高3位为000，**该状态的线程池不会接收新任务，但会处理阻塞队列中的任务**；(shutdwon())
3、STOP ： 1 << COUNT_BITS，即高3位为001，**该状态的线程不会接收新任务，也不会处理阻塞队列中的任务，而且会中断正在运行的任务**；(shutdownNow())
4、TIDYING ： 2 << COUNT_BITS，即高3位为010；所有的任务都终止了，workCount为0，线程池如果处于该情况下，调用terminated()方法之后，进入到 TERMINATED 状态；
5、TERMINATED： 3 << COUNT_BITS，即高3位为011。

### 四、Worker内部类

```java
private final class Worker
    extends AbstractQueuedSynchronizer // 继承自AQS
    implements Runnable
{
    final Thread thread;  // 对应线程
    
    Runnable firstTask;   // 第一个任务
    
    volatile long completedTasks; // 已完成任务数
    
	Worker(Runnable firstTask) { 
        setState(-1); // 在执行runWorker方法之前抑制中断
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

	public void run() {
        runWorker(this); // 线程执行，最终在这个方法
    }

    protected boolean isHeldExclusively() { // Worker线程是否持有了锁
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }
	
	// runWorker方法调用之后，state>=0，因此只有Worker线程启动了，才能中断
    void interruptIfStarted() { 
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

### 五、重要方法实现说明

**1.execute()**

```java
public void execute(Runnable command) {
    if (command == null)  //任务为空，抛出异常
        throw new NullPointerException();

    int c = ctl.get(); //得到工作线程数、线程池状态
    if (workerCountOf(c) < corePoolSize) { //线程池当前工作线程数小于核心池大小
        if (addWorker(command, true)) //尝试添加工作者worker，以核心池大小为界，成功，则返回
            return;
        c = ctl.get(); //线程并发运行的时候，添加worker不成功，重新得到ctl，因为线程池状态、工作线程数可能发生变化。
    }
	//工作线程数>=corePoolSize  RUNNING  && 队列未满
    if (isRunning(c) && workQueue.offer(command)) {//如果线程池运行中，则尝试添加任务到阻塞队列
        int recheck = ctl.get(); //重新检查线程池状态
		//如果线程池没运行了shutdown或者其他状态，则要抛弃任务，因此工作队列中移除任务
        if (! isRunning(recheck) && remove(command)) 
            reject(command);  //采用相应的拒绝策略拒绝任务
		//线程池处于RUNNING状态 || 线程池处于非RUNNING状态但是任务移除失败
        else if (workerCountOf(recheck) == 0) 
			//这行代码是为了SHUTDOWN状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况。
        	//添加一个null任务是因为SHUTDOWN状态下，线程池不再接受新任务
            addWorker(null, false); 
    }
	//两种情况
	//1.线程池处于RUNNING，工作线程数目已经超过了核心线程数，而且阻塞队列已经满了，则以最大线程数为边界添加worker
	//2.线程池不处于RUNNING状态，则拒绝任务
    else if (!addWorker(command, false))
        reject(command);  //不成功，就采用相应策略拒绝任务
}
```

具体的执行流程如下：
1、当前有效线程数小于核心线程数时，应该添加一个新线程addWorker(command, true)，成功则退出，否则继续；
2、再次获取ctl值(状态、线程数可能发生变化)；
3、如果线程池处于运行状态，并且当前任务成功加入到workQueue中，再次进行double-check，理由同上。如果发现线程池不再处于运行状态，而且remove成功，则需要拒绝该任务，退出;否则当前有效线程数为0时，创建一个空的线程，只是为了保证线程池在SHUTDOWN状态存在一个可用的线程去执行任务；
4、运行最后一个else则有如下情况：
①线程不处于RUNNING状态；
耳线程是RUNNING状态，但是工作线程数目已经超过了核心线程数，而且阻塞队列已经满了；
这个时候通过调用addWorker(command, false)，false表示线程上限设置为最大线程数去添加该任务。
5、如果没有正常添加，则拒绝该任务。

**2.addWorker()**

addWorker主要负责创建新的线程并执行任务。
	
```java
//添加任务与线程 core=true，则以核心池大小为边界，否则以最大池大小为边界
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c); //线程池状态

        //这条语句等价：rs >= SHUTDOWN && (rs != SHUTDOWN || firstTask != null || workQueue.isEmpty())
		// 满足下列条件则直接返回false，线程创建失败:
        // ①rs > SHUTDOWN:STOP || TIDYING || TERMINATED 此时不再接受新的任务，且所有任务执行结束
        // ②rs = SHUTDOWN:firtTask != null 此时不再接受任务，但是仍然会执行队列中的任务
        // ③rs = SHUTDOWN:firtTask == null workQueue为空----任务为null && 队列为空
		//见execute方法的addWorker(null, false)
        // 最后一种情况也就是说SHUTDONW状态下，如果队列不为空还得接着往下执行，为什么？add一个null任务目的到底是什么？
        // 看execute方法只有workCount==0的时候firstTask才会为null，结合这里的条件就是线程池SHUTDOWN了不再接受新任务，但是此时队列不为空，那么还得创建线程把任务给执行完才行。
        if (rs >= SHUTDOWN && ! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty()))
            return false;
			
		// 走到这的情形：
        // 1.线程池状态为RUNNING
        // 2.SHUTDOWN状态，但队列中还有任务需要执行
        for (;;) {
            int wc = workerCountOf(c); //当前线程数
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;  //大于容量，返回false
            if (compareAndIncrementWorkerCount(c)) //CAS递增workCount
                break retry; //操作成功，跳出重试的最外层循环，直接执行下面的启动线程代码
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)  // 如果线程池的状态发生变化则重试
                continue retry;
            // CAS失败，在内循环重试
        }
    }

	 // wokerCount递增成功
    boolean workerStarted = false; //线程启动状态
    boolean workerAdded = false;  //线程添加到set集合的状态
    Worker w = null; //worker
    try {
        w = new Worker(firstTask); //新建worker
        final Thread t = w.thread; //对应线程
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock; //可重入互斥锁
            mainLock.lock(); //并发的访问线程池workers对象必须加锁
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
				
				//RUNNING状态 || SHUTDONW状态下清理队列中剩余的任务
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // 如果该线程已经存活，则抛出异常，肯定不正常，线程还是刚创建，压根没启动
                        throw new IllegalThreadStateException();
                    workers.add(w); //将新启动的线程添加到线程池中，HashSet<Worker>
                    int s = workers.size();  //largestPoolSize 表示线程池中存在的最大线程数的情况，更新
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true; //线程池加入worker成功
                }
            } finally {
                mainLock.unlock(); //释放锁
            }
			// 启动新添加的线程，这个线程首先执行firstTask，然后不停的从队列中取任务执行
            // 当等待keepAlieTime还没有任务执行则该线程结束。见runWoker和getTask方法的代码。
            if (workerAdded) { //
                t.start(); //启动线程，runWorker()
                workerStarted = true;
            }
        }
    } finally {
		 // 线程启动失败，则从wokers中移除w并递减wokerCount
        if (! workerStarted)
			// 递减wokerCount会触发tryTerminate方法
            addWorkerFailed(w);
    }
    return workerStarted;
}

private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w); //移除
        decrementWorkerCount();
        tryTerminate(); //尝试更新线程池状态到TERMINATED状态
    } finally {
        mainLock.unlock();
    }
}
```

1、判断线程池的状态，如果线程池的状态值大于或等于SHUTDOWN，则不处理提交的任务，直接返回；
2、通过参数core判断当前需要创建的线程是否为核心线程，如果core为true，且当前线程数小于corePoolSize，则跳出循环，开始创建新的线程。
3、线程池的工作线程通过Worker类实现，在ReentrantLock锁的保证下，把Woker实例插入到HashSet后，并启动Woker中的线程。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/ThreadPoolExecutor-Worker.png)

从Worker类的构造方法实现可以发现：线程工厂在创建线程thread时，将Worker实例本身this作为参数传入，当执行start方法启动线程thread时，本质是执行了Worker的run方法(即runWorker方法)。

**3.runWorker()**

任务添加成功后实际执行的是runWorker这个方法，这个方法非常重要，简单来说它做的就是：
(a)第一次启动会执行初始化传进来的任务firstTask；
(b)然后会从workQueue中取任务执行，如果队列为空则等待keepAliveTime这么长时间。

```java
//worker线程运行的核心方法
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread(); //当前工作者线程
    Runnable task = w.firstTask; //任务
    w.firstTask = null; //使worker任务为null
    w.unlock(); // Worker的构造函数中抑制了线程中断setState(-1)，所以这里需要unlock从而允许中断
	//用于标识是否异常终止，finally中processWorkerExit的方法会有不同逻辑
    // 为true的情况：执行任务抛出异常
    boolean completedAbruptly = true;
    try {
		// 如果getTask返回null那么getTask中会将workerCount递减(正常结束)，如果异常了这个递减操作会在processWorkerExit中处理
        while (task != null || (task = getTask()) != null) { //获取任务
            // 获取独占锁，此时interruptIdleWorkers方法不能中断当前线程，因为当前线程现在忙于执行任务
            w.lock(); 
         	//rs：STOP状态(可能调用了shutdownNow方法，此时线程必须是中断状态)，线程没有被中断，则中断线程；
			// 线程已被中断，rs：STOP状态，再次检查中断状态发现没有中断，则中断线程。
            // rs不是STOP状态，必须保证当前线程不是中断状态。
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
				// 任务执行前可以插入一些处理，子类重载该方法
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    task.run();  // 执行用户任务
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {	
					// 和beforeExecute一样，留给子类去重载
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                w.completedTasks++; //当前Worker线程完成任务数加1
                w.unlock();
            }
        }
        completedAbruptly = false; //正常退出
    } finally {
        processWorkerExit(w, completedAbruptly); // 结束线程的一些清理工作
    }
}
```


执行流程：

1、线程启动之后，通过unlock方法释放锁，设置AQS的state为0，表示线程允许被中断；
2、获取第一个任务firstTask，执行任务的run方法，不过在执行任务之前，会进行加锁操作，此时interruptIdleWorkers方法不能中断当前线程，因为当前线程现在忙于执行任务，任务执行完会释放锁；
3、在执行任务的前后，可以根据业务场景自定义beforeExecute和afterExecute方法；
4、firstTask执行完成之后，通过getTask方法从阻塞队列中获取等待的任务，如果队列中没有任务，getTask方法会被阻塞并挂起，不会占用cpu资源。


**4.getTask()**
	
```java
// 如果发生了以下四件事中的任意一件，那么Worker需要被回收：
// 1. Worker个数比线程池最大大小maximumPoolSize要大
// 2. 线程池处于STOP状态
// 3. 线程池处于SHUTDOWN状态并且阻塞队列为空
// 4. 使用超时时间从阻塞队列里拿数据，并且超时之后没有拿到数据(allowCoreThreadTimeOut || workerCount > corePoolSize)

//获取任务
private Runnable getTask() {
    boolean timedOut = false; //上一次获取动作是否超时，若超时置为true,在后面程序中判断会将线程数-1
	//自循环
    for (;;) { 
        int c = ctl.get();
        int rs = runStateOf(c);

        // 1.rs > SHUTDOWN 所以rs至少等于STOP,这时不再处理队列中的任务
        // 2.rs = SHUTDOWN 所以rs>=STOP肯定不成立，这时还需要处理队列中的任务，除非队列为空
        // 这两种情况都会返回null让runWoker退出while循环也就是当前线程结束了，所以必须要decrementWokerCount
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount(); //递减workerCount值
            return null;
        }
		
		// 1.RUNING状态
        // 2.SHUTDOWN状态，但队列中还有任务需要执行
        int wc = workerCountOf(c);  //当前工作线程数

        // 标记从队列中取任务时是否设置超时时间
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
		
		//1.Worker个数比线程池最大大小mmaximumPoolSize要大。即调用了setMaximumPoolSize，调整了最大线程数配置，使得当前值大于maximumPoolSize，因此空闲线程需要退出。
		//2.使用超时时间从阻塞队列里拿数据，并且超时之后没有拿到数据(allowCoreThreadTimeOut || workerCount > corePoolSize)
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))  //尝试将workCount减一
                return null; //成功，返回null
            continue; //失败，则重试
        }

        try {
			// 从阻塞队列中获取任务，可能是需要设置超时，如果一旦设置时间超时，则需要在规定的时间内获取到任务
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r; //任务不为空，直接返回
			//如果没有有效的时间内获取到任务，设置这个timeOut为true
            timedOut = true; 
        } catch (InterruptedException retry) {
            timedOut = false; //workQueue.take();处阻塞时线程被中断，重试 
        }
    }
}
```

整个getTask操作在自旋下完成：
1、workQueue.take：如果阻塞队列为空，当前线程会被挂起等待；当队列中有任务加入时，线程被唤醒，take方法返回任务，并执行；
2、workQueue.poll：如果在keepAliveTime时间内，阻塞队列还是没有任务，则返回null；

所以，线程池中实现的线程可以一直执行由用户提交的任务。

**5.processWorkerExit**



```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
	// 正常的话，在runWorker的getTask方法workerCount已经被减一了;任务执行异常，会在这里减一
    if (completedAbruptly) // 任务执行异常
        decrementWorkerCount(); //-1

    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
		// 累加线程的completedTasks
        completedTaskCount += w.completedTasks;
		// 从线程池中移除超时或者出现异常的线程
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    tryTerminate(); //尝试停止线程池

    int c = ctl.get();
	//runState为RUNNING或SHUTDOWN
    if (runStateLessThan(c, STOP)) {
		// 线程不是异常结束
        if (!completedAbruptly) {
			//线程池最小空闲数，允许core thread超时就是0，否则就是corePoolSize
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
			// 如果min == 0但是队列不为空要保证有1个线程来执行队列中的任务
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
			// 1.线程池里的线程已经足够处理任务了，那就不用担心了，死掉的线程不需要替换
            // 2.线程池中线程数大于核心线程数
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
		// 1.线程执行任务异常退出，补一个线程
        // 2.线程池线程不够处理任务，队列中还有任务没执行，增加一个线程
        // 3.线程池中线程数小于核心线程数
        addWorker(null, false);
    }
}
```

**6.tryTerminate()**

processWorkerExit方法中会尝试调用tryTerminate来终止线程池。这个方法在任何可能导致线程池终止的动作后执行，例如Worker数递减。

```java
final void tryTerminate() {
    for (;;) { //自旋
        int c = ctl.get();
		// 以下状态直接返回：
        // 1.线程池还处于RUNNING状态
        // 2.SHUTDOWN状态但是任务队列非空
        // 3.runState >= TIDYING 线程池已经停止了或在停止了
        if (isRunning(c) ||
            runStateAtLeast(c, TIDYING) ||
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;

		// 只能是以下情形会继续下面的逻辑：结束线程池。
        // 1.SHUTDOWN状态，这时不再接受新任务而且任务队列也空了
        // 2.STOP状态，当调用了shutdownNow方法

		//走到这里，线程池已经不在运行，阻塞队列已经没有任务，但是还要回收空闲的Worker
        if (workerCountOf(c) != 0) { // Eligible to terminate
			// 中断闲置Worker，直到回收全部的Worker。这里没有那么暴力，只中断一个，中断之后退出方法，中断了Worker之后，
			//Worker会回收，然后还是会调用tryTerminate方法，如果还有闲置线程，那么继续中断
            interruptIdleWorkers(ONLY_ONE); 
            return;
        }
        // 空闲线程移除完毕，进入TIDYING状态，工作线程为0
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) { //工作线程设为0
                try {
                    terminated(); // 子类重载：一些资源清理工作
                } finally {
                    ctl.set(ctlOf(TERMINATED, 0)); // TERMINATED状态，真的关闭了
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // CAS失败，则重试
    }
}
```

**7.shutdown()**

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess(); //检查关闭权限
        advanceRunState(SHUTDOWN);// 线程池状态设为SHUTDOWN，如果已经至少是这个状态那么则直接返回
		// 注意这里是中断所有空闲的线程：runWorker中等待任务的线程(getTask)被中断 → 进入processWorkerExit → tryTerminate方法中会保证队列中剩余的任务得到执行
        interruptIdleWorkers();
        onShutdown(); //空方法
    } finally {
        mainLock.unlock();
    }
    tryTerminate(); //尝试终止
}
```

**8.shutdownNow()**

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();  //检查关闭权限
        advanceRunState(STOP);  //设置状态为STOP
        interruptWorkers(); // 中断所有线程
        tasks = drainQueue();// 返回队列中还没有被执行的任务。
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks; 
}
```


### 六、其他方法

**1.interruptIdleWorkers()**

中断空闲的Worker，判断条件是**没被中断的线程且worker没有在执行**。

```java
private void interruptIdleWorkers() {
    interruptIdleWorkers(false);
}

private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
			// w.tryLock能获取到锁，说明该线程没有在运行，因为runWorker中执行任务会先lock
        	// 因此保证了中断的肯定是空闲的线程。
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

**2.interruptWorkers()**   中断所有线程

```java
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted(); // 中断Worker的执行
    } finally {
        mainLock.unlock();
    }
}

void interruptIfStarted() {
    Thread t;
	// Worker无论是否持有锁，只要还没被中断，那就中断Worker
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```

### 七、配置线程池的大小

一般需要根据**任务的类型**来配置线程池大小：

- 如果是CPU密集型任务，就需要尽量压榨CPU，参考值可以设为 **NCPU+1**

- 如果是IO密集型任务，参考值可以设置为**2*NCPU**

参考文献：

1. Java并发包源码学习之线程池（一）ThreadPoolExecutor源码分析 [http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-ThreadPoolExecutor.html](http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-ThreadPoolExecutor.html)

2. 深入分析java线程池的实现原理 [http://www.jianshu.com/p/87bff5cc8d8c](http://www.jianshu.com/p/87bff5cc8d8c)

3. jdk1.8 线程池源码学习 [http://www.jianshu.com/p/a60d40b0e4e9](http://www.jianshu.com/p/a60d40b0e4e9)

4. Java线程池ThreadPoolExecutor源码分析 [http://www.jianshu.com/p/a60d40b0e4e9](http://www.jianshu.com/p/a60d40b0e4e9)