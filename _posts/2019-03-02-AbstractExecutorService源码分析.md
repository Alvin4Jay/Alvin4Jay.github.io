---
layout:     post
title:      AbstractExecutorService源码分析
subtitle:   AbstractExecutorService
date:       2019-03-02
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# AbstractExecutorService源码分析

`AbstractExecutorService`的类图如下:

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/AbstractExecutorService%E7%B1%BB%E5%9B%BE.png?x-oss-process=style/markdown-pic)

`AbstractExecutorService`抽象类实现了`Executor`和`ExecutorService`接口，提供了`ExecutorService`执行方法的默认实现。`AbstractExecutorService`使用`newTaskFor`方法返回的`RunnableFuture`(接口)实例实现了`submit/invokeAny/ invokeAll`方法。默认`RunnableFuture`实现是`FutureTask`类。

## 一、AbstractExecutorService实现分析

```java
// 抽象类
public abstract class AbstractExecutorService implements ExecutorService {

    // 包装Runnable任务和结果为RunnableFuture实例(默认为FutureTask实例，子类可以重写newTaskFor方法)
    protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
        return new FutureTask<T>(runnable, value);
    }

    // 包装Callable任务为RunnableFuture实例(默认为FutureTask实例，子类可以重写newTaskFor方法)
    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
        return new FutureTask<T>(callable);
    }

    // 提交Runnable任务以执行，返回代表该任务的RunnableFuture实例(默认为FutureTask实例)
    public Future<?> submit(Runnable task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<Void> ftask = newTaskFor(task, null); // 包装task
        execute(ftask); // 执行任务
        return ftask;
    }

    // 提交Runnable任务以执行，返回代表该任务的RunnableFuture实例(默认为FutureTask实例)
    public <T> Future<T> submit(Runnable task, T result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task, result);// 包装task和结果
        execute(ftask); // 执行任务
        return ftask;
    }

    // 提交Callable任务以执行，返回代表该任务的RunnableFuture实例(默认为FutureTask实例)
    public <T> Future<T> submit(Callable<T> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<T> ftask = newTaskFor(task); // 包装task
        execute(ftask); // 执行任务
        return ftask;
    }

    // invokeAny方法的核心实现逻辑
    private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks, boolean timed, long nanos) throws InterruptedException, ExecutionException, TimeoutException {
        if (tasks == null)
            throw new NullPointerException();
        int ntasks = tasks.size(); // 待提交任务的个数
        if (ntasks == 0)
            throw new IllegalArgumentException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks); // Future列表
        // ExecutorCompletionService中按照任务完成的顺序保存了这些任务的Future
        ExecutorCompletionService<T> ecs = new ExecutorCompletionService<T>(this);

        try {
            // 如果没有一个任务正常完成，这个ee变量用来记录最后一个ExecutionException异常，并抛出
            ExecutionException ee = null;
            // 根据是否用到超时机制，得到截止时间deadline
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Iterator<? extends Callable<T>> it = tasks.iterator();  // 任务的迭代器

            futures.add(ecs.submit(it.next())); // 先提交一个任务执行，剩余的慢慢增加
            --ntasks; // 待提交任务减1
            int active = 1; // 执行中的任务数

            for (;;) {
                Future<T> f = ecs.poll(); // 尝试获取一个已完成任务的Future
                if (f == null) { // 没有已完成的任务
                    if (ntasks > 0) { // 还有待提交任务，则先提交执行
                        --ntasks;
                        futures.add(ecs.submit(it.next()));
                        ++active;
                    }
                    else if (active == 0) // f=null，执行中的任务数为0，则所有任务都已完成，直接跳出循环
                        break;
                    else if (timed) {
                        // 超时版本，超时等待获取已完成任务的Future
                        f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                        if (f == null) 
                            // 超时了没获取到Future，抛出TimeoutException，未完成任务被取消
                            throw new TimeoutException();
                        nanos = deadline - System.nanoTime(); // 剩余的时间
                    }
                    else
                        f = ecs.take(); // 非超时版本，等待获取已完成任务的Future
                }
                if (f != null) {
                    --active; // 得到一个已完成任务的Future，执行中的任务数减1
                    try {
                        return f.get(); // 如果该任务是正常完成，没抛出异常，则直接返回
                    } catch (ExecutionException eex) {
                        ee = eex; // 这个已完成任务执行中抛出了异常，则只记录该异常
                    } catch (RuntimeException rex) {
                        // 其它运行时异常，比如任务被取消而完成，获取结果时抛出的CancellationException异常等，包装为ExecutionException
                        ee = new ExecutionException(rex);
                    }
                }
            }

            if (ee == null) // 所有任务都已完成，最终没有正常完成的任务，抛出ExecutionException
                ee = new ExecutionException();
            throw ee;
        } finally {
            // 当前线程在等待获取已完成任务的Future时被中断，抛出InterruptedException；在等待获取已完成任务的Future时超时，抛出TimeoutException，此时未完成任务被取消
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
        }
    }

    // 执行给定的任务集，在这些任务中只要有一个正常完成了就返回其结果T，不包括执行过程中抛出异常而终止或因为取消而终止的任务
    // 在其中一个任务正常完成或抛出异常(线程被中断)之后，未完成的任务被取消
    // 没有任务正常执行完成，抛出ExecutionException；当前线程在等待获取已完成任务的Future时被中断，抛出InterruptedException
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException {
        try {
            return doInvokeAny(tasks, false, 0); // 非超时版本
        } catch (TimeoutException cannotHappen) { // 不会抛出TimeoutException
            assert false;
            return null;
        }
    }

    // 执行给定的任务集，在这些任务中只要有一个正常完成了并且未超时就返回其结果T，不包括执行过程中抛出异常而终止或因为取消而终止的任务
    // 在其中一个任务正常完成或抛出异常(线程被中断、获取Future超时)之后，未完成的任务被取消
    // 没有任务正常执行完成，抛出ExecutionException；当前线程在等待获取已完成任务的Future时被中断，抛出InterruptedException；在等待获取已完成任务的Future时超时，抛出TimeoutException
    public <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException {
        return doInvokeAny(tasks, true, unit.toNanos(timeout));
    }
	
    // 执行给定的任务集，在所有任务都完成后返回代表这些任务状态和结果的Future列表。
    // 对于每个返回的Future，isDone()返回true。
    // 任务完成指正常完成、任务取消或任务执行时抛出异常而终止。
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size()); // Future列表
        boolean done = false; // 所有任务是否已完成
        try {
            for (Callable<T> t : tasks) {
                RunnableFuture<T> f = newTaskFor(t); // 每个任务包装成RunnableFuture实例
                futures.add(f); 
                execute(f); // 执行
            }
            for (int i = 0, size = futures.size(); i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) { // 取出每个Future，判断每个任务是否已完成
                    try {
                        f.get(); // 没完成就等待；正常完成则获取结果
                    } catch (CancellationException ignore) { // 任务被取消也是完成状态
                    } catch (ExecutionException ignore) { // 任务执行过程中抛出异常而终止也是完成状态
                    }
                }
            }
            done = true; // 所有任务已完成
            return futures;
        } finally {
            // 在等待任务完成过程中，有可能当前线程被中断，抛出了InterruptedException异常。
        	// 此时done=false，则将未完成的任务取消
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }

    // (invokeAll限时版本)
    // 执行给定的任务集，在所有任务完成后或者超时后(无论谁先发生)返回代表这些任务状态和结果的Future列表。
    // 对于每个返回的Future，isDone()返回true。返回时未完成的任务被取消。
    // 任务完成指正常完成、任务取消或任务执行时抛出异常而终止。
    public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                         long timeout, TimeUnit unit)
        throws InterruptedException {
        if (tasks == null)
            throw new NullPointerException();
        long nanos = unit.toNanos(timeout); // 转为纳秒
        ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size()); // Future列表
        boolean done = false; // 所有任务是否已完成
        try {
            for (Callable<T> t : tasks)
                futures.add(newTaskFor(t)); // 每个任务包装成RunnableFuture实例，存入Future列表

            final long deadline = System.nanoTime() + nanos; // 截止时间(纳秒)
            final int size = futures.size();

            // 提交任务以执行
            for (int i = 0; i < size; i++) {
                execute((Runnable)futures.get(i));
                nanos = deadline - System.nanoTime(); 
                if (nanos <= 0L) // 提交任务执行期间超时则返回，此时未完成的任务被取消
                    return futures;
            }

            for (int i = 0; i < size; i++) {
                Future<T> f = futures.get(i);
                if (!f.isDone()) { // 取出每个Future，判断每个任务是否已完成
                    if (nanos <= 0L) // 剩余时间nanos<=0，且存在任务未完成，直接返回，此时未完成的任务被取消
                        return futures;
                    try {
                        // 没完成就等待nanos时间；等待时间内正常完成则获取结果
                        f.get(nanos, TimeUnit.NANOSECONDS);
                    } catch (CancellationException ignore) { // 等待时间内任务被取消也是完成状态
                    } catch (ExecutionException ignore) { // 等待时间内，任务执行过程中抛出异常而终止也是完成状态
                    } catch (TimeoutException toe) { // 等待获取结果超时则返回，此时未完成的任务被取消
                        return futures;
                    }
                    nanos = deadline - System.nanoTime(); // 还剩余的时间nanos
                }
            }
            done = true; // 所有任务已完成
            return futures;
        } finally {
            // 当前线程在等待任务完成过程中被中断，抛出了InterruptedException异常或等待任务超时
            // 此时done=false，则将未完成的任务取消
            if (!done)
                for (int i = 0, size = futures.size(); i < size; i++)
                    futures.get(i).cancel(true);
        }
    }

}
```

## 二、AbstractExecutorService实例

```java
// 自定义线程池，在提交Runnable任务时(submit(Runnable) )会创建RunnableFuture实例，原本默认为
// FutureTask实例，这里自定义了CustomTask类，替代FutureTask
public class CustomThreadPoolExecutor extends ThreadPoolExecutor {
	// 自定义任务类，实现RunnableFuture接口，替代FutureTask
    static class CustomTask<V> implements RunnableFuture<V> {...}

    // 重写newTaskFor方法
    protected <V> RunnableFuture<V> newTaskFor(Callable<V> c) {
        return new CustomTask<V>(c);
    }
    protected <V> RunnableFuture<V> newTaskFor(Runnable r, V v) {
        return new CustomTask<V>(r, v);
    }
    // ... add constructors, etc.
}
```