---
layout:     post
title:      CompletableFuture使用示例
subtitle:   CompletableFuture/CompletionStage
date:       2019-07-10
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
---

# CompletableFuture使用示例

### 一、示例1

```java
// {@link CompletableFuture} 使用实例
public class CompletableFutureTest {

    /* ------主动完成计算join/get/getNow/complete/completeExceptionally------ */

    private static void test01() throws Exception {
        CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
            int i = 1 / 0;
            return 100;
        });
        // join()/get()抛出不同的异常
         future.join();
//        future.get();
    }

    private static void test02() throws Exception {
        final CompletableFuture<Integer> f = compute(); // 未完成的future
        class Client extends Thread {
            CompletableFuture<Integer> f;

            public Client(String name, CompletableFuture<Integer> f) {
                super(name);
                this.f = f;
            }

            @Override
            public void run() {
                try {
                    System.out.println(this.getName() + ": " + f.get());
                } catch (InterruptedException | ExecutionException e) {
                    e.printStackTrace();
                }
            }
        }
        new Client("Client1", f).start();
        new Client("Client2", f).start();
        System.out.println("waiting");
        TimeUnit.SECONDS.sleep(2);
        // 主动完成future
         f.complete(1000);
//        f.completeExceptionally(new RuntimeException("test exception"));
    }

    // 未完成的future
    private static CompletableFuture<Integer> compute() {
        return new CompletableFuture<>();
    }

    /* ------创建CompletableFuture supplyAsync------ */

    private static void test03() {
        CompletableFuture<String> f = CompletableFuture.supplyAsync(() -> {
            // 长时间的异步计算
            return ".00";
        });
    }

    /* ------计算结果完成时的处理whenComplete------*/

    private static void test04() throws Exception {
        CompletableFuture<Void> f1 = CompletableFuture.runAsync(() -> {
            System.out.println("f1 done...");
        });
        CompletableFuture<Double> f2 = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(1);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 1.0;
        });

        f2.whenComplete((result, e) -> {
            System.out.println("result: " + result);
        });

        System.in.read();
    }

    private static class CompletableFutureInnerTest {
        private static Random rand = new Random();
        private static long t = System.currentTimeMillis();

        private static int getMoreData() {
            System.out.println("begin to start compute");
            try {
                Thread.sleep(10000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
            System.out.println("end to start compute. passed " + (System.currentTimeMillis() - t) / 1000 + " seconds");
            return rand.nextInt(1000);
        }

        public static void test05() throws Exception {
            CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(CompletableFutureInnerTest::getMoreData);
            CompletableFuture<Integer> f2 = f1.whenComplete((v, e) -> {
                System.out.println(v);
                System.out.println(e);
            });
            System.out.println(f2.get()); // 返回与f1相同的结果
            System.out.println("end...");
        }
    }

    /* ------转换thenApplyAsync/thenApply------ */

    private static void test06() throws Exception {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> 100)
                .thenApplyAsync(i -> i * 10).thenApply(Object::toString);
        System.out.println(future.get());
    }

    /* ------纯消费(执行Action) thenAccept/thenAcceptBoth/thenRun------*/

    private static void test07() throws Exception {
        CompletableFuture<Void> f = CompletableFuture.supplyAsync(() -> 100).thenAccept(System.out::println);
        System.out.println(f.get());
    }

    private static void test08() throws Exception {
        CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> 100)
                .thenAcceptBoth(CompletableFuture.completedFuture(10), (x, y) -> System.out.println(x * y));
        System.out.println(future.get());
    }

    private static void test09() throws Exception {
        CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> 100)
                .thenRun(() -> System.out.println("finished"));
        System.out.println(future.get());
    }

    /* ------组合thenCompose/thenCombine------*/

    /**
     * {@link CompletableFuture#thenCompose(Function)} future之间有先后依赖顺序
     */
    private static void test10() throws Exception {
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> 100)
                .thenCompose(i -> CompletableFuture.supplyAsync(() -> String.valueOf(i * 10)));
        System.out.println(future.get());
    }

    private static void test11() {
        final String original = "message";
        CompletableFuture<String> cf = CompletableFuture
                .completedFuture(original)
                .thenApply(String::toUpperCase)
                .thenCompose(upper -> CompletableFuture
                        .completedFuture(original)
                        .thenApply(String::toLowerCase)
                        .thenApply(s -> upper + s)
                );
        System.out.println("MESSAGEmessage: " + cf.join());
    }

    /**
     * {@link CompletableFuture#thenCombine(CompletionStage, BiFunction)} 并行执行，future之间没有先后依赖顺序
     */
    private static void test12() throws Exception {
        CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> 100);
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "abc");
        CompletableFuture<String> f = f1.thenCombine(f2, (x, y) -> x + "-" + y);
        System.out.println(f.get());
    }

    /* ------Either applyToEither------*/

    private static void test13() throws Exception {
        Random random = new Random();
        CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(5000 + random.nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 100;
        });
        CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(5000 + random.nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 200;
        });
        // 谁先完成，先用谁
        CompletableFuture<String> f = f1.applyToEither(f2, Object::toString);
        System.out.println(f.get());
    }

    /* ------anyOf 与 allOf------ */

    private static void test14() throws Exception {
        Random random = new Random();
        CompletableFuture<Integer> f1 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(3000 + random.nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 100;
        });
        CompletableFuture<Integer> f2 = CompletableFuture.supplyAsync(() -> {
            try {
                Thread.sleep(3000 + random.nextInt(1000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 200;
        });
        // 谁先完成，就完成计算，返回该结果
        CompletableFuture<Object> f = CompletableFuture.anyOf(f1, f2);
        System.out.println(f.get());
    }

    public static void main(String[] args) throws Exception {

//         test01();
//         test02();
//         test03();
//         test04();
//        CompletableFutureInnerTest.test05();
//         test06();
//         test07();
//         test08();
//         test09();
//         test10();
//         test11();
//         test12();
//         test13();
         test14();
    }

}
```

