---
layout:     post
title:      LongAdder源码分析
subtitle:   LongAdder/LongAccumulator/DoubleAdder/DoubleAccumulator
date:       2019-02-22
author:     Jay
header-img: img/post-bg-swift.jpg
catalog: true
tags:
    - Java基础
    - JUC
---

# LongAdder源码分析

`LongAdder`是jdk8新增的用于并发环境的计数器，目的是为了在高并发情况下，代替`AtomicLong/AtomicInteger`，成为一个用于高并发情况下的高效的通用计数器。`LongAdder`与`LongAccumulator/DoubleAdder/DoubleAccumulator`类似，都继承于`Striped64`父类，依赖于其核心逻辑。

下面先分析父类`Striped64`的逻辑，再以`LongAder`为例，分析其原理。

### 一、核心实现Striped64

`LongAdder/LongAccumulator/DoubleAdder/DoubleAccumulator`这四个类的核心实现都在`Striped64`中，这个类使用分段的思想，来尽量平摊并发压力。`Striped64`中使用了一个叫`Cell`的类，是一个普通的二元算术累积单元，线程是通过`hash`取模操作映射到一个`Cell`上进行累积(递增)。为了加快取模运算效率，把`Cell`数组的大小设置为`2^n`，同时大量使用`Unsafe`提供的底层操作。

##### 1.累积单元Cell

```java
// 这个类可以看成是一个简化的AtomicLong，通过CAS操作来更新value的值
// @sun.misc.Contended注解，避免伪共享
@sun.misc.Contended static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }

    // Unsafe mechanics Unsafe机制，初始化
    private static final sun.misc.Unsafe UNSAFE;
    private static final long valueOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> ak = Cell.class;
            valueOffset = UNSAFE.objectFieldOffset
                (ak.getDeclaredField("value"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

##### 2.Striped64代码主体 

```java
abstract class Striped64 extends Number {
    // Cell类
    @sun.misc.Contended static final class Cell {}
    
    // CPU个数，限制Cell数组大小
    static final int NCPU = Runtime.getRuntime().availableProcessors();

    // Cell数组，长度是2^n
    transient volatile Cell[] cells;

    // 累积器的基本值，在两种情况下会使用:
    // 1、没有遇到并发的情况，直接使用base，速度更快；
    // 2、多线程并发初始化table数组时，必须要保证table数组只被初始化一次，因此只有一个线程能够竞争成功，这种情况下竞争失败的线程会尝试在base上进行一次累积操作
    transient volatile long base;

    // 自旋标识，在对cells进行初始化，或者后续扩容时，需要通过CAS操作把此标识设置为1（busy，忙标识，相当于加锁），取消busy时可以直接使用cellsBusy = 0，相当于释放锁
    transient volatile int cellsBusy;

    Striped64() {
    }

