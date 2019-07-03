---
layout:     post
title:      Thread类之Java并发编程的基础
subtitle:   Thread
date:       2019-07-03
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
---

# Thread类之Java并发编程的基础

### 一、线程的简介和一些属性

##### 1.ThreadMXBean获取线程相关信息

```java
public class MultiThread {
	public static void main(String[] args) {
		// 获取Java线程管理MXBean，一个Java虚拟机只有该接口的一个单独实例(一个Java进程)
		ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();

		// 获取线程信息
		ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);

		// 打印线程id和name
		for (ThreadInfo threadInfo : threadInfos) {
			System.out.println("[" + threadInfo.getThreadId() + "] " + threadInfo.getThreadName());
		}
	}
}
// 输出
/**
[5] Monitor Ctrl-Break
[4] Signal Dispatcher
[3] Finalizer
[2] Reference Handler
[1] main
*/
```

##### 2.线程优先级(priority)

在`Java`线程中，线程优先级的范围为1至10，默认优先级为5。**设置线程优先级时，针对频繁阻塞(休眠或者`I/O`操作)的线程需要设置较高的优先级，而偏计算(需要较多`CPU`时间或偏运算)的线程则设置较低的优先级，确保处理器不会被占用。**在不同的`JVM`和操作系统上，线程设置存在差异，<u>有些操作系统会忽略对线程优先级的设置</u>。

```java
public class Priority {
	private static volatile boolean notStart = true;
	private static volatile boolean notEnd = true;

	public static void main(String[] args) throws InterruptedException {
		List<Task> tasks = new ArrayList<>();

		for (int i = 0; i < 10; i++) {
			int priority = i < 5 ? Thread.MIN_PRIORITY : Thread.MAX_PRIORITY;
			Task task = new Task(priority);
			tasks.add(task);
			Thread thread = new Thread(task, "Thread: " + i);
			thread.setPriority(priority);
			thread.start();
		}

		// 10个线程开始计数
		notStart = false;
		// 主线程main Sleep 10s
		TimeUnit.SECONDS.sleep(10);
		// 10个线程计数结束
		notEnd = false;

		for (Task task : tasks) {
			System.out.println("Task Priority: " + task.priority + ", count: " + task.count);
		}

	}

	static class Task implements Runnable {
		private int priority;
		private int count;

		public Task(int priority) {
			this.priority = priority;
		}

		@Override
		public void run() {
			while (notStart) {
				// 当前线程让出CPU执行权，由RUNNING转为READY就绪状态，
				// 下次这个线程仍有可能执行
				Thread.yield();
			}
			while (notEnd) {
				Thread.yield();
				count++;
			}
		}
	}

}

// 输出，最终的count计数基本相同
/**
Task Priority: 1, count: 690565
Task Priority: 1, count: 690718
Task Priority: 1, count: 692321
Task Priority: 1, count: 691728
Task Priority: 1, count: 691294
Task Priority: 10, count: 691301
Task Priority: 10, count: 691471
Task Priority: 10, count: 691912
Task Priority: 10, count: 691599
Task Priority: 10, count: 690636
*/
```

##### 3.线程的状态

`Java`线程在其生命周期中有如下的6种状态:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/practicesummary/AJCP-C4-1.png)

线程在自身的生命周期中，随着代码的执行，在不同的状态之间切换。如下图所示:

![线程状态转换](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/practicesummary/AJCP-C4-2.png)

注：①操作系统中的运行和就绪两种状态在`Java`中称为`RUNNABLE`状态；②`BLOCKED`状态是线程阻塞在进入`synchronized`方法或代码块时的状态；③阻塞在`concurrent`包中`Lock`接口实现的线程状态是`WAITING`状态，因为`concurrent`包中的`Lock`接口对于阻塞的实现使用了`LockSupport`类中的相关方法。

##### 4.Daemon线程

守护线程，主要用于程序中后台调度以及支持性工作。当一个`Java`虚拟机中不存在非`Daemon`线程时，`Java`虚拟机将退出。`Thread.setDaemon(boolean)`用于设置线程是否为`Daemon`线程，需在线程启动前设置。

`Java`虚拟机退出时，`Daemon`线程中的`finally`块并不一定会执行，如下所示：

```java
// 无任何输出
public class Daemon {
	public static void main(String[] args){
	    Thread thread = new Thread(new DaemonRunner(), "DaemonRunner");
	    // 设置为daemon线程
	    thread.setDaemon(true);
	    thread.start();
	}

	static class DaemonRunner implements Runnable {
		@Override
		public void run() {
			try {
				SleepUtils.second(10);
			} finally {
				System.out.println("DaemonThread finally run.");
			}
		}
	}
}
```

