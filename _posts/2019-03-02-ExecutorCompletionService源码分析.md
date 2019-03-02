---
layout:     post
title:      ExecutorCompletionService源码分析
subtitle:   ExecutorCompletionService
date:       2019-03-02
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# ExecutorCompletionService源码分析

`CompletionService`将新异步任务的提交与已完成任务结果的使用分离开来，生产者提交新任务，消费者按照任务完成的顺序取出这些任务并使用其结果。`ExecutorCompletionService`实现了这个`CompletionService`接口，并使用给定的`Executor`执行任务，在任务完成之后，将会把这些任务(`Future`)放置在`BlockingQueue`中，以使用`take`方法取出、使用。

## 一、CompletionService接口

```java
public interface CompletionService<V> {
    // 提交Callable任务以执行，返回Future
    Future<V> submit(Callable<V> task);

    // 提交Runnable任务以执行，返回Future
    Future<V> submit(Runnable task, V result);

    // 取出一个代表已完成任务的Future，没有则阻塞等待
    Future<V> take() throws InterruptedException;

    // 取出一个代表已完成任务的Future，没有则返回null
    Future<V> poll();

    // 取出一个代表已完成任务的Future，没有则最多等待timeout时间，否则返回null
    Future<V> poll(long timeout, TimeUnit unit) throws InterruptedException;
}
```

## 二、ExecutorCompletionService分析

```java
// ExecutorCompletionService类--CompletionService的实现
public class ExecutorCompletionService<V> implements CompletionService<V> {
    private final Executor executor; // 任务执行器
    private final AbstractExecutorService aes; // 如果executor是AbstractExecutorService实例，则aes就是executor；否则为null
    private final BlockingQueue<Future<V>> completionQueue; // 已完成任务存放的队列(按照完成的顺序存放)

    // QueueingFuture为FutureTask扩展(任务)，本身执行结果为null。在自身执行过程中，会执行传入的RunnableFuture任务，并使得该RunnableFuture类型的任务(带有结果)在执行完成后，存入completionQueue队列
    private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null); // 本身执行结果为null
            this.task = task;
        }
        protected void done() { completionQueue.add(task); } // 在本实例执行完成后，将RunnableFuture<V> task存入completionQueue队列
        private final Future<V> task; // RunnableFuture任务
    }

    // 包装Callable为RunnableFuture实例
    private RunnableFuture<V> newTaskFor(Callable<V> task) {
        if (aes == null)
            return new FutureTask<V>(task);
        else
            return aes.newTaskFor(task);
    }

    // 包装Runnable为RunnableFuture实例
    private RunnableFuture<V> newTaskFor(Runnable task, V result) {
        if (aes == null)
            return new FutureTask<V>(task, result);
        else
            return aes.newTaskFor(task, result);
    }

    // 指定executor执行器
    public ExecutorCompletionService(Executor executor) {
        if (executor == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = new LinkedBlockingQueue<Future<V>>(); // 默认队列LinkedBlockingQueue
    }

    // 指定执行器和已完成任务的队列
    public ExecutorCompletionService(Executor executor, BlockingQueue<Future<V>> completionQueue) {
        if (executor == null || completionQueue == null)
            throw new NullPointerException();
        this.executor = executor;
        this.aes = (executor instanceof AbstractExecutorService) ?
            (AbstractExecutorService) executor : null;
        this.completionQueue = completionQueue;
    }

    // 提交Callable任务，返回Future
    public Future<V> submit(Callable<V> task) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task); // 包装Callable为RunnableFuture实例
        executor.execute(new QueueingFuture(f)); // 再将RunnableFuture实例包装为QueueingFuture实例，执行
        return f;
    }

    // 提交Runnable任务，返回Future
    public Future<V> submit(Runnable task, V result) {
        if (task == null) throw new NullPointerException();
        RunnableFuture<V> f = newTaskFor(task, result); // 包装Runnable为RunnableFuture实例
        executor.execute(new QueueingFuture(f)); // 再将RunnableFuture实例包装为QueueingFuture实例，执行
        return f;
    }

    // 从completionQueue取出一个代表已完成任务的Future，没有则阻塞等待
    public Future<V> take() throws InterruptedException {
        return completionQueue.take();
    }

    // 从completionQueue取出一个代表已完成任务的Future，没有则返回null
    public Future<V> poll() {
        return completionQueue.poll();
    }

    // 从completionQueue取出一个代表已完成任务的Future，没有则最多等待timeout时间，否则返回null
    public Future<V> poll(long timeout, TimeUnit unit)
            throws InterruptedException {
        return completionQueue.poll(timeout, unit);
    }
}
```

## 三、ExecutorCompletionService实例

```java
// JDK8提供的例子
// 假设有针对某个问题的一组求解程序(Collection<Callable<Result>> solvers)，每个求解程序都能返回某种类型的Result值，并且想同时运行它们，使用方法use(Result r)处理返回非null值的每个求解程序(Callable<Result>)的返回结果。
void solve(Executor e, Collection<Callable<Result>> solvers)
     throws InterruptedException, ExecutionException {
     CompletionService<Result> ecs = new ExecutorCompletionService<Result>(e);
     for (Callable<Result> s : solvers)
         ecs.submit(s); // 任务提交
     int n = solvers.size();
     for (int i = 0; i < n; ++i) {
         Result r = ecs.take().get(); // 循环等待每一个任务完成
         if (r != null)
             use(r); // 使用结果
     }
}

// 如果想使用任务集中的第一个非null结果，而忽略任何遇到异常的任务，并且在第一个任务完成时取消其他所有任务的执行：
void solve(Executor e, Collection<Callable<Result>> solvers)
     throws InterruptedException {
     CompletionService<Result> ecs = new ExecutorCompletionService<Result>(e);
     int n = solvers.size();
     List<Future<Result>> futures = new ArrayList<Future<Result>>(n);
     Result result = null;
     try {
         for (Callable<Result> s : solvers)
             futures.add(ecs.submit(s)); // 提交任务，将Future存入List<Future<Result>> futures
         for (int i = 0; i < n; ++i) {
             try {
                 Result r = ecs.take().get();
                 if (r != null) { // 获取任务集中的第一个非null结果(正常执行完成)
                     result = r;
                     break;
                 }
             } catch (ExecutionException ignore) {} // 对于任务结果抛出异常的，忽略
         }
     } finally {
         for (Future<Result> f : futures)
             f.cancel(true); // 第一个任务完成时取消其他所有任务的执行
     }

     if (result != null)
         use(result); // 使用结果
}
```

