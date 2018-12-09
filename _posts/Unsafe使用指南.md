---
layout:     post
title:      sun.misc.Unsafe使用指南
subtitle:   Unsafe简介
date:       2018-12-09
author:     Jay
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# sun.misc.Unsafe使用指南

**转自 [朱小厮的博客](https://mp.weixin.qq.com/s?__biz=MzU0MzQ5MDA0Mw==&mid=2247484510&idx=2&sn=621f1c67603c8ada3dec2ab0467c65c5&chksm=fb0beecacc7c67dcd7466a246722da66792060ed079dcceb83dcf8e9c9ad515919cef1dc0641&mpshare=1&scene=1&srcid=#rd)**

`Java`是一个安全的开发语言，它阻止开发人员犯很多低级的错误，而大部分的错误都是基于内存管理方面的。如果想搞破坏，可以使用`Unsafe`这个类。这个类是属于`sun.* API`中的类，并且它不是`J2SE`中真正的一部分。

### 实例化sun.misc.Unsafe

如果尝试创建`Unsafe`类的实例，基于以下两种原因是不被允许的：

- `Unsafe`类的构造函数是私有的；

- 虽然它有静态的`getUnsafe()`方法，但是如果尝试调用`Unsafe.getUnsafe()`，会得到一个`SecutiryException`。这个类只有被`JDK`信任的类实例化。

  但是这总会是有变通的解决办法的，一个简单的方式就是**使用反射进行实例化**：

  ```java
  public class UnsafeAcquireTest {
  	public static void main(String[] args) throws Exception {
  		// 不能直接获取，抛出异常 java.lang.SecurityException: Unsafe
  		// Unsafe unsafe = Unsafe.getUnsafe();
  
  		// 可以通过反射获取
  		Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
  		theUnsafe.setAccessible(true);
  		// Unsafe unsafe = (Unsafe) theUnsafe.get(Unsafe.class);
  		 Unsafe unsafe = (Unsafe) theUnsafe.get(null);
  		System.out.println(unsafe);
  	}
  }
  ```

### 使用sun.misc.Unsafe

#### 1.实例化对象，避免初始化

通过`allocateInstance()`方法，可以创建一个类的实例，但是却**不需要调用它的构造函数、初始化代码、各种`JVM`安全检查以及其它的一些底层的东西**。即使构造函数是私有，也可以通过这个方法创建它的实例。

```java
public class UnsafeAllocateInstanceTest {
	public static void main(String[] args) throws Exception {
		// 反射创建Unsafe实例
		Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
		theUnsafe.setAccessible(true);
		Unsafe unsafe = (Unsafe) theUnsafe.get(Unsafe.class);

		// 实例化构造器私有的类
		User user = (User) unsafe.allocateInstance(User.class);
		user.setAge(1);
		user.setName("Jay");
		System.out.println(user);
		System.out.println(unsafe.objectFieldOffset(User.class.getDeclaredField("age")));
	}

	private static class User {
		private String name;
		private int age;
		private static int count = 1;

		/** 构造器私有化 */
		private User() {
		}

		public String getName() {
			return name;
		}

		public void setName(String name) {
			this.name = name;
		}

		public int getAge() {
			return age;
		}

		public void setAge(int age) {
			this.age = age;
		}

		@Override
		public String toString() {
			return "User{" + "name='" + name + '\'' + ", age=" + age + '}';
		}
	}
}
/* 输出
User{name='Jay', age=1}
12
*/
```

#### 2.可以分配内存和释放内存

类中提供的3个本地方法`allocateMemory`、`reallocateMemory`、`freeMemory`分别用于分配内存，扩充内存和释放内存，与`C`语言中的3个方法对应。

#### 3.通过内存偏移地址修改变量值

- **public native long objectFieldOffset(Field field);**
返回指定对象实例`field`的内存地址偏移量，在这个类的其他方法中这个值只是被用作一个访问特定`field`的一个方式。**这个值对于给定的`field`是唯一的，并且后续对该方法的调用都应该返回相同的值。**

- **public native int arrayBaseOffset(Class arrayClass);**
获取给定数组中第一个元素的偏移地址。为了存取数组中的元素，这个偏移地址与`arrayIndexScale`方法的非0返回值一起被使用。
**public native int arrayIndexScale(Class arrayClass)**
获取用户给定数组寻址的换算因子。如果不能返回一个合适的换算因子的时候就会返回0。这个返回值能够与`arrayBaseOffset`一起使用去存取这个数组`class`中的元素。

- **public native boolean compareAndSwapInt(Object obj, long offset,int expect, int update);**
在`obj`的`offset`位置比较`integer field`和期望的值，如果相同则更新。这个方法的操作应该是原子的，因此提供了一种不可中断的方式更新`integer field`。当然还有与`Object`、`Long`对应的`compareAndSwapObject`和`compareAndSwapLong`方法。

- **public native void putOrderedInt(Object obj, long offset, int value);**
  设置`obj`对象中`offset`偏移地址对应的整型`field`的值为指定值。这是一个**有序或者有延迟**的`putIntVolatile`方法，并且不保证值的改变被其他线程立即看到。只有在`field`被`volatile`修饰并且期望被意外修改的时候使用才有用。当然还有与`Object`、`Long`对应的`putOrderedObject`和`putOrderedLong`方法。

- **public native void putObjectVolatile(Object obj, long offset, Object value);**
设置`obj`对象中`offset`偏移地址对应的`object`型`field`的值为指定值。支持`volatile store`语义。
与这个方法对应的`get`方法为：
**public native Object getObjectVolatile(Object obj, long offset);**
获取`obj`对象中`offset`偏移地址对应的`object`型`field`的值,支持`volatile load`语义
这两个方法还有与`Int、Boolean、Byte、Short、Char、Long、Float、Double`等类型对应的相关方法.

- **public native void putObject(Object obj, long offset, Object value);**
  设置`obj`对象中`offset`偏移地址对应的`object`型`field`的值为指定值。与`putObject`方法对应的是`getObject`方法。`Int、Boolean、Byte、Short、Char、Long、Float、Double`等类型都有`getXXX`和`putXXX`形式的方法。

下面通过一个组合示例来了解一下如何使用它们，详细如下：

```java
public class UnsafeCASTest {
	public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, InstantiationException {
		// 反射获取Unsafe实例
		Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
		theUnsafe.setAccessible(true);
		Unsafe unsafe = (Unsafe) theUnsafe.get(Unsafe.class);

		// 实例化构造器私有的类 allocateInstance
		Player player = (Player) unsafe.allocateInstance(Player.class);
		player.setName("Jack");
		player.setAge(10);
		for (Field fd : Player.class.getDeclaredFields()) {
			// 对象Field偏移
			System.out.println("[" +fd.getName() + "]对应的内存偏移地址: " + unsafe.objectFieldOffset(fd));
		}

		Field name = Player.class.getDeclaredField("name");
		long nameOffset = unsafe.objectFieldOffset(name);
		Field age = Player.class.getDeclaredField("age");
		long ageOffset = unsafe.objectFieldOffset(age);
		// CAS compareAndSwapInt
		System.out.println(unsafe.compareAndSwapInt(player, ageOffset, 10, 20));
		System.out.println("age修改后的值:" + player.getAge());

		unsafe.putOrderedInt(player, ageOffset, player.getAge() + 2);
		System.out.println("age修改后的值:" + player.getAge());

		unsafe.putObjectVolatile(player, nameOffset, "tom");
		System.out.println("name:" + player.getName());
		System.out.println("name:" + unsafe.getObjectVolatile(player, nameOffset));
	}

	private static class Player {
		private String name;
		private int age;

		private Player() {
		}

		public String getName() {
			return name;
		}

		public void setName(String name) {
			this.name = name;
		}

		public int getAge() {
			return age;
		}

		public void setAge(int age) {
			this.age = age;
		}
	}

}
/*
输出
[name]对应的内存偏移地址: 16
[age]对应的内存偏移地址: 12
true
age修改后的值:20
age修改后的值:22
name:tom
name:tom
*/
```

#### 4.挂起与恢复线程

将一个线程进行挂起是通过`park`方法实现的，调用 `park`后，线程将一直阻塞直**到超时或者中断等条件**出现。`unpark`可以终止一个挂起的线程，使其恢复正常。整个并发框架中对线程的挂起操作被封装在` LockSupport`类中，`LockSupport`类中有各种版本`park`方法，但最终都调用了`Unsafe.park()`方法。

- **public native void park(boolean isAbsolute, long timeout);**
  阻塞一个线程直到`unpark`出现、线程被中断或者`timeout`时间到期。如果一个`unpark`调用已经出现了，这里只计数。`timeout`为0表示永不过期。当`isAbsolute`为`true`时，`timeout`是相对于新纪元之后的毫秒(`ms`)。否则这个值就是超时前的纳秒数(`ns`)。这个方法执行时也可能不合理地返回。

- **public native void unpark(Thread thread);**
  释放被`park`的一个线程。这个方法也可以被使用来终止一个先前调用`park`导致的阻塞这个操作。操作是不安全的，因此**线程必须保证是活的**。这是`java`代码不是`native`代码。参数`thread`指要解除阻塞的线程。

下面来看一下`LockSupport`类中关于`Unsafe.park`和`Unsafe.unpark`的使用：

```java
private static void setBlocker(Thread t, Object arg) {
    // Even though volatile, hotspot doesn't need a write barrier here.
    UNSAFE.putObject(t, parkBlockerOffset, arg);
}
// 恢复阻塞线程
public static void unpark(Thread thread) {
    if (thread != null)
        UNSAFE.unpark(thread);
}
// 一直阻塞当前线程
public static void park(Object blocker) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(false, 0L); // 相对，0表示永不过期
    setBlocker(t, null);
}
// 阻塞当前线程nanos纳秒
public static void parkNanos(Object blocker, long nanos) {
    if (nanos > 0) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        UNSAFE.park(false, nanos); // 相对，nanos纳秒
        setBlocker(t, null);
    }
}

// 阻塞当前线程 直到deadline(绝对，单位ms)
public static void parkUntil(Object blocker, long deadline) {
    Thread t = Thread.currentThread();
    setBlocker(t, blocker);
    UNSAFE.park(true, deadline); // 绝对，到deadline时刻(ms)
    setBlocker(t, null);
}

// 一直阻塞当前线程
public static void park() {
    UNSAFE.park(false, 0L); // 相对，0表示永不过期 
}

// 阻塞当前线程nanos纳秒
public static void parkNanos(long nanos) {
    if (nanos > 0)
        UNSAFE.park(false, nanos); // 相对，nanos纳秒
}

// 阻塞当前线程 直到deadline(绝对，单位ms)
public static void parkUntil(long deadline) {
    UNSAFE.park(true, deadline); // 绝对，到deadline时刻(ms)
}
```

下面是使用`LockSupport`的示例：

```java
public class LockSupportByUnsafeTest {
	public static void main(String[] args) throws InterruptedException {
		ThreadPark threadPark = new ThreadPark("ThreadPark");
		threadPark.start();
		ThreadUnpark threadUnPark = new ThreadUnpark("ThreadUnpark", threadPark);
		threadUnPark.start();

		// main线程等待ThreadUnpark执行成功
		threadUnPark.join();
		System.out.println(Thread.currentThread().getName() + "--运行成功....");
	}

	private static class ThreadPark extends Thread {
		public ThreadPark(String name) {
			super(name);
		}

		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + "--我将被阻塞在这了60s....");
			LockSupport.parkNanos(1000000000L * 60);
			System.out.println(Thread.currentThread().getName() + "--我被恢复正常了....");
		}
	}

	private static class ThreadUnpark extends Thread {
		public Thread thread;

		public ThreadUnpark(String name, Thread thread) {
			super(name);
			this.thread = thread;
		}

		@Override
		public void run() {
			System.out.println(Thread.currentThread().getName() + "--提前恢复阻塞线程ThreadPark");
			// 恢复阻塞线程
			LockSupport.unpark(thread);

			try {
				TimeUnit.SECONDS.sleep(1);
			} catch (InterruptedException e) {
				e.printStackTrace();
			}
		}
	}

}
/*
ThreadPark--我将被阻塞在这了60s....
ThreadUnpark--提前恢复阻塞线程ThreadPark
ThreadPark--我被恢复正常了....
main--运行成功....
*/
```

当然`sun.misc.Unsafe`中还有一些其它的功能，可以继续深挖。`sun.misc.Unsafe`提供了可以随意查看及修改`JVM`中运行时的数据结构，尽管这些功能在`JAVA`开发本身是不适用的，`Unsafe`是一个用于研究学习`HotSpot`虚拟机非常棒的工具，因为它不需要调用`C++`代码，或者需要创建即时分析的工具。后续有空继续研究这个类。

###参考资料

1. http://mishadoff.com/blog/java-magic-part-4-sun-dot-misc-dot-unsafe/
2. https://zhuanlan.zhihu.com/p/37579394
3. https://blog.csdn.net/fenglibing/article/details/17138079
4. https://blog.csdn.net/dfdsggdgg/article/details/51538601
5. https://blog.csdn.net/aesop_wubo/article/details/7537278
6. https://blog.csdn.net/dfdsggdgg/article/details/51543545