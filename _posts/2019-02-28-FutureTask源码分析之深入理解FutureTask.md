---
layout:     post
title:      FutureTask源码分析之深入理解FutureTask
subtitle:   深入理解
date:       2019-02-28
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# FutureTask源码分析之深入理解FutureTask

接上一篇预备知识之后，下面开始分析`FutureTask`的工作原理(基于JDK 1.8)。

### 一、Future和Task

`FutureTask`包含了`Future`和`Task`两部分。上一篇说过，`FutureTask`实现了`RunnableFuture`接口，即`Runnable`接口和`Future`接口。其中`Runnable`接口对应了`FutureTask`名字中的`Task`，代表`FutureTask`本质上也是表征了一个任务。而`Future`接口就对应了`FutureTask`名字中的`Future`，表示了对于这个任务可以执行某些操作，例如判断任务是否执行完毕、获取任务的执行结果、取消任务的执行等。

所以说，`FutureTask`本质上就是一个“Task”，可以把它当做简单的`Runnable`对象来使用。但是它又同时实现了`Future`接口，因此可以对它所代表的“Task”进行额外的控制操作。

### 二、FutureTask的概貌

下面以**状态，队列，CAS**这三部分为切入点，快速了解`FutureTask`的概貌。

##### 1.状态

在`FutureTask`中，状态是由`state`属性来表示的，它是`volatile`类型的，确保了不同线程对它修改的可见性：

```java
// 状态
private volatile int state;
private static final int NEW          = 0; // 初始状态
private static final int COMPLETING   = 1; // 执行完成，设置结果中
private static final int NORMAL       = 2; // 正常结束
private static final int EXCEPTIONAL  = 3; // 异常结束
private static final int CANCELLED    = 4; // 取消
private static final int INTERRUPTING = 5; // 中断中
private static final int INTERRUPTED  = 6; // 被中断了
```

`state`属性是贯穿整个`FutureTask`最核心的属性，该属性的值代表了任务在运行过程中的状态，随着任务的执行，状态将不断地进行转变，从上面的定义中可以看出，总共有7种状态：包括了1个初始态，2个中间态和4个终止态。

虽说状态有这么多，但是状态的转换路径却只有四种：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/FutureTask%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A2.png)

- 任务的初始状态都是`NEW`，这一点是构造函数保证的，后面分析构造函数的时候再讲；
- 任务的终止状态有4种：
  - `NORMAL`：任务正常执行完毕；
  - `EXCEPTIONAL`：任务执行过程中发生异常；
  - `CANCELLED`：任务被取消；
  - `INTERRUPTED`：任务被中断；
- 任务的中间状态有2种：
  - `COMPLETING` 正在设置任务结果；
  - `INTERRUPTING` 正在中断运行任务的线程。

任务的中间状态是一个瞬态，它非常的短暂。而且**任务的中间态并不代表任务正在执行，而是任务已经执行完了，正在设置最终的返回结果**，所以可以这么说：

> 只要state不处于 `NEW` 状态，就说明任务已经执行完毕。

注意，这里的执行完毕是指传入的`Callable`对象的`call`方法执行完毕，或者抛出了异常。所以这里的`COMPLETING`的名字显得有点迷惑性，**它并不意味着任务正在执行中，而意味着`call`方法已经执行完毕，正在设置任务执行的结果。**

而将一个任务的状态设置成终止态只有三种方法(在下面的源码解析中再分析这三个方法)：

- `set`
- `setException`
- `cancel`

##### 2.队列

