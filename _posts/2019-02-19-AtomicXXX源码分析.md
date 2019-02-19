---
layout:     post
title:      AtomicXXX源码分析
subtitle:   AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference
date:       2019-02-19
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# AtomicXXX源码分析

- `AtomicXXX`主要包含`AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference`这几种类型，这几种类型的变量相对于普通变量的区别，主要体现在读写的线程安全上。对原子变量的写是原子的(比如多线程下的共享变量`i++`就不是原子的)，**由CAS操作保证原子性**。对原子量的读可以读到最新值，**由`volatile`关键字来保证可见性**。
- 原子变量多用于数据统计(如接口调用次数)、一些序列号生成(多线程环境下)以及一些同步数据结构中。

此外，原子变量的底层操作来自于`sun.misc.Unsafe`这个类，关于这个类的使用可参考[sun.misc.Unsafe使用指南](https://xuanjian1992.top/2018/12/09/Unsafe%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97/)这篇文章。由于上述四种类型的原理都类似，以下为`AtomicLong`为例进行源码分析。

```java
public class AtomicLong extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 1927816293512124184L;

    // 基于Unsafe的CAS操作实现变量的原子更新
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // AtomicLong实例的volatile long类型变量value的内存偏移量
    private static final long valueOffset;

    /**
     * Records whether the underlying JVM supports lockless
     * compareAndSwap for longs. While the Unsafe.compareAndSwapLong
     * method works in either case, some constructions should be
     * handled at Java level to avoid locking user-visible locks.
     */
    // 记录底层JVM是否支持long类型变量的无锁化CAS操作
    static final boolean VM_SUPPORTS_LONG_CAS = VMSupportsCS8();

    /**
     * Returns whether underlying JVM supports lockless CompareAndSet
     * for longs. Called only once and cached in VM_SUPPORTS_LONG_CAS.
     */
    // 判断底层JVM是否支持long类型变量的无锁化CAS操作
    private static native boolean VMSupportsCS8();

    static {
        try {
            // 计算value变量的内存偏移量
            valueOffset = unsafe.objectFieldOffset
                (AtomicLong.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
	// volatile变量，保证内存可见性
    private volatile long value;

    // 创建初始值为initialValue的AtomicLong实例
    public AtomicLong(long initialValue) {
        value = initialValue;
    }

    // 创建初始值为0的AtomicLong实例
    public AtomicLong() {
    }

    // 获取当前值value
    public final long get() {
        return value;
    }

    // 将当前值设置为newValue
    public final void set(long newValue) {
        value = newValue;
    }

    // 最终将value设置为newValue，lazySet(unsafe.putOrderedLong)为延迟操作，
    // 并且不保证值的改变被其他线程立即看到
    public final void lazySet(long newValue) {
        unsafe.putOrderedLong(this, valueOffset, newValue);
    }

    // 原子地将value设置为newValue，返回旧值
    public final long getAndSet(long newValue) {
        return unsafe.getAndSetLong(this, valueOffset, newValue);
    }

    // CAS，如果当前value=expect，则原子地将当前value设置为update
    public final boolean compareAndSet(long expect, long update) {
        return unsafe.compareAndSwapLong(this, valueOffset, expect, update);
    }

    // JDK8实现与compareAndSet(long expect, long update)一致
    public final boolean weakCompareAndSet(long expect, long update) {
        return unsafe.compareAndSwapLong(this, valueOffset, expect, update);
    }

    // 原子地将value加1，返回旧值
    public final long getAndIncrement() {
        return unsafe.getAndAddLong(this, valueOffset, 1L);
    }

    // 原子地将value减1，返回旧值
    public final long getAndDecrement() {
        return unsafe.getAndAddLong(this, valueOffset, -1L);
    }

    // 原子地将value加delta，返回旧值
    public final long getAndAdd(long delta) {
        return unsafe.getAndAddLong(this, valueOffset, delta);
    }

    // 原子地将value加1，返回最新值
    public final long incrementAndGet() {
        return unsafe.getAndAddLong(this, valueOffset, 1L) + 1L;
    }

    // 原子地将value减1，返回最新值
    public final long decrementAndGet() {
        return unsafe.getAndAddLong(this, valueOffset, -1L) - 1L;
    }

    // 原子地将value加delta，返回最新值
    public final long addAndGet(long delta) {
        return unsafe.getAndAddLong(this, valueOffset, delta) + delta;
    }

    // 将updateFunction应用到当前值，得到欲更新的值，并CAS原子化地更新，返回旧值
    public final long getAndUpdate(LongUnaryOperator updateFunction) {
        long prev, next;
        do {
            prev = get(); // 获取当前值
            next = updateFunction.applyAsLong(prev); // 以当前值prev为参数，调用applyAsLong，得到next，以next为更新的值
        } while (!compareAndSet(prev, next)); // CAS，直到成功
        return prev; // 返回旧值
    }

    // 将updateFunction应用到当前值，得到欲更新的值，并CAS原子化地更新，返回新值
    // 逻辑与getAndUpdate(LongUnaryOperator updateFunction)一致
    public final long updateAndGet(LongUnaryOperator updateFunction) {
        long prev, next;
        do {
            prev = get();
            next = updateFunction.applyAsLong(prev);
        } while (!compareAndSet(prev, next));
        return next;
    }

    // 以当前值和x为参数，应用accumulatorFunction，得到欲更新的值，并CAS原子化地更新，返回旧值
    public final long getAndAccumulate(long x, LongBinaryOperator accumulatorFunction) {
        long prev, next;
        do {
            prev = get(); // 当前值
            next = accumulatorFunction.applyAsLong(prev, x); // 欲更新的值
        } while (!compareAndSet(prev, next)); // CAS更新
        return prev; // 返回旧值
    }

    // 以当前值和x为参数，应用accumulatorFunction，得到欲更新的值，并CAS原子化地更新，返回新值
    // 逻辑与getAndAccumulate(long x, LongBinaryOperator accumulatorFunction)一致
    public final long accumulateAndGet(long x, LongBinaryOperator accumulatorFunction) {
        long prev, next;
        do {
            prev = get();
            next = accumulatorFunction.applyAsLong(prev, x);
        } while (!compareAndSet(prev, next));
        return next;
    }

    // 该实例的字符串表示
    public String toString() {
        return Long.toString(get());
    }

    // 转为int类型值
    public int intValue() {
        return (int)get();
    }

    // long类型值
    public long longValue() {
        return get();
    }

    // 转为float类型值
    public float floatValue() {
        return (float)get();
    }

    // 转为double类型值
    public double doubleValue() {
        return (double)get();
    }

}
```