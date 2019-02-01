---
layout:     post
title:      LockSupport源码分析
subtitle:   LockSupport
date:       2019-02-01
author:     Jay
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# LockSupport源码分析

`LockSupport`是用来创建锁和其他同步类的基本线程阻塞原语。简而言之，当调用`LockSupport.park`时，表示当前线程将会等待，直至获得许可，当调用`LockSupport.unpark`时，必须把等待获得许可的线程作为参数进行传递，才能让此线程继续运行。

### 一、LockSupport分析

##### 1.属性

```java
// Hotspot implementation via intrinsics API
// Unsafe实例
private static final sun.misc.Unsafe UNSAFE;
// 表示Thread parkBlocker字段内存偏移地址
private static final long parkBlockerOffset;
// 表示Thread threadLocalRandomSeed字段的内存偏移地址
private static final long SEED;
// 表示Thread threadLocalRandomProbe字段的内存偏移地址
private static final long PROBE;
// 表示Thread threadLocalRandomSecondarySeed字段的内存偏移地址
private static final long SECONDARY;

static {
    try {
        // 获取Unsafe实例
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        // Class<Thread>
        Class<?> tk = Thread.class;
        // 获取Thread的parkBlocker字段的内存偏移地址
        parkBlockerOffset = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("parkBlocker"));
        // 获取Thread的threadLocalRandomSeed字段的内存偏移地址
        SEED = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSeed"));
        // 获取Thread的threadLocalRandomProbe字段的内存偏移地址
        PROBE = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomProbe"));
        // 获取Thread的threadLocalRandomSecondarySeed字段的内存偏移地址
        SECONDARY = UNSAFE.objectFieldOffset
            (tk.getDeclaredField("threadLocalRandomSecondarySeed"));
    } catch (Exception ex) { throw new Error(ex); }
}
```