接着来看队列，在`FutureTask`中，队列的实现是一个单向链表，它表示**所有等待任务执行完毕的线程的集合**。`FutureTask`实现了`Future`接口，可以获取“Task”的执行结果，那么如果获取结果时，任务还没有执行完毕，那么获取结果的线程就会在一个等待队列中挂起，直到任务执行完毕被唤醒。这一点有点类似于之前学习的[AQS](https://xuanjian1992.top/2019/01/06/AQS%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%901-%E7%8B%AC%E5%8D%A0%E9%94%81%E7%9A%84%E8%8E%B7%E5%8F%96%E4%B8%8E%E9%87%8A%E6%94%BE/)中的`sync queue`。

前面说过，在并发编程中使用队列通常是**将当前线程包装成某种类型的数据结构扔到等待队列中**，先来看看队列中的每一个节点的数据结构：

```java
static final class WaitNode {
    // 阻塞的线程
    volatile Thread thread;
    // 下一个节点
    volatile WaitNode next;
    WaitNode() { thread = Thread.currentThread(); }
}
```

可见，相比于AQS的`sync queue`所使用的双向链表中的`Node`，这个`WaitNode`要简单多，它只包含了一个记录线程的`thread`属性和指向下一个节点的`next`属性。

值得一提的是，`FutureTask`中的这个单向链表是当做**栈**来使用的，确切来说是当做**Treiber栈**来使用的(可以简单的把它当做是一个线程安全的栈，它使用CAS来完成入栈出栈操作，参考[Treiber Stack简单分析](https://segmentfault.com/a/1190000012463330))。使用线程安全的栈的原因是同一时刻可能有多个线程都在获取任务的执行结果，如果任务还在执行过程中，则这些线程就要被包装成`WaitNode`扔到`Treiber`栈的栈顶，即完成入栈操作，这样就有可能出现多个线程同时入栈的情况，因此需要使用CAS操作保证入栈的线程安全，对于出栈的情况也是同理。

由于`FutureTask`中的队列本质上是一个`Treiber`栈，那么使用这个队列就只需要一个指向栈顶节点的指针就行了，在`FutureTask`中，就是`waiters`属性：

```java
/** Treiber stack of waiting threads */
private volatile WaitNode waiters; // Treiber stack栈顶
```

事实上，它就是整个单向链表的头节点。综上，`FutureTask`中所使用的队列的结构如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/FutureTask%E7%AD%89%E5%BE%85%E9%98%9F%E5%88%97.png?x-oss-process=style/markdown-pic)

##### 3.CAS操作

CAS操作大多数是用来改变状态的，在`FutureTask`中也不例外。一般在静态代码块中初始化需要CAS操作的属性的内存偏移量：

```java
// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE;
private static final long stateOffset;
private static final long runnerOffset;
private static final long waitersOffset;
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = FutureTask.class;
        stateOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("state"));
        runnerOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("runner"));
        waitersOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("waiters"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
```

从这个静态代码块中也可以看出，CAS操作主要针对3个属性，包括`state`、`runner`和`waiters`，说明这3个属性基本是会被多个线程同时访问的。其中`state`属性代表了任务的状态，`waiters`属性代表了指向栈顶节点的指针，这两个上面已经分析过。`runner`属性代表了执行`FutureTask`中的“Task”的线程，该属性是为了中断或者取消任务做准备的，只有知道了执行任务的线程，才能去中断它。

定义完属性的内存偏移量之后，接下来就是CAS操作本身了。在`FutureTask`，CAS操作最终调用的还是`Unsafe`类的`compareAndSwapXXX`方法，关于这一点，上一篇预备知识中已经讲过，这里不再介绍。

### 三、核心属性

前面以**状态，队列，CAS**为切入点分析了`FutureTask`的状态、队列和CAS操作，对这个类有了初步的认识。接下来就开始源码分析。首先先来看看`FutureTask`的几个核心属性：

```java
/**
     * The run state of this task, initially NEW.  The run state
     * transitions to a terminal state only in methods set,
     * setException, and cancel.  During completion, state may take on
     * transient values of COMPLETING (while outcome is being set) or
     * INTERRUPTING (only while interrupting the runner to satisfy a
     * cancel(true)). Transitions from these intermediate to final
     * states use cheaper ordered/lazy writes because values are unique
     * and cannot be further modified.
     *
     * Possible state transitions:
     * NEW -> COMPLETING -> NORMAL
     * NEW -> COMPLETING -> EXCEPTIONAL
     * NEW -> CANCELLED
     * NEW -> INTERRUPTING -> INTERRUPTED
     */
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;

    /** The underlying callable; nulled out after running */
    private Callable<V> callable;
    /** The result to return or exception to throw from get() */
    private Object outcome; // non-volatile, protected by state reads/writes
    /** The thread running the callable; CASed during run() */
    private volatile Thread runner;
    /** Treiber stack of waiting threads */
    private volatile WaitNode waiters;
```

可以看出，`FutureTask`的核心属性只有5个：

- `state`
- `callable`
- `outcome`
- `runner`
- `waiters`

关于 `state`、 `waiters`、 `runner`三个属性上面已经解释过。剩下的`callable`属性代表了要执行的任务本身，即**FutureTask中的“Task”部分**，为`Callable`类型，这里之所以用`Callable`而不用`Runnable`，是因为`FutureTask`实现了`Future`接口，需要获取任务的执行结果。`outcome`属性代表了任务的执行结果或者抛出的异常，为`Object`类型，也就是说`outcome`可以是**任意类型**的对象，所以当将正常的执行结果返回给调用者时，需要进行强制类型转换，返回由`Callable`定义的`V`类型。这5个属性综合起来就完成了整个`FutureTask`的工作，使用关系如下：

- 任务：`callable`
- 任务的执行者：`runner`
- 任务的结果：`outcome`
- 获取任务的结果：`state` + `outcome` + `waiters`
- 中断或者取消任务：`state` + `runner` + `waiters`

### 四、构造函数

介绍完核心属性之后，来看看`FutureTask`的构造函数:

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    this.state = NEW;       // 由volatile int state 保证callable变量的可见性
}

public FutureTask(Runnable runnable, V result) {
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // 由volatile int state 保证callable变量的可见性
}
```

`FutureTask`共有2个构造函数，这2个构造函数一个是直接传入`Callable`对象，一个是传入一个`Runnable`对象和一个指定的`result`，然后通过`Executors`工具类将它适配成`Callable`对象，所以这两个构造函数的本质是一样的:

1. 用传入的参数初始化`callable`成员变量；
2. 将`FutureTask`的状态设为`NEW`。

### 五、接口实现

前面提过，`FutureTask`实现了`RunnableFuture`接口：

```java
public class FutureTask<V> implements RunnableFuture<V> {
    ...
}
```

因此，它必须实现`Runnable`和`Future`接口的所有方法。

##### 1.Runnable接口实现

要实现`Runnable`接口, 就得覆写`run`方法，下面看看`FutureTask.run()`方法的逻辑:

###### run()

```java
public void run() {
    if (state != NEW || // 如果state不是NEW或者CAS设置runner属性失败，则退出
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call(); // 执行任务
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex); // 设置异常
            }
            if (ran)
                set(result); // 设置结果
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state; 
        if (s >= INTERRUPTING) // 检查中断
            handlePossibleCancellationInterrupt(s);
    }
}
```

首先在`run`方法的一开始，就检查当前状态是不是`New`, 并且使用CAS操作将`runner`属性设置为当前线程，即记录执行任务的线程。`compareAndSwapObject`的用法在上一篇预备知识中已经介绍过了，这里不再赘述。可见，`runner`属性是在运行时被初始化的。

接下来调用`Callable`对象的`call`方法来执行任务，如果任务执行成功，就使用`set(result)`设置结果，否则，用`setException(ex)`设置抛出的异常。

先来看看`set(result)`方法：

```java
// 设置结果
protected void set(V v) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {// 先将state置为COMPLETING
        outcome = v;
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL); // final state // 再将state置为NORMAL
        finishCompletion();
    }
}
```

这个方法一开始通过CAS操作将`state`属性由原来的`NEW`状态修改为`COMPLETING`状态。在一开始介绍`state`状态的时候说过，`COMPLETING`是一个非常短暂的中间态，表示正在设置执行的结果。

状态设置成功后，就把任务执行结果赋值给`outcome`, 然后直接把`state`状态设置成`NORMAL`。注意，这里是直接设置，没有先比较再设置的操作，由于`state`属性被设置成`volatile`, 结合上一篇预备知识的介绍，这里`putOrderedInt`应当和`putIntVolatile`是等价的，保证了`state`状态对其他线程的可见性。在这之后，调用了` finishCompletion()`将所有等待线程唤醒。

接下来再来看看发生了异常的版本`setException(ex)`：

```java
// 设置异常
protected void setException(Throwable t) {
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
        outcome = t;
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); // final state 异常状态
        finishCompletion();
    }
}
```

可见，除了将`outcome`属性赋值为异常对象，以及将`state`的终止状态修改为`EXCEPTIONAL`，其余都和set方法类似。在方法的最后，都调用了` finishCompletion()`将所有等待线程唤醒。

```java
// 唤醒等待结果的线程
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t); // 唤醒
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
```

这个方法事实上完成了一个“善后”工作。先来看看if条件语句中的CAS操作：

```java
UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)
```

该方法是将`waiters`属性的值由原值设置为`null`，由于`waiters`属性指向了Treiber栈的栈顶节点，可以说是代表了整个Treiber栈，将该值设为`null`的目的就是清空整个栈。如果设置不成功，则if语句块不会被执行，又进行下一轮for循环，而下一轮for循环的判断条件又是`waiters!=null` ，由此，虽然最外层的for循环乍一看好像是什么遍历节点的操作，其实只是为了确保`waiters`属性被成功设置成`null`，本质上相当于一个自旋操作。

将`waiters`属性设置成`null`以后，接下了` for (;;)`死循环才是真正的遍历节点，可以看出，循环内部就是一个普通的遍历链表的操作，前面讲属性的时候说过，Treiber栈里面存放的WaitNode代表了当前等待任务执行结束的线程，这个循环的作用也正是遍历链表中所有等待的线程，并唤醒他们。

将Treiber栈中所有挂起的线程都唤醒后，下面就是执行`done`方法：

```java
/**
 * Protected method invoked when this task transitions to state
 * {@code isDone} (whether normally or via cancellation). The
 * default implementation does nothing.  Subclasses may override
 * this method to invoke completion callbacks or perform
 * bookkeeping. Note that you can query status inside the
 * implementation of this method to determine whether this task
 * has been cancelled.
 */