因此，**不能依靠`Daemon`线程中`finally`块中的内容来确保执行关闭或者清理资源的逻辑。**

### 二、启动和终止线程

##### 1.线程初始化与启动

线程在运行之前首先需要初始化，线程在初始化时需要提供一些属性如线程所属的线程组`ThreadGroup`、线程名`name`、线程优先级`priority`、是否为`Daemon`线程等，`Thread`中简化的初始化过程如下所示：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/practicesummary/AJCP-C4-3.png)

新线程对象由父线程进行空间分配，且新线程继承了父线程是否为`Daemon`线程、优先级、加载资源的`contextClassLoader`以及可继承的`ThreadLocal`等属性，也分配了一个惟一的TID表示这个子线程。

线程初始化之后，通过调用`start()`方法启动线程。父线程同步告知`Java`虚拟机，只要线程调度器空闲，应立即执行线程。

##### 2.中断interrupt

中断是线程的一个标识位属性，表示一个运行中的线程是否被其他线程进行了中断操作。其他线程调用该线程的`interrupt()`方法对其进行中断操作。

线程可通过`isInterrupted()`实例方法和`interrupted()`静态方法判断自己是否被中断。`interrupted()`静态方法在判断是否被中断的同时，会清除中断标志位。

- 对于许多声明抛出`InterruptedException`异常的方法，如果某线程的这个方法在运行中被中断，则在抛出`InterruptedException`异常之前，中断标志位会被清除，然后再抛出该异常。
- 对于未声明抛出`InterruptedException`异常的方法，如果某线程的这个方法在运行中被中断，则中断标志位置位，不会被清除，程序可通过`Thread.currentThread().isInterrupted()`方法进行判断。

以下是中断的测试用例:

```java
// 中断测试，查看中断标志位
public class InterruptedTest {
	public static void main(String[] args) {
		Thread sleepThread = new Thread(new SleepRunner(), "SleepRunner");
		sleepThread.setDaemon(true);

		Thread busyThread = new Thread(new BusyRunner(), "BusyRunner");
		busyThread.setDaemon(true);

		sleepThread.start();
		busyThread.start();
		// 等待5s
		SleepUtils.second(5);
		// 触发中断
		sleepThread.interrupt();
		busyThread.interrupt();

		// sleep线程 抛出异常，清除中断标志位，线程正常运行  false
		System.out.println("SleepThread interrupted is " + sleepThread.isInterrupted());
		// 中断标志位置位，不会被清除，线程正常运行  true
		System.out.println("BusyThread interrupted is " + busyThread.isInterrupted());

		SleepUtils.second(25);
	}

	// 一直睡眠
	static class SleepRunner implements Runnable {
		private static int num = 0;

		@Override
		public void run() {
			while (true) {
				SleepUtils.second(10);
				System.out.println("Here " + num++);
			}
		}
	}

	// 一直执行
	static class BusyRunner implements Runnable {
		@Override
		public void run() {
			while (true) {

			}
		}
	}

}
```

##### 3.安全地终止线程

一般有以下两种方式可用于安全地终止线程:

- 因为中断状态是线程的一个标志位，且中断操作是一种线程间交互的方式，因此可以使用中断操作来终止线程的执行。
- 利用一个`volatile`的`boolean`变量，控制线程的运行。

```java
// 正确地关闭线程
public class CorrectShutdownTest {
	public static void main(String[] args) {
		Runner one = new Runner();
		Thread countThread = new Thread(one, "CountThreadOne");
		countThread.start();

		SleepUtils.second(1);
		// 根据中断状态终止线程
		countThread.interrupt();

		Runner two = new Runner();
		Thread countThread2 = new Thread(two, "CountThreadTwo");
		countThread2.start();

		SleepUtils.second(1);
		// 根据volatile boolean变量终止线程
		two.cancel();
	}

	/**
	 * 计数线程
	 */
	private static class Runner implements Runnable {
		private int count = 0;
		/** 开关变量 */
		private volatile boolean on = true;

		@Override
		public void run() {
			// 根据中断标志位和volatile boolean变量进行判断，终止线程
			while (on && !Thread.currentThread().isInterrupted()) {
				count++;
			}
			System.out.println(Thread.currentThread().getName() + " Count: " + count);
		}

		public void cancel() {
			on = false;
		}
	}

}
```