### 二、示例2

```java
// {@link CompletableFuture} Util
public class CompletableFutureUtil {

    // 将多个CompletableFuture组合成一个CompletableFuture，这个组合后的CompletableFuture的
    // 计算结果是个List,它包含前面所有的CompletableFuture的计算结果
    //
    // @param futures {@link CompletableFuture<T>}
    // @param <T>     Type Parameter
    // @return {@link CompletableFuture<List<T>>}
    public static <T> CompletableFuture<List<T>> sequence(List<CompletableFuture<T>> futures) {
        CompletableFuture<Void> allDoneFutures = CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]));

        return allDoneFutures.thenApply(v -> futures.stream().map(CompletableFuture::join).collect(Collectors.toList()));
    }

     // 将多个CompletableFuture组合成一个CompletableFuture，这个组合后的CompletableFuture的
     // 计算结果是个List,它包含前面所有的CompletableFuture的计算结果
     //
     // @param futures {@link CompletableFuture<T>}
     // @param <T>     Type Parameter
     // @return {@link CompletableFuture<List<T>>}
    public static <T> CompletableFuture<List<T>> sequence(Stream<CompletableFuture<T>> futures) {
        List<CompletableFuture<T>> futureList = futures.filter(Objects::nonNull).collect(Collectors.toList());
        return sequence(futureList);
    }

    // 实现JDK {@code Future<T>} 与 {@code CompletableFuture<T>}的转换
    //
    // @param future   {@link Future}
    // @param executor {@link Executor}
    // @param <T>      Type Parameter
    // @return {@link CompletableFuture<T>}
    public static <T> CompletableFuture<T> toCompletable(Future<T> future, Executor executor) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                return future.get();
            } catch (InterruptedException | ExecutionException e) {
                throw new RuntimeException(e);
            }
        }, executor);
    }

}
```

### 三、示例3

