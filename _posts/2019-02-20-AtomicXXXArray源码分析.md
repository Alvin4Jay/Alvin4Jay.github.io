---
layout:     post
title:      AtomicXXXArray源码分析
subtitle:   AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray
date:       2019-02-20
author:     Jay
header-img: img/post-bg-swift2.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# AtomicXXXArray源码分析

`AtomicXXXArray`主要包括`AtomicIntegerArray、AtomicLongArray、AtomicReferenceArray`这三个类，底层操作同样是`Unsafe`这个类，关于这个类的使用可参考[sun.misc.Unsafe使用指南](https://xuanjian1992.top/2018/12/09/Unsafe%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97/)这篇文章。由于三者原理类似，下面以`AtomicLongArray`为例进行实现分析。

### 一、源码分析

```java
public class AtomicLongArray implements java.io.Serializable {
    private static final long serialVersionUID = -2308431214976778248L;

    // 基于Unsafe的CAS操作实现数组元素值的原子更新
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // arrayBaseOffset获取array数组对象在内存中从数组的地址到首个元素的地址的偏移量
    private static final int base = unsafe.arrayBaseOffset(long[].class);
    private static final int shift; // 1 << shift代表数组中每个元素在内存中的大小(scale)
    private final long[] array; // 数组，final保证内存可见性

    static {
        // arrayIndexScale获取array数组中每个元素在内存中的大小。1，2，4，8等(字节)
        int scale = unsafe.arrayIndexScale(long[].class);
        if ((scale & (scale - 1)) != 0) // scale必须是2的幂次方
            throw new Error("data type scale not a power of two");
        // Integer.numberOfLeadingZeros(scale)这个方法获取scale中高位的0的个数，
        // 得到的shift表示第一个不为0的index，shift可以理解为表示scale的2幂次方的这个幂。
        // 1 << shift就表示数组中每个元素在内存中的大小，所以byteOffset(i)方法中
        // ((long) i << shift) + base表示在内存中从数组的地址开始到索引为i的数组元素的偏移量
        shift = 31 - Integer.numberOfLeadingZeros(scale); 
    }

    // 检查数组元素索引i是否越界，若未越界，返回在内存中从数组的地址开始到索引为i的数组元素的偏移量
    private long checkedByteOffset(int i) {
        if (i < 0 || i >= array.length)
            throw new IndexOutOfBoundsException("index " + i);

        return byteOffset(i);
    }

    // 返回在内存中从数组的地址开始到索引为i的数组元素的偏移量
    private static long byteOffset(int i) {
        return ((long) i << shift) + base;
    }

    // 创建AtomicLongArray实例，array数组元素都为0
    public AtomicLongArray(int length) {
        array = new long[length];
    }

    // 创建AtomicLongArray实例，元素拷贝自入参array
    public AtomicLongArray(long[] array) {
        this.array = array.clone();
    }

    // 数组长度
    public final int length() {
        return array.length;
    }

   	// 获取索引为i的数组元素值
    public final long get(int i) {
        return getRaw(checkedByteOffset(i));
    }

    // 获取内存偏移量为offset的数组元素值
    private long getRaw(long offset) {
        return unsafe.getLongVolatile(array, offset);
    }

    // 原子地设置索引为i的数组元素值为newValue
    public final void set(int i, long newValue) {
        unsafe.putLongVolatile(array, checkedByteOffset(i), newValue);
    }

    // 最终设置索引为i的数组元素值为newValue，不保证值的改变被其他线程立即看到
    public final void lazySet(int i, long newValue) {
        unsafe.putOrderedLong(array, checkedByteOffset(i), newValue);
    }

    // 原子地设置索引为i的数组元素值为newValue，返回旧值
    public final long getAndSet(int i, long newValue) {
        return unsafe.getAndSetLong(array, checkedByteOffset(i), newValue);
    }

    // CAS，如果索引i处的数组元素值为expect，则原子地更新为update
    public final boolean compareAndSet(int i, long expect, long update) {
        return compareAndSetRaw(checkedByteOffset(i), expect, update);
    }

    // CAS，如果内存偏移量为offset处的值为expect，则原子地更新为update
    private boolean compareAndSetRaw(long offset, long expect, long update) {
        return unsafe.compareAndSwapLong(array, offset, expect, update);
    }

    // JDK1.8实现与compareAndSet相同
    public final boolean weakCompareAndSet(int i, long expect, long update) {
        return compareAndSet(i, expect, update);
    }

    // 原子地将索引为i处的数组元素值加1，返回旧值
    public final long getAndIncrement(int i) {
        return getAndAdd(i, 1);
    }

    // 原子地将索引为i处的数组元素值减1，返回旧值
    public final long getAndDecrement(int i) {
        return getAndAdd(i, -1);
    }

    // 原子地将索引为i处的数组元素值加delta，返回旧值
    public final long getAndAdd(int i, long delta) {
        return unsafe.getAndAddLong(array, checkedByteOffset(i), delta);
    }

    // 原子地将索引为i处的数组元素值加1，返回新值
    public final long incrementAndGet(int i) {
        return getAndAdd(i, 1) + 1;
    }

    // 原子地将索引为i处的数组元素值减1，返回新值
    public final long decrementAndGet(int i) {
        return getAndAdd(i, -1) - 1;
    }

    // 原子地将索引为i处的数组元素值加delta，返回新值
    public long addAndGet(int i, long delta) {
        return getAndAdd(i, delta) + delta;
    }

    // 将updateFunction应用于索引为i处的数组元素值，得到新值，并原子地更新，返回旧值
    public final long getAndUpdate(int i, LongUnaryOperator updateFunction) {
        long offset = checkedByteOffset(i); // 元素内存偏移量
        long prev, next;
        do {
            prev = getRaw(offset); // 当前值
            next = updateFunction.applyAsLong(prev); // 新值
        } while (!compareAndSetRaw(offset, prev, next)); // CAS更新
        return prev; // 返回旧值
    }

    // 逻辑与getAndUpdate(int i, LongUnaryOperator updateFunction)一致，返回新值
    public final long updateAndGet(int i, LongUnaryOperator updateFunction) {
        long offset = checkedByteOffset(i);
        long prev, next;
        do {
            prev = getRaw(offset);
            next = updateFunction.applyAsLong(prev);
        } while (!compareAndSetRaw(offset, prev, next));
        return next;
    }

    // 以x和索引为i处的数组元素值为参数，应用accumulatorFunction，得到新值，并CAS原子
    // 地更新数组元素，返回旧值
    public final long getAndAccumulate(int i, long x,
                                      LongBinaryOperator accumulatorFunction) {
        long offset = checkedByteOffset(i); // 元素内存偏移量
        long prev, next;
        do {
            prev = getRaw(offset); // 当前值
            next = accumulatorFunction.applyAsLong(prev, x); // 新值
        } while (!compareAndSetRaw(offset, prev, next)); // CAS更新
        return prev; // 返回旧值
    }

    // 逻辑与updateAndGet(int i, LongUnaryOperator updateFunction)一致，返回新值
    public final long accumulateAndGet(int i, long x,
                                      LongBinaryOperator accumulatorFunction) {
        long offset = checkedByteOffset(i);
        long prev, next;
        do {
            prev = getRaw(offset);
            next = accumulatorFunction.applyAsLong(prev, x);
        } while (!compareAndSetRaw(offset, prev, next));
        return next;
    }

    // 该实例的字符串表示，包含array数组元素值。
    public String toString() {
        int iMax = array.length - 1;
        if (iMax == -1)
            return "[]";

        StringBuilder b = new StringBuilder();
        b.append('[');
        for (int i = 0; ; i++) {
            b.append(getRaw(byteOffset(i)));
            if (i == iMax)
                return b.append(']').toString();
            b.append(',').append(' ');
        }
    }
}
```

### 二、参考文章

- [java-juc-原子类-AtomicIntegerArray初探(重要)](http://cruise1008.github.io/2016/04/02/java-juc-%E5%8E%9F%E5%AD%90%E7%B1%BB-AtomicIntegerArray%E5%88%9D%E6%8E%A2/)
- [sun.misc.Unsafe使用指南](https://xuanjian1992.top/2018/12/09/Unsafe%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97/)