protected void done() { } // 子类实现
```

这个方法是一个空方法，从注释上看，它是提供给子类覆写的，以实现一些任务执行结束前的额外操作。

`done`方法之后就是`callable`属性的清理了（`callable = null`）。

`finishCompletion`方法执行完毕后，是`run`方法`finally`代码块的执行:

```java
finally {
    // runner must be non-null until state is settled to
    // prevent concurrent calls to run()
    runner = null;
    // state must be re-read after nulling runner to prevent
    // leaked interrupts
    int s = state;
    if (s >= INTERRUPTING)
        handlePossibleCancellationInterrupt(s);
}
```

在`finally`块中，将`runner`属性置为null，并且检查有没有遗漏的中断，如果发现`s >= INTERRUPTING`, 说明执行任务的线程有可能被中断了，因为`s >= INTERRUPTING` 只有两种可能，`state`状态为`INTERRUPTING`和`INTERRUPTED`。(在多线程的环境中，在当前线程执行`run`方法的同时，有可能其他线程取消了任务的执行。)

关于任务取消的操作，后面讲`Future`接口的实现的时候再讲，回到现在的问题，来看看`handlePossibleCancellationInterrupt`方法的逻辑：

```java
/**
 * Ensures that any interrupt from a possible cancel(true) is only
 * delivered to a task while in run or runAndReset.
 */