```java
// {@link CompletableFuture} 使用实例2
public class CompletableFutureTest2 {

    // 1.创建完成的CompletableFuture
    private static void completedFutureExample() {
        CompletableFuture<String> future = CompletableFuture.completedFuture("message");
        assertTrue(future.isDone());
        assertEquals("message", future.getNow(null));
    }

   // 2.运行简单的异步场景
   // <p>
   // - CompletableFuture 是异步执行方式.
   // - 使用 ForkJoinPool 实现异步执行，这种方式使用了 daemon 线程执行 Runnable 任务.
    private static void runAsyncExample() {
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
            assertTrue(Thread.currentThread().isDaemon()); // ForkJoin pool daemon线程
            randomSleep();
        });

        assertFalse(future.isDone());
        sleepEnough();
        assertTrue(future.isDone());
    }

    private static void sleepEnough() {
        try {
            TimeUnit.SECONDS.sleep(6);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    private static void randomSleep() {
        Random random = new Random();
        try {
            TimeUnit.SECONDS.sleep(random.nextInt(5) + 1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 3.同步执行动作示例
    private static void thenApplyExample() {
        // 同步执行
        CompletableFuture<String> cf = CompletableFuture.completedFuture("message").thenApply(str -> {
            assertFalse(Thread.currentThread().isDaemon()); // main线程
            return str.toUpperCase();
        });
        assertEquals("MESSAGE", cf.getNow(null));
    }

    // 4.异步执行动作示例
    private static void thenApplyAsyncExample() {
        CompletableFuture<String> cf = CompletableFuture.completedFuture("message").thenApplyAsync(s -> {
            assertTrue(Thread.currentThread().isDaemon()); // ForkJoin pool daemon线程
            randomSleep();
            return s.toUpperCase();
        });
        assertNull(cf.getNow(null));
        assertEquals("MESSAGE", cf.join());
    }

    // 5.使用固定的线程池完成异步执行动作示例
    private static ExecutorService executor = Executors.newFixedThreadPool(3, new ThreadFactory() {
        int count = 1;

        @Override
        public Thread newThread(Runnable runnable) {
            return new Thread(runnable, "custom-executor-" + count++);
        }
    });

    private static void thenApplyAsyncWithExecutorExample() {
        CompletableFuture<String> cf = CompletableFuture.completedFuture("message").thenApplyAsync(s -> {
            assertTrue(Thread.currentThread().getName().startsWith("custom-executor-"));
            assertFalse(Thread.currentThread().isDaemon());
            randomSleep();
            return s.toUpperCase();
        }, executor);
        assertNull(cf.getNow(null));
        assertEquals("MESSAGE", cf.join());
    }

    // 6.作为消费者消费计算结果示例
    private static void thenAcceptExample() {
        StringBuilder sb = new StringBuilder();
        CompletableFuture.completedFuture("thenAccept message").thenAccept(sb::append);
        assertTrue("Result was empty", sb.length() > 0);
    }

    // 7.异步消费示例
    private static void thenAcceptAsyncExample() {
        StringBuilder result = new StringBuilder();
        CompletableFuture<Void> cf = CompletableFuture.completedFuture("thenAcceptAsync message")
                .thenAcceptAsync(result::append);
        cf.join();
        assertTrue("Result was empty", result.length() > 0);
    }

    // 8.运行两个阶段后执行
    private static void runAfterBothExample() {
        String original = "Message";
        StringBuilder result = new StringBuilder();
        CompletableFuture.completedFuture(original).thenApply(String::toUpperCase).runAfterBoth(
                CompletableFuture.completedFuture(original).thenApply(String::toLowerCase),
                () -> result.append("done"));
        assertTrue("Result was empty", result.length() > 0);
    }

    private static void thenAcceptBothExample() {
        String original = "Message";
        StringBuilder result = new StringBuilder();
        CompletableFuture.completedFuture(original).thenApply(String::toUpperCase).thenAcceptBoth(
                CompletableFuture.completedFuture(original).thenApply(String::toLowerCase),
                (s1, s2) -> result.append(s1).append(s2));
        assertEquals("MESSAGEmessage", result.toString());
    }

    public static void main(String[] args) throws Exception {

        // completedFutureExample();
        // runAsyncExample();
        // thenApplyExample();
        // thenApplyAsyncExample();
        // thenApplyAsyncWithExecutorExample();
        // thenAcceptExample();
        // thenAcceptAsyncExample();
        // runAfterBothExample();
        thenAcceptBothExample();

    }

}
```

### 参考

- [JDK CompletionStage](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html)
- [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html)
- [Java CompletableFuture 详解](https://colobu.com/2016/02/29/Java-CompletableFuture/)
- [通过实例理解 JDK8 的 CompletableFuture](https://www.ibm.com/developerworks/cn/java/j-cf-of-jdk8/index.html)