`UNSAFE`字段表示`sun.misc.Unsafe`类实例，可参考文章[Unsafe使用指南](https://xuanjian1992.top/2018/12/09/Unsafe%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97/)的分析，一般程序中不允许直接调用；而`long`型的字段表示Thread实例对象相应字段在内存中的偏移地址，可以通过该偏移地址获取或者设置该字段的值。

##### 2.构造方法

```java
private LockSupport() {} // Cannot be instantiated. 构造器私有化，LockSupport不允许实例化
```

##### 3.核心方法分析

在分析`LockSupport`中的方法之前，先引入`sun.misc.Unsafe`类中的`park`和`unpark`函数，因为`LockSupport`的核心方法都是基于`Unsafe`类中定义的`park`和`unpark`函数实现的，下面给出两个函数的定义。

```java
public native void park(boolean isAbsolute, long time);
public native void unpark(Thread thread);
```

对两个函数的说明如下

　　① `park`函数，阻塞线程，并且该线程在下列情况发生之前都会被阻塞：① 调用`unpark`函数② 该线程被中断。③ 设置的时间到了。并且当`time`为绝对时间时，`isAbsolute`为`true`，否则`isAbsolute`为`false`。当`time`为0时，表示无限等待，直到`unpark`发生。

　　② `unpark`函数，即激活调用`park`后阻塞的线程。

###### (1) park

`park`方法有两个重载版本:

```java
// 阻塞当前线程
public static void park()；
public static void park(Object blocker)；
```

两个方法的区别在于`park()`方法没有`blocker`参数，即没有设置线程的`parkBlocker`字段。`park(Object)`方法如下:

```java
// 阻塞当前线程
public static void park(Object blocker) {
    Thread t = Thread.currentThread(); // 当前线程
    setBlocker(t, blocker); // 设置blocker
    UNSAFE.park(false, 0L); // 阻塞线程，一直阻塞(0L)
    setBlocker(t, null); // 继续运行之后，设置blocker
}
```

调用`park`方法时，首先获取当前线程，然后设置当前线程的`parkBlocker`字段，即调用`setBlocker`方法，之后调用`Unsafe`类的`park`方法，之后再调用`setBlocker`方法。那么问题来了，为什么要在此`park`方法中要调用两次`setBlocker`方法呢？原因其实很简单，调用`park`方法时，当前线程首先设置好`parkBlocker`字段，然后再调用`Unsafe`的`park`方法，此后，当前线程就已经阻塞了，等待该线程的`unpark`方法被调用，所以后面的一个`setBlocker`方法无法运行，`unpark`方法被调用，该线程获得许可后，就可以继续运行了，也就运行第二个`setBlocker`，把该线程的`parkBlocker`字段设置为null，这样就完成了整个`park`方法的逻辑。如果没有第二个`setBlocker`，那么之后没有调用`park(Object blocker)`，而直接调用`getBlocker`方法，得到的还是前一个`park(Object blocker)`设置的`blocker`，显然是不符合逻辑的。总之，必须要保证在`park(Object blocker)`整个方法执行完后，该线程的`parkBlocker`字段又恢复为null。所以，`park(Object)`方法里必须要调用`setBlocker`方法两次。`setBlocker`方法如下。　

```java
// 设置线程t的parkBlocker字段为arg
private static void setBlocker(Thread t, Object arg) {
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}
```

另外一个无参重载版本`park()`方法如下:

```java
// 阻塞当前线程
public static void park() {
    UNSAFE.park(false, 0L); // 阻塞线程，一直阻塞(0L)
}
```

调用了`park`方法后，会禁用当前线程，除非许可可用。在以下三种情况之一发生之前，当前线程都将处于休眠状态，即下列情况发生时，当前线程会获取许可，可以继续运行。

　　① 其他某个线程将当前线程作为目标调用`unpark`。

　　② 其他某个线程中断当前线程。

　　③ 该调用不合逻辑地（即毫无理由地）返回。

###### (2).parkNanos

此方法表示在许可可用前阻塞当前线程，并最多等待指定的等待时间。

```java
// 限时(纳秒)阻塞当前线程
public static void parkNanos(Object blocker, long nanos) {
    if (nanos > 0) {
        Thread t = Thread.currentThread(); // 当前线程
        setBlocker(t, blocker); // set Blocker
        UNSAFE.park(false, nanos); // 限时阻塞  ns
        setBlocker(t, null); // set blocker = null
    }
}
```

###### (3).parkUntil

```java
// 限时阻塞当前线程(绝对时间，毫秒)
public static void parkUntil(Object blocker, long deadline) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(true, deadline); // true表示使用的是绝对时间，单位毫秒
    setBlocker(t, null);
}
```

###### (4).unpark

```java
// 取消阻塞当前线程
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
```

### 二、示例

下面对比使用对象监视器锁和`LockSupport`，来实现线程之间的通信。

##### 1. 对象监视器锁

```java
public class ObjectMonitorLockTest {
    public static void main(String[] args) {

        ThreadA ta = new ThreadA("ta");

        synchronized(ta) { // 通过synchronized(ta)获取“对象ta的同步锁”
            try {
                System.out.println(Thread.currentThread().getName()+" start ta");
                ta.start(); // 主线程运行，ta阻塞

                System.out.println(Thread.currentThread().getName()+" block");
                // 主线程等待，ta运行
                ta.wait();

                System.out.println(Thread.currentThread().getName()+" continue");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }

    static class ThreadA extends Thread{

        public ThreadA(String name) {
            super(name);
        }

        @Override
      public void run() {
            synchronized (this) { // 通过synchronized(this)获取“当前对象的同步锁”
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println(Thread.currentThread().getName()+" wakeup others");
                notify();    // 唤醒“当前对象上的等待线程”
            }
        }
    }
}
// 输出
main start ta
main block
ta wakeup others
main continue
```

##### 2. LockSupport

```java
public class LockSupportTest {
    private static Thread mainThread;

    public static void main(String[] args) {
        ThreadA t = new ThreadA("test");
        mainThread = Thread.currentThread();

        System.out.println(Thread.currentThread().getName()+" start ta");
        t.start();

        System.out.println(Thread.currentThread().getName()+" block");
        LockSupport.park(mainThread);

        System.out.println(Thread.currentThread().getName()+" continue");

    }

    static class ThreadA extends Thread{

        public ThreadA(String name) {
            super(name);
        }

        @Override
      public void run() {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName()+" wakeup others");
            LockSupport.unpark(mainThread);
        }

    }

}
// 输出
main start ta
main block
test wakeup others
main continue
```

### 参考文献

- [Unsafe使用指南](https://xuanjian1992.top/2018/12/09/Unsafe%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97/)
- [JDK1.8源码分析之LockSupport(一)](http://www.cnblogs.com/leesf456/p/5347293.html)
- [JDK LockSupport](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/locks/LockSupport.html)