private void handlePossibleCancellationInterrupt(int s) {
    // It is possible for our interrupter to stall before getting a
    // chance to interrupt us.  Let's spin-wait patiently.
    if (s == INTERRUPTING)
        while (state == INTERRUPTING)
            Thread.yield(); // wait out pending interrupt
}
```

可见该方法是一个自旋操作，如果当前的`state`状态是`INTERRUPTING`，线程在原地自旋，直到`state`状态转换成终止态。

至此，`run`方法分析结束，总结如下：

`run`方法重点做了以下几件事：

1. 将`runner`属性设置成当前正在执行`run`方法的线程；
2. 调用`callable`成员变量的`call`方法来执行任务；
3. 设置执行结果`outcome`，如果执行成功，则`outcome`保存的就是执行结果；如果执行过程中发生了异常, 则`outcome`中保存的就是异常，设置结果之前，先将`state`状态设为中间态；
4. 对`outcome`的赋值完成后，设置`state`状态为终止态(`NORMAL`或者`EXCEPTIONAL`)；
5. 唤醒`Treiber`栈中所有等待的线程；
6. 善后清理(`waiters, callable，runner`设为`null`)
7. 检查是否有遗漏的中断，如果有，等待中断状态完成。

这里再插一句，前面说“**state只要不是NEW状态，就说明任务已经执行完成了**”就体现在这里，因为`run`方法中，是在`c.call()`执行完毕或者抛出了异常之后才开始设置中间态和终止态的。

##### 2.Future接口的实现

`Future`接口一共定义了5个方法，下面一个个来看：

###### cancel(boolean mayInterruptIfRunning)

上面在分析`run`方法的最后，提到了任务可能被别的线程取消，下面看下任务取消的逻辑：

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW && UNSAFE.compareAndSwapInt(this, stateOffset, NEW, mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt(); // 中断执行任务的线程
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```

上一篇在介绍`Future`接口的时候，对`cancel`方法的说明如下：

> 关于`cancel`方法，这里要补充说几点： 
> 首先有以下三种情况之一的，`cancel`操作一定是失败的：
>
> 1. 任务已经执行完成了；
> 2. 任务已经被取消过了；
> 3. 任务因为某种原因不能被取消。
>
> 其它情况下，`cancel`操作将返回`true`。值得注意的是，**`cancel`操作返回`true`并不代表任务真的就是被取消了**，这取决于调用`cancel`方法时任务所处的状态：
>
> - 如果发起`cancel`时任务还没有开始运行，则随后任务就不会被执行；
> - 如果发起`cancel`时任务已经在运行了，则这时就需要看`mayInterruptIfRunning`参数。
>   - 如果`mayInterruptIfRunning` 为`true`，则当前执行任务的线程会被中断；
>   - 如果`mayInterruptIfRunning` 为`false`, 则可以**允许正在执行的任务继续运行，直到它执行完**。

下面来看看`FutureTask`是如何实现`cancel`方法的这几个规范的:

首先，对于“任务已经执行完成了或者任务已经被取消过了，则`cancel`操作一定是失败的(返回`false`)”这两条，是通过简单的判断`state`值是否为`NEW`实现的，因为前面说过了，只要`state`不为`NEW`，说明任务已经执行完毕了。从代码中可以看出，只要`state`不为`NEW`，则直接返回`false`。

如果`state`还是`NEW`状态，再往下看：

```java
UNSAFE.compareAndSwapInt(this, stateOffset, NEW, mayInterruptIfRunning ? INTERRUPTING : CANCELLED)
```