    // 使用CAS更新base的值
    final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }

    // 使用CAS将cells自旋标识更新为1
    // 更新为0时可以不用CAS，直接使用cellsBusy就行
    final boolean casCellsBusy() {
        return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
    }

    // 下面这两个方法是ThreadLocalRandom中的方法，不过因为包访问关系，这里又重新写一遍
    
    // probe翻译过来是探测/探测器/探针这些，不好理解，它是ThreadLocalRandom里面的一个属性，
    // 不过并不影响对Striped64的理解，这里可以把它理解为线程本身的hash值
    static final int getProbe() {
        return UNSAFE.getInt(Thread.currentThread(), PROBE);
    }

    // 相当于rehash，重新算一遍线程的hash值
    static final int advanceProbe(int probe) {
        probe ^= probe << 13;   // xorshift
        probe ^= probe >>> 17;
        probe ^= probe << 5;
        UNSAFE.putInt(Thread.currentThread(), PROBE, probe);
        return probe;
    }

    /**
     * 核心实现，此方法建议在外部进行一次CAS操作（cells == null时尝试CAS更新base值，cells != null时，CAS更新hash值取模后对应的cell.value）
     * @param x 前面说的二元运算中的第二个操作数，也就是外部提供的那个操作数
     * @param 外部提供的二元算术操作，实例持有并且只能有一个，生命周期内保持不变，null代表LongAdder这种特殊但是最常用的情况，可以减少一次方法调用
     * @param wasUncontended false if CAS failed before call 如果为false，表明调用者预先调用的一次CAS操作都失败了，即出现了竞争
     */
    final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
        int h; // 当前线程的hash probe
        // 这个if相当于给线程生成一个非0的hash值，
        // 如果当前线程的hash probe未初始化则去初始化。
        if ((h = getProbe()) == 0) {
            ThreadLocalRandom.current(); // force initialization 初始化
            h = getProbe();
            // hash重新选取以后，视为未竞争过
            wasUncontended = true; 
        }
        // 此值可以看作是扩容意向
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            Cell[] as; Cell a; int n; long v;
            if ((as = cells) != null && (n = as.length) > 0) { // cells已经被初始化了
				// hash取模映射得到的Cell单元还为null（为null表示还没有被使用）
                if ((a = as[(n - 1) & h]) == null) {
                    // 当前没有线程获取锁
                    if (cellsBusy == 0) {       // Try to attach new Cell
                        Cell r = new Cell(x);   // Optimistically create  先创建新的累积单元
                        if (cellsBusy == 0 && casCellsBusy()) { // 尝试加锁
                            boolean created = false;
                            try {               // 在有锁的情况下再检测一遍之前的判断
                                Cell[] rs; int m, j;
                                if ((rs = cells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r; // 如果通过,就将cells对应位置设置为新创建的Cell对象
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0; // 释放锁
                            }
                            if (created) // 新Cell创建成功即累加成功，返回
                                break;
                            // 未通过双重检测，有其他线程在同一个cells位置新增了cell
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                // 该方法调用前的一次CAS更新a.value（进行一次递增）的尝试已经失败了，说明已经发生了
                // 线程竞争，本次不再尝试CAS递增。而是在调用advanceProbe rehash线程hash以后再去
                // 重新尝试，advanceProbe会修改probe的值，下次循环会选取不同的Cell去尝试CAS递增。
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                             fn.applyAsLong(v, x))))
					// 尝试去递增，成功就退出循环，返回
                    break;
                else if (n >= NCPU || cells != as)
                    // 限制cells不得大于或等于cpu核心数，超过了就不再对cells扩容
	                // cells!=as表明最近被扩容过
    	            // 以上满足任何一个都去标记collide为false，避免扩容
                    collide = false;            // At max size or stale
                else if (!collide)
                    // 执行到这里证明满足扩容需要的条件，设置collide为true，在下次循环中去扩容，
                    // 如果那时还满足扩容条件的话
                    collide = true;
                else if (cellsBusy == 0 && casCellsBusy()) {
                    // 最后再考虑扩容，能到这一步说明竞争很激烈，尝试加锁进行扩容
                    try {
                        // 检查下是否被别的线程扩容了
                        if (cells == as) {      // Expand table unless stale
                            Cell[] rs = new Cell[n << 1]; // 执行2倍扩容
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            cells = rs;
                        }
                    } finally {
                        cellsBusy = 0; // 释放锁
                    }
                    collide = false; // 扩容意向为false
                    continue;    // Retry with expanded table 扩容后重头再来
                }
                // 重新给线程生成一个hash值，降低hash冲突，减少映射到同一个Cell导致CAS竞争的情况
                h = advanceProbe(h);
            }
            // cells没有被加锁，并且它没有被初始化，那么就尝试对它进行加锁，加锁成功进入这个else if
            else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
                boolean init = false;
                try {                           // Initialize table 初始化
                    if (cells == as) { // 双重检测，如果还是null，或者空数组，那么就执行初始化
                        Cell[] rs = new Cell[2]; // 第一次初始化容量为2
                        rs[h & 1] = new Cell(x); // 对其中一个单元进行递增操作，另一个不管，继续为null
                        cells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0; // 释放锁
                }
                if (init) // 初始化的同时也完成了递增，直接返回
                    break;
            }
            // 竞争初始化cells失败的去尝试在base上做递增，如果成功就直接返回
            else if (casBase(v = base, ((fn == null) ? v + x :
                                        fn.applyAsLong(v, x))))
                // 直接在base上进行累积操作成功了，任务完成，可以退出循环了
                break;                          // Fall back on using base
        }
    }

    // 逻辑与longAccumulate相同
    final void doubleAccumulate(double x, DoubleBinaryOperator fn,
                                boolean wasUncontended) {}
       
    // Unsafe mechanics Unsafe机制初始化
    private static final sun.misc.Unsafe UNSAFE;
    private static final long BASE;
    private static final long CELLSBUSY;
    private static final long PROBE;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> sk = Striped64.class;
            BASE = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("base"));
            CELLSBUSY = UNSAFE.objectFieldOffset
                (sk.getDeclaredField("cellsBusy"));
            Class<?> tk = Thread.class;
            PROBE = UNSAFE.objectFieldOffset
                (tk.getDeclaredField("threadLocalRandomProbe"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }

}
```

### 二、LongAdder

```java
public class LongAdder extends Striped64 implements Serializable {
    private static final long serialVersionUID = 7249069246863182397L;

