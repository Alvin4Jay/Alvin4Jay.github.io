---
layout:     post
title:      FutureTask源码分析之预备知识
subtitle:   预备知识
date:       2019-02-28
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# FutureTask源码分析之预备知识

`FutureTask` 实现了`Future`语义，表示了一种抽象的可生成结果的计算。在包括线程池在内的许多工具类中都会用到，弄懂它的实现将有利于深入理解Java异步操作的实现。`FutureTaask`的继承类图如下:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/FutureTask%E7%BB%A7%E6%89%BF%E7%B1%BB%E5%9B%BE.png?x-oss-process=style/markdown-pic)

可以看到`FutureTask`实现了`RunableFuture`接口，而`RunableFuture`接口继承自`Runnbale`、`Future`接口。在分析`FutureTask`的源码之前，需要先了解这些接口的作用以及其他一些预备知识。下面先来看看`FutureTask`中所使用到的接口：`Runnable`、`Callable`、`Future`、`RunnableFuture`以及所使用到的工具类`Executors`，`Unsafe`。

### 一、FutureTask所使用到的接口

##### 1.Runnable接口

创建线程的一种方式是实现该接口，并实现其`run`方法，这个方法定义了这个线程要做的事情。

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

但是这个`run`方法并没有任何返回值，如果希望执行某种类型的操作**并拿到它的执行结果**，则需要使用`Callable`接口。

##### 2.从Runnable到Callable

要从某种类型的操作中拿到执行结果, 最简单的方式自然是令这个操作自己返回操作结果, 则相较于`run`方法返回`void`，可以令一个操作返回特定类型的对象, 这种思路的实现就是`Callable`接口:

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

对比`Callable`接口与`Runnable`接口，可以发现它们最大的不同点在于:

1. `Callable`有返回值；
2. `Callable`可以抛出异常。

关于有返回值这点，这是`Callable`接口的需求。`call`方法的返回值类型采用的泛型，该类型是在创建`Callable`对象的时候指定的。

除了有返回值外，相较于`Runnable`接口，`Callable`还可以抛出异常，这点看上去好像没啥特别的，但是却有大用处——这意味着如果在任务执行过程中发生了异常，可以将它向上抛出给任务的调用者来妥善处理，甚至可以利用这个特性来**中断一个任务的执行**。而`Runnable`接口的`run`方法不能抛出异常，只能在方法内部`catch`住处理，丧失了一定的灵活性。

使用`Callable`接口解决了返回执行结果的问题, 但是也带来了一个新的问题，即如何获取执行结果。以下是获取结果的一种方式:

```java
public static void main(String[] args) {
    Callable<String> myCallable = () -> "This is the results.";
    try {
        String result = myCallable.call();
        System.out.println("Callable 执行的结果是: " + result);
    } catch (Exception e) {
        System.out.println("There is a exception.");
    }
}
```

这种方法确实可以, 但是它存在几个问题:

1. `call`方法是在当前线程中直接调用的，无法利用多线程。
2. `call`方法可能是一个特别耗时的操作，这将导致程序停在`myCallable.call()`调用处，无法继续运行，直到`call`方法返回。
3. 如果`call`方法始终不返回，没办法取消它的运行。

因此，理想的操作应当是，将`call`方法提交给另外一个线程执行，并在合适的时候判断任务是否完成，然后获取线程的执行结果或者撤销任务，这种思路的实现就是`Future`接口。

##### 3.Future接口

`Future`接口被设计用来代表一个异步操作的执行结果。可以用它来获取一个操作的执行结果、取消一个操作、判断一个操作是否已经完成或者是否被取消。

```java
public interface Future<V> {
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
    
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    
    boolean isDone();
}
```

`Future`接口一共定义了5个方法：

- `get()`
  - 该方法用来获取执行结果，如果任务还在执行中，就阻塞等待;
- `get(long timeout, TimeUnit unit)`
  - 该方法同`get`方法类似，所不同的是，它最多等待指定的时间，如果指定时间内任务没有完成，则会抛出`TimeoutException`异常;
- `cancel(boolean mayInterruptIfRunning)`
  - 该方法用来尝试取消一个任务的执行，它的返回值是`boolean`类型，表示取消操作是否成功。
- `isCancelled()`
  - 该方法用于判断任务是否被取消了。如果一个任务在正常执行完成之前被`cancel`掉了, 则返回`true`。
- `isDone()`
  - 如果一个任务已经结束，则返回`true`。注意，这里的任务结束包含了以下三种情况:
    - 任务正常执行完毕；
    - 任务抛出了异常；
    - 任务已经被取消。

关于`cancel`方法，这里要补充说几点： 
首先有以下三种情况之一的，`cancel`操作一定是失败的：

1. 任务已经执行完成了；
2. 任务已经被取消过了；
3. 任务因为某种原因不能被取消。

其它情况下，`cancel`操作将返回`true`。值得注意的是，**`cancel`操作返回`true`并不代表任务真的就是被取消了**，这取决于调用`cancel`方法时任务所处的状态：