这一段是根据`mayInterruptIfRunning`的值将`state`的状态由`NEW`设置成`INTERRUPTING`或者`CANCELLED`，当这一操作也成功之后，就可以执行后面的try语句了，但无论怎么，该方法最后都返回了`true`。

接着看try块的逻辑:

```java
try {    // in case call to interrupt throws exception
    if (mayInterruptIfRunning) {
        try {
            Thread t = runner;
            if (t != null)
                t.interrupt(); // 中断执行线程
        } finally { // final state
            UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
        }
    }
} finally {
    finishCompletion();
}
```

`runner`属性中存放的是当前正在执行任务的线程，因此，这个try块的目的就是中断当前正在执行任务的线程，最后将`state`的状态设为`INTERRUPTED`，当然，中断操作完成后，还需要通过`finishCompletion()`来唤醒所有在Treiber栈中等待的线程。

现在总结一下，`cancel`方法实际上完成以下两种状态转换之一:

1. `NEW -> CANCELLED `(对应于`mayInterruptIfRunning=false`)
2. `NEW -> INTERRUPTING -> INTERRUPTED` (对应于`mayInterruptIfRunning=true`)

对于第一条路径，虽说`cancel`方法最终返回了`true`，但它只是简单的把`state`状态设为`CANCELLED`，并不会中断线程的执行。**但是这样带来的后果是，任务即使执行完毕了，也无法设置任务的执行结果**，因为根据前面分析`run`方法的结论，设置任务结果有一个中间态，而这个中间态的设置，是以当前`state`状态为`NEW`为前提的。

对于第二条路径，则会中断执行任务的线程，再倒回上面的`run`方法看看：