    // 构造器，直接使用默认值，base = 0, cells = null，sum = 0
    public LongAdder() {
    }

    // add方法，根据父类的longAccumulate方法的要求，这里要进行一次CAS操作
    // （虽然这里有两个CAS，但是第一个CAS执行了就不会执行第二个，要执行第二个，第一个就被“短路”了不会被执行）
    // 在线程竞争不激烈时，这样做更快
    public void add(long x) {
        Cell[] as; long b, v; int m; Cell a;
        // cells非null或者在base上cas递增失败
        if ((as = cells) != null || !casBase(b = base, b + x)) {
            boolean uncontended = true; // 是否没有竞争
            // cells为null或cells数组没有元素或根据线程hash得到数组元素为null或用CAS递增得到的cell失败
            if (as == null || (m = as.length - 1) < 0 ||
                // getProbe()获取当前线程的hash，与m求与运算，能获得一个最小0最大m的数，即得到cells数组中元素的索引
                (a = as[getProbe() & m]) == null ||
                // 根据得到的索引，取出Cell元素，进行CAS递增
                !(uncontended = a.cas(v = a.value, v + x)))
                // 如果递增失败，即出现竞争，调用longAccumulate方法
                longAccumulate(x, null, uncontended);
        }
    }

    // 自增1
    public void increment() {
        add(1L);
    }

    // 自减1
    public void decrement() {
        add(-1L);
    }

    // 返回累加的和，也就是“当前时刻”的计数值，此返回值可能不是绝对准确的，因为调用这个方法时还有其他线程可能正在进行累加。
    public long sum() {
        Cell[] as = cells; Cell a;
        long sum = base;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }

    // 重置计数器，只应该在明确没有并发的情况下调用，可以用来避免重新new一个LongAdder
    public void reset() {
        Cell[] as = cells; Cell a;
        base = 0L;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    a.value = 0L;
            }
        }
    }

    // 相当于sum()后再调用reset()
    public long sumThenReset() {
        Cell[] as = cells; Cell a;
        long sum = base;
        base = 0L;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null) {
                    sum += a.value;
                    a.value = 0L;
                }
            }
        }
        return sum;
    }
	    
   	// 序列化代理
    private static class SerializationProxy implements Serializable {
        private static final long serialVersionUID = 7249069246863182397L;

        // LoadAdder的和
        private final long value;

        SerializationProxy(LongAdder a) {
            value = a.sum();
        }

        // 反序列化的时候生成的对象替换为LongAdder
        private Object readResolve() {
            LongAdder a = new LongAdder();
            a.base = value;
            return a;
        }
    }

    // LongAdder对象序列化时的对象替换为new SerializationProxy(this)
    private Object writeReplace() {
        return new SerializationProxy(this);
    }
}
```

### 参考文献

- [jdk1.8 LongAdder源码学习](https://blog.csdn.net/u011392897/article/details/60480108)
- [JDK1.8的LongAdder分析](http://footmanff.com/2018/03/21/2018-03-21-LongAdder-1/)
- [Java并发学习(十一)-LongAdder和LongAccumulator探究](https://blog.csdn.net/anLA_/article/details/78680080)