- 如果发起`cancel`时任务还没有开始运行，则随后任务就不会被执行；
- 如果发起`cancel`时任务已经在运行了，则这时就需要看`mayInterruptIfRunning`参数。
  - 如果`mayInterruptIfRunning` 为`true`，则当前执行任务的线程会被中断；
  - 如果`mayInterruptIfRunning` 为`false`, 则可以**允许正在执行的任务继续运行，直到它执行完**。

这个`cancel`方法的规范看起来有点绕，现在不太理解没关系，后面结合实例去看就容易弄明白了，我们将在下一篇分析`FutureTask`源码的时候详细说明`FutureTask`对这一方法的实现。

##### 4.RunnableFuture接口

`RunnableFuture`接口同时实现了`Runnable`接口和`Future`接口:

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run(); 
}
```

`FutureTask`实现了该接口，相当于同时实现了`Runnable`接口和`Future`接口。

### 二、FutureTask所使用到的工具类

##### 1.Executors

`Executors`是一个用于创建线程池的工厂类，这个类同时也提供了一些有用的静态方法。前面提到了`Callable`接口，它是JDK1.5才引入的，而`Runnable`接口在JDK1.0就有了，有时候需要将一个已经存在的`Runnable`对象转换成`Callable`对象，`Executors`工具类为我们提供了这一实现:

```java
public class Executors {
    public static <T> Callable<T> callable(Runnable task, T result) {
        if (task == null)
            throw new NullPointerException();
        return new RunnableAdapter<T>(task, result); // 适配器模式
    }
    
    static final class RunnableAdapter<T> implements Callable<T> {
        final Runnable task;
        final T result;
        RunnableAdapter(Runnable task, T result) {
            this.task = task;
            this.result = result;
        }
        public T call() {
            task.run();
            return result;
        }
    }
}
```

可以看出，这个方法采用了设计模式中的**适配器模式**，将一个`Runnable`类型对象适配成`Callable`类型。

因为`Runnable`接口没有返回值，所以为了与`Callable`兼容，额外传入了一个`result`参数，使得返回的`Callable`对象的`call`方法直接执行`Runnable`的`run`方法，然后返回传入的`result`参数。

##### 2.Unsafe

`Unsafe`类对于并发编程来说是个很重要的类，这个类最大的特点在于，它提供了硬件级别的CAS原子操作。CAS可以说是实现了最轻量级的锁，当多个线程尝试使用CAS同时更新同一个变量时，只有其中的一个线程能成功地更新变量的值，而其他的线程将失败。然而，失败的线程并不会被挂起。

CAS操作包含了三个操作数： **需要读写的内存位置，进行比较的原值，拟写入的新值。**

在`Unsafe`类中，实现CAS操作的方法是： `compareAndSwapXXX`，例如:

```java
public native boolean compareAndSwapObject(Object obj, long offset, Object expect, Object update);
```

- `obj`是要操作的目标对象；
- `offset`表示了目标对象中，对应的属性的内存偏移量；
- `expect`是进行比较的原值；
- `update`是拟写入的新值。

所以该方法实现了对目标对象obj中的某个成员变量（`field`）进行CAS操作的功能。目标对象obj中的某个成员变量`field`的内存偏移量的获取，使用以下方法:

```java
public native long objectFieldOffset(Field field);
```

该方法的参数是要进行CAS操作的`Field`对象，要怎么获得这个`Field`对象呢？最直接的办法就是通过反射：

```java
Class<?> k = FutureTask.class;
Field stateField = k.getDeclaredField("state");
```

这样以后，就能对`FutureTask`的`state`属性进行CAS操作了。

除了`compareAndSwapObject`，`Unsafe`类还提供了更为具体的对`int`和`long`类型的CAS操作：

```
public native boolean compareAndSwapInt(Object obj, long offset, int expect, int update);
public native boolean compareAndSwapLong(Object obj, long offset, long expect, long update);
```

从方法签名可以看出，这里只是把目标`Field`的类型限定成`int`和`long`类型，而不是通用的`Object`。

最后，`FutureTask`还用到了一个方法:

```java
public native void putOrderedInt(Object obj, long offset, int value);
```

可以看出，该方法只有三个参数，所以它没有比较再交换的概念，某种程度上就是一个赋值操作，即设置obj对象中`offset`偏移地址对应的`int`类型的`field`的值为指定值。这其实是`Unsafe`的另一个方法`putIntVolatile`的**有序或者有延迟**的版本，并且**不保证值的改变被其他线程立即看到**，只有在`field`被`volatile`修饰并且期望被意外修改的时候使用才有用。

```java
public native void putIntVolatile(Object obj, long offset, int value);
```

`putIntVolatile`的作用是: 该方法设置obj对象中`offset`偏移地址对应的整型`field`的值为指定值，支持`volatile store`语义。由此可以看出，当操作的`int`类型`field`本身已经被`volatile`修饰时，`putOrderedInt`和`putIntVolatile`是等价的。



到此为止，在分析`FutureTask`之前需要的预备知识已经介绍完毕。