```java
public void run() {
    if (state != NEW || !UNSAFE.compareAndSwapObject(this, runnerOffset, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                setException(ex);
            }
            if (ran)
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

虽然第二条路径中断了当前正在执行的线程，但是响不响应这个中断是由执行任务的线程自己决定的，更具体的说，这取决于`c.call()`方法内部是否对中断进行了响应，是否将中断异常抛出。

那`call`方法中是怎么处理中断的呢？从上面的代码中可以看出，`catch`语句处理了所有的`Throwable`的异常，这自然也包括了中断异常。

然而，即使这里进入了`catch (Throwable ex){}`代码块，`setException(ex)`的操作一定是失败的，因为在取消任务执行的线程中，已经先把`state`状态设为`INTERRUPTING`了，而`setException(ex)`的操作要求设置前线程的状态为`NEW`。所以这里响应**`cancel`方法所造成的中断**最大的意义不是为了对中断进行处理，而是简单的停止任务线程的执行，节省CPU资源。

既然这个`setException(ex)`的操作一定是失败的，那放在这里有什么用呢？事实上，这个`setException(ex)`是用来处理任务自己在正常执行过程中产生的异常的，在没有主动去`cancel`任务时，任务的`state`状态在执行过程中就会始终是`NEW`，如果任务此时自己发生了异常，则这个异常就会被`setException(ex)`方法成功的记录到`outcome`中。

无论如何，`run`方法最终都会进入`finally`块，而这时候它会发现`s >= INTERRUPTING`，如果检测发现`s = INTERRUPTING`，说明`cancel`方法还没有执行到中断当前线程的地方，那就等待它将`state`状态设置成`INTERRUPTED`。到这里，对`cancel`方法的分析就和上面对`run`方法的分析对接上了。

`cancel`方法到这里就分析完了，如果一条条的去对照`Future`接口对于`cancel`方法的规范，它每一条都是实现了的，而它实现的核心机理，就是对`state`的当前状态的判断和设置。**由此可见，`state`属性是贯穿整个`FutureTask`的最核心的属性。**

###### isCancelled()

```java
public boolean isCancelled() {
    return state >= CANCELLED;
}
```

`state >= CANCELLED` 包含了 `CANCELLED` `INTERRUPTING` `INTERRUPTED`这三种状态。

再来回忆下上一篇讲的`Future`接口对于`isCancelled()`方法的规范：

> 该方法用于判断任务是否被取消了。如果一个任务在正常执行完成之前被cancel掉了, 则返回true

再对比`state`的状态图:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/FutureTask%E7%8A%B6%E6%80%81%E8%BD%AC%E6%8D%A22.png)

可见选取这三个状态作为判断依据是很合理的, 因为只有调用了`cancel`方法，才会使`state`状态进入这三种状态。

###### isDone()

与`isCancelled`方法类似，`isDone`方法也是简单地通过`state`状态来判断。

```java
public boolean isDone() {
    return state != NEW;
}
```

关于这一点，其实之前已经说过了，只要`state`状态不是`NEW`，则任务已经执行完毕了，因为`state`状态不存在类似“任务正在执行中”这种状态，即使是短暂的中间态，也是发生在任务已经执行完毕，正在设置任务结果的时候。

###### get()

最后来看看获取执行结果的`get`方法，先来看看无参的版本：

```java
public V get() throws InterruptedException, ExecutionException {
    int s = state;
    if (s <= COMPLETING) // 还未完成或者正在设置执行结果
        s = awaitDone(false, 0L);
    return report(s);
}
```

该方法其实很简单，当任务还没有执行完毕或者正在设置执行结果时，就使用`awaitDone`方法等待任务进入终止态，注意，**`awaitDone`的返回值是任务的状态，而不是任务的结果**。任务进入终止态之后，就根据任务的执行结果来返回计算结果或者抛出异常。

先来看看等待任务完成的`awaitDone`方法，该方法是获取任务结果最核心的方法，它完成了获取结果，挂起线程，响应中断等诸多操作：

```java
private int awaitDone(boolean timed, long nanos) throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q); // 移除节点
            throw new InterruptedException();
        }
        int s = state;
        if (s > COMPLETING) { // 任务已完成
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet 正在设置结果
            Thread.yield();
        else if (q == null) // 新建节点
            q = new WaitNode();
        else if (!queued) // 入栈
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else // 阻塞
            LockSupport.park(this);
    }
}
```

在具体分析它的源码之前，有一点先特别说明一下，`FutureTask`中会涉及到两类线程，一类是执行任务的线程，它只有一个，`FutureTask`的`run`方法就由该线程来执行；一类是获取任务执行结果的线程，它可以有多个，这些线程可以并发执行，每一个线程都是独立的，都可以调用`get`方法来获取任务的执行结果。如果任务还没有执行完，则这些线程就需要进入Treiber栈中挂起，直到任务执行结束，或者等待的线程自身被中断。

理清了这一点后，再来详细看看`awaitDone`方法。可以看出，该方法的大框架是一个自旋操作，一段一段来看:

```java
for (;;) {
    if (Thread.interrupted()) {
        removeWaiter(q);
        throw new InterruptedException();
    }
    // ...
}
```

首先一开始，先检测当前线程是否被中断了，这是因为`get`方法是阻塞式的，如果等待的任务还没有执行完，则调用`get`方法的线程会被扔到Treiber栈中挂起等待，直到任务执行完毕。但是，如果任务迟迟没有执行完毕，则也有可能直接中断在Treiber栈中的线程，以停止等待。

当检测到线程被中断后，调用了`removeWaiter`:

```java
private void removeWaiter(WaitNode node) {
    if (node != null) {
        ...
    }
}
```

`removeWaiter`的作用是将参数中的`node`从等待队列（即Treiber栈）中移除。如果此时线程还没有进入Treiber栈，则 `q=null`，那么`removeWaiter(q)`啥也不干。在这之后，就直接抛出了`InterruptedException`异常。

接着往下看：

```java
for (;;) {
    /*if (Thread.interrupted()) {
        removeWaiter(q);
        throw new InterruptedException();
    }*/
    int s = state;
    if (s > COMPLETING) {
        if (q != null)
            q.thread = null;
        return s;
    }
    else if (s == COMPLETING) // cannot time out yet
        Thread.yield();
    else if (q == null)
        q = new WaitNode();
    else if (!queued)
        queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                             q.next = waiters, q);
    else if (timed) {
        nanos = deadline - System.nanoTime();
        if (nanos <= 0L) {
            removeWaiter(q);
            return state;
        }
        LockSupport.parkNanos(this, nanos);
    }
    else
        LockSupport.park(this);
}
```



- 如果任务已经进入终止态（`s > COMPLETING`），就直接返回任务的状态;
- 否则，如果任务正在设置执行结果（`s == COMPLETING`），就让出当前线程的CPU资源，继续等待；
- 否则，就说明任务还没有执行，或者任务正在执行过程中，那么这时，如果q现在还为`null`, 说明当前线程还没有进入等待队列，于是新建了一个`WaitNode`， `WaitNode`的构造函数之前已经看过了，就是生成了一个记录了当前线程的节点；
- 如果q不为`null`，说明代表当前线程的`WaitNode`已经被创建出来了，则接下来如果`queued=false`，表示当前线程还没有入队，所以执行了:

```java
queued = UNSAFE.compareAndSwapObject(this, waitersOffset, q.next = waiters, q);
```

这行代码的作用是通过CAS操作将新建的q节点添加到`waiters`链表的头节点之前，**其实就是Treiber栈的入栈操作**，写的还是很简洁的，一行代码就搞定了，下面是它等价的伪代码：

```java
q.next = waiters; // 当前节点的next指向目前的栈顶元素
// 如果栈顶节点在这个过程中没有变，即没有发生并发入栈的情况
if(waiters的值还是上面q.next所使用的waiters值){ 
    waiters = q; // 修改栈顶的指针，指向刚刚入栈的节点
}
```

这个CAS操作就是为了保证同一时刻如果有多个线程在同时入栈，则只有一个能够操作成功，也即Treiber栈的规范。

如果以上的条件都不满足，则再接下来因为现在是不带超时机制的`get`，`timed`为`false`，则`else if`代码块跳过，然后来到最后一个`else`, 把当前线程挂起，此时线程就处于阻塞等待的状态。

至此，在任务没有执行完毕的情况下，获取任务执行结果的线程就会在Treiber栈中被`LockSupport.park(this)`挂起了。

那么这个挂起的线程什么时候会被唤醒呢？有两种情况：

1. 任务执行完毕了，在`finishCompletion`方法中会唤醒所有在Treiber栈中等待的线程；
2. 等待的线程自身因为被中断等原因而被唤醒。

接下来就继续看看线程被唤醒后的情况，此时，线程将回到`for(;;)`循环的开头，继续下一轮循环：

```java
for (;;) {
    if (Thread.interrupted()) {
        removeWaiter(q);
        throw new InterruptedException();
    }

    int s = state;
    if (s > COMPLETING) {
        if (q != null)
            q.thread = null;
        return s;
    }
    else if (s == COMPLETING) // cannot time out yet
        Thread.yield();
    else if (q == null)
        q = new WaitNode();
    else if (!queued)
        queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                             q.next = waiters, q);
    else if (timed) {
        nanos = deadline - System.nanoTime();
        if (nanos <= 0L) {
            removeWaiter(q);
            return state;
        }
        LockSupport.parkNanos(this, nanos);
    }
    else
        LockSupport.park(this); // 挂起的线程从这里被唤醒
}
```

首先自然还是检测中断，所不同的是，此时q已经不为`null`了，因此在有中断发生的情况下，在抛出中断之前，多了一步`removeWaiter(q)`操作，该操作是将当前线程从等待的Treiber栈中移除，相比入栈操作，这个出栈操作要复杂一点，这取决于节点是否位于栈顶。下面来仔细分析这个出栈操作：

```java
// node出栈
private void removeWaiter(WaitNode node) {
    if (node != null) {
        node.thread = null;
        retry:
        for (;;) {          // restart on removeWaiter race
            for (WaitNode pred = null, q = waiters, s; q != null; q = s) {
                s = q.next;
                if (q.thread != null)
                    pred = q;
                else if (pred != null) {
                    pred.next = s;
                    if (pred.thread == null) // check for race
                        continue retry;
                }
                // node是栈顶节点
                else if (!UNSAFE.compareAndSwapObject(this, waitersOffset, q, s))
                    continue retry;
            }
            break;
        }
    }
}
```

首先，把要出栈的`WaitNode`的`thread`属性设置为`null`, 这相当于一个**标记**，是后面在`waiters`链表中定位该节点的依据。

(1) 要移除的节点就在栈顶

先来看看该节点就位于栈顶的情况，这说明在该节点入栈后，并没有别的线程再入栈了。由于一开始就将该节点的`thread`属性设为了`null`，因此，前面的`q.thread != null` 和 `pred != null`都不满足，直接进入到最后一个`else if `分支：

```
else if (!UNSAFE.compareAndSwapObject(this, waitersOffset, q, s))
    continue retry;
```

这一段是栈顶节点出栈的操作，和入栈类似，采用了CAS比较，将栈顶元素设置成原栈顶节点的下一个节点。

值得注意的是，当CAS操作不成功时，程序会回到`retry`处重来，**但即使CAS操作成功了，程序依旧会遍历完整个链表**，找寻`node.thread == null` 的节点，并将它们一并从链表中剔除。

(2) 要移除的节点不在栈顶

当要移除的节点不在栈顶时，会一直遍历整个链表，直到找到`q.thread == null`的节点，找到之后，将进入

```java
else if (pred != null) {
    pred.next = s;
    if (pred.thread == null) // check for race
        continue retry;
}
```

这是**因为节点不在栈顶，则其必然是有前驱节点pred的**，这时只是简单的让前驱节点指向当前节点的下一个节点，从而将目标节点从链表中剔除。

注意，后面多加的那个if判断是很有必要的，因为`removeWaiter`方法并没有加锁，所以可能有多个线程在同时执行，`WaitNode`的两个成员变量`thread`和`next`都被设置成`volatile`，这保证了它们的可见性，如果在这时发现了`pred.thread == null`，那就意味着它已经被另一个线程标记了，将在另一个线程中被拿出`waiters`链表，而当前目标节点q的原后继节点s现在是接在这个pred节点上的，因此，如果pred已经被其他线程标记为要拿出去的节点，现在这个线程再继续往后遍历就没有什么意义了，所以这时就调到`retry`处，从头再遍历。

如果`pred`节点没有被其他线程标记，那就接着往下遍历，直到整个链表遍历完。

至此，将节点从`waiters`链表中移除的`removeWaiter`操作就分析完了，总结一下该方法：

在该方法中，会传入一个需要移除的节点，会将这个节点的`thread`属性设置成`null`，以标记该节点。然后无论如何，会遍历整个链表，清除那些被标记的节点（只是简单的将节点从链表中剔除）。如果要清除的节点就位于栈顶，则还需要注意重新设置`waiters`的值，指向新的栈顶节点。所以可以看出，虽说`removeWaiter`方法传入了需要剔除的节点，但是事实上它可能剔除的不止是传入的节点，而是所有已经被标记了的节点，这样不仅清除操作容易了些（不需要专门去定位传入的node在哪里），而且提升了效率（可以同时清除所有已经被标记的节点）。

再回到`awaitDone`方法里：

```java
private int awaitDone(boolean timed, long nanos) throws InterruptedException {
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        if (Thread.interrupted()) {
            removeWaiter(q); // 刚刚分析到这里了，接着往下看
            throw new InterruptedException();
        }

        int s = state;
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
        else if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) {
                removeWaiter(q);
                return state;
            }
            LockSupport.parkNanos(this, nanos);
        }
        else
            LockSupport.park(this);
    }
}
```

如果线程不是因为中断被唤醒，则会继续往下执行，此时会再次获取当前的`state`状态。所不同的是，此时q已经不为`null`，`queued`已经为`true`了，所以已经不需要将当前节点再入`waiters`栈了。

至此除非被中断，否则`get`方法会在原地自旋等待(用的是`Thread.yield`，对应于`s == COMPLETING`)或者直接挂起（对应任务还没有执行完的情况），直到任务执行完成。而前面分析`run`方法和`cancel`方法的时候知道，在`run`方法结束后，或者`cancel`方法取消完成后，都会调用`finishCompletion()`来唤醒挂起的线程，使它们得以进入下一轮循环，获取任务执行结果。

最后，等`awaitDone`函数返回后，`get`方法返回了`report(s)`，以根据任务的状态，汇报执行结果:

```java
@SuppressWarnings("unchecked")
private V report(int s) throws ExecutionException {
    Object x = outcome;
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    throw new ExecutionException((Throwable)x);
}
```

可见，`report`方法非常简单，它根据当前`state`状态，返回正常执行的结果，或者抛出指定的异常。

至此，`get`方法就分析结束了。

值得注意的是，`awaitDone`方法和`get`方法都没有加锁，这在多个线程同时执行`get`方法的时候会不会产生线程安全问题呢？通过查看方法内部的参数知道，整个方法内部用的大多数是局部变量，因此不会产生线程安全问题，对于全局的共享变量`waiters`的修改时，也使用了CAS操作，保证了线程安全，而`state`变量本身是`volatile`的，保证了读取时的可见性，因此整个方法调用虽然没有加锁，它仍然是线程安全的。

###### get(long timeout, TimeUnit unit)

最后来看看带超时版本的`get`方法：

```java
public V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    // 还未完成，限时等待；若超时，抛出TimeoutException
    if (s <= COMPLETING && (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    return report(s);
}
```

它和上面不带超时时间的`get`方法很类似，只是在`awaitDone`方法中多了超时检测：

```java
else if (timed) {
    nanos = deadline - System.nanoTime();
    if (nanos <= 0L) {
        removeWaiter(q);
        return state;
    }
    LockSupport.parkNanos(this, nanos);
}
```

即如果指定的超时时间到了，则直接返回，如果返回时，任务还没有进入终止状态，则直接抛出`TimeoutException`异常，否则就像`get()`方法一样，正常的返回执行结果。

### 六、总结

`FutureTask`实现了`Runnable`和`Future`接口，它表示了一个带有任务状态和任务结果的任务，**它的各种操作都是围绕着任务的状态展开的**，值得注意的是，在所有的7个任务状态中，只要不是`NEW`状态，就表示任务已经执行完毕或者不再执行了，并没有表示“任务正在执行中”的状态。

除了代表了任务的`Callable`对象、代表任务执行结果的`outcome`属性，`FutureTask`还包含了一个代表所有等待任务结束的线程的`Treiber`栈，这一点其实和各种锁的等待队列特别像，即如果拿不到锁，则当前线程就会被扔进等待队列中；这里则是如果任务还没有执行结束，则所有等待任务执行完毕的线程就会被扔进`Treiber`栈中，直到任务执行完毕了，才会被唤醒。



### 参考

- [FutureTask源码解析(1)——预备知识](https://segmentfault.com/a/1190000016542779)
- [FutureTask源码解析(2)——深入理解FutureTask](https://segmentfault.com/a/1190000016572591)
- [Treiber Stack简单分析](https://segmentfault.com/a/1190000012463330)

