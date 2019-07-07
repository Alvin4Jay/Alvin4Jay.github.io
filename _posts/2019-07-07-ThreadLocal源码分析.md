---
layout:     post
title:      ThreadLocal源码分析
subtitle:   ThreadLocal
date:       2019-07-07
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
---

# ThreadLocal源码分析

ThreadLocal类用来提供线程内部的局部变量。这种变量在多线程环境下访问(通过get或set方法访问)时能保证各个线程里的变量相对独立于其他线程内的变量。ThreadLocal实例通常来说都是`static`类型的，用于关联线程和线程的上下文。可以总结为一句话：**ThreadLocal的作用是提供线程内的局部变量，这种变量在线程的生命周期内起作用，减少同一个线程内多个函数或者组件之间一些公共变量的传递的复杂度。**

### 一、ThreadLocal的属性

```java
private final int threadLocalHashCode = nextHashCode(); // 实例化ThreadLocal时设置该值

private static AtomicInteger nextHashCode = new AtomicInteger(); 

private static final int HASH_INCREMENT = 0x61c88647; // hash增量

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT); // CAS递增
}
```

ThreadLocal通过 `threadLocalHashCode` 来标识每一个ThreadLocal实例的唯一性。`threadLocalHashCode`通过CAS操作进行更新，每次计算threadLocalHashCode时的增量为 **0x61c88647**(减少ThreadLocalMap的hash冲突见[深入理解 ThreadLocal (这些细节不应忽略)](https://www.jianshu.com/p/56f64e3c1b6c)对该魔数的说明与测试)。

### 二、ThreadLocal的基本操作

##### 1.构造函数

```java
// Creates a thread local variable.
public ThreadLocal() { // 方法内部什么都没做
}
```

##### 2.initialValue方法

```java
protected T initialValue() { // 设置线程本地变量的初始值
    return null;
}
```

initialValue方法用于设置当前线程相对于ThreadLocal实例(Key)的初始值(Value)。该方法在调用get方法的时候会第一次调用，但是如果一开始就调用了set方法，则该方法不会被调用。通常该方法只会被调用一次，除非手动调用了remove方法之后又调用get方法，这种情况下，get()方法中还是会调用initialValue方法。该方法是protected类型的，很显然是建议在子类重载该函数的，所以通常该方法都会以匿名内部类的形式被重载，以指定初始值，如：

```java
private static final ThreadLocal<Integer> value = new ThreadLocal<Integer>() {
        @Override
        protected Integer initialValue() {
            return 0;
        }
    };
```

此外，可直接使用ThreadLocal提供的withInitial方法，提供一个Supplier<?>：

```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier); 
}

static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() { // 重写initialValue()方法，提供初始值
        return supplier.get();
    }
}
```

因此上面的例子可改造为

```java
private static final ThreadLocal<Integer> value = ThreadLocal.withInitial(() -> 0);
```

##### 3.get方法

该方法用来获取与当前线程关联的ThreadLocal的值。如果当前线程没有该ThreadLocal的值，则调用initialValue方法获取初始值返回。

```java
public T get() {
    Thread t = Thread.currentThread(); // 当前线程
    ThreadLocalMap map = getMap(t); // 获取当前线程的ThreadLocalMap
    if (map != null) { // map已创建
	      // 调用ThreadLocalMap.getEntry()方法获取Entry
        ThreadLocalMap.Entry e = map.getEntry(this); 
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result; // entry不为空，返回value
        }
    }
    return setInitialValue();// 否则调用setInitialValue获取初始值，并设置到线程的ThreadLocalMap中
}
// 获取并设置初始值
private T setInitialValue() {
    T value = initialValue(); // 获取初始值
    Thread t = Thread.currentThread(); 
    ThreadLocalMap map = getMap(t); // 获取当前线程的ThreadLocalMap
    if (map != null) // map已创建
        map.set(this, value); // 调用ThreadLocalMap.set()方法插入key value对
    else
      	// 如果map不存在，创建线程的ThreadLocalMap，并直接以当前ThreadLocal为键，value为值，
      	// 插入到该map中
        createMap(t, value); 
    return value;
}
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals; // ThreadLocalMap threadLocals是Thread的属性
}
// 创建线程的ThreadLocalMap
void createMap(Thread t, T firstValue) {
	  // <ThreadLocal, value>是map的初始entry
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

从getMap方法可以看出，每个Thread里面都有一个ThreadLocal.ThreadLocalMap成员变量，也就是说每个线程通过ThreadLocal.ThreadLocalMap与ThreadLocal相绑定，这样可以确保每个线程访问到的thread-local variable都是本线程的。

##### 4.set方法

set方法用来设置当前线程的该ThreadLocal的值。

```java
public void set(T value) { // 设置当前线程的ThreadLocal的值为value
    Thread t = Thread.currentThread(); // 当前线程
    ThreadLocalMap map = getMap(t); // 获取当前线程的ThreadLocalMap
    if (map != null) // map已创建
        map.set(this, value); // 直接将当前value设置到ThreadLocalMap中
    else
        createMap(t, value); // 说明当前线程是第一次使用线程本地变量，构造map
}
```

- getMap方法根据当前线程得到当前线程的ThreadLocalMap对象
- 如果map不为空，说明当前线程已经构造过ThreadLocalMap，直接将值存储到map中
- 如果map为空，说明是第一次使用，调用createMap构造

##### 5.remove方法

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this); // 调用ThreadLocalMap的remove()方法删除Entry<ThreadLocal, value>
}
```

remove方法用来将当前线程的ThreadLocal绑定的值删除。在某些情况下需要手动调用该函数，防止内存泄露。

### 三、ThreadLocalMap

ThreadLocalMap是 ThreadLocal的静态嵌套类。ThreadLocalMap有一个常量和三个成员变量：

```java
private static final int INITIAL_CAPACITY = 16; // table数组初始化容量

private Entry[] table; // table数组保存Entry，长度为2的幂次

private int size = 0; // 现有Entry个数

private int threshold; // 扩容rehash阈值
```

Entry类是ThreadLocalMap的静态嵌套类，用于存储数据。Entry类继承了WeakReference<ThreadLocal<?>>，即每个Entry对象都有一个ThreadLocal的弱引用(作为key)。因此，只要ThreaLocal不存在强引用，Entry对ThreadLocal的弱引用不会阻止GC对它的回收。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k); // key ThreadLocal为弱引用
        value = v;
    }
}
```

##### 1.构造函数

ThreadLocalMap类有两个构造函数，其中常用的是ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue)：

```java
// 用于ThreadLocal.createMap()方法
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY]; // 初始化数组
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1); // 获取数组索引
    table[i] = new Entry(firstKey, firstValue); // 新建Entry
    size = 1;
    setThreshold(INITIAL_CAPACITY); // 设置resize阈值
}
```

构造函数的第一个参数就是本ThreadLocal实例(`this`)，第二个参数就是要保存的线程本地变量。构造函数首先创建一个长度为16的Entry数组，然后计算出firstKey对应的哈希值，然后存储到table中，并设置size和threshold。

注意一个细节，计算hash的时候里面采用了`hashCode & (size - 1)`的算法，这相当于取模运算`hashCode % size`的一个更高效的实现（和HashMap中的思路相同）。正是因为这种算法，要求数组容量必须是 **2的指数**，因为这可以使得hash发生冲突的次数减小。

##### 2.getEntry方法

getEntry方法主要用于ThreadLocal.get()中，获取ThreadLocal key对应的Entry。

```java
// 获取ThreadLocal key对应的Entry
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1); // 根据hash计算数组索引
    Entry e = table[i];
    if (e != null && e.get() == key) // entry存在且key对应，则返回
        return e;
    else
        return getEntryAfterMiss(key, i, e); // 直接命中失败后，调用getEntryAfterMiss遍历获取
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) { // 遍历获取
        ThreadLocal<?> k = e.get();
        if (k == key) // key对应，则直接返回
            return e;
      	// e的key为null，则其ThreadLocal key已被GC回收。e为遗留entry，则从i开始进行一次遗留entry
      	// 的清理工作
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len); // 继续遍历
        e = tab[i];
    }
    return null; // 没有key对应的entry，返回null
}
```

###### 2.1 expungeStaleEntry方法

清除从staleSlot开始，到下一个entry为null的槽之间的遗留节点(key==null)。

```java
private int expungeStaleEntry(int staleSlot) { // 从staleSlot开始，清除遗留entry
    Entry[] tab = table;
    int len = tab.length;

    // expunge entry at staleSlot
    tab[staleSlot].value = null; // 将staleSlot位置的Entry清除
    tab[staleSlot] = null;
    size--;

    // Rehash until we encounter null 清除staleSlot到下一个entry为null的槽之间的遗留entry，
  	// 并rehash
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get(); // key
        if (k == null) { // 发现遗留节点，删除
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1); // 计算索引
            if (h != i) { // 该entry不在计算出来的索引位置上
                tab[i] = null; // rehash，调整位置

                while (tab[h] != null) // rehash过程中若发现欲放入的位置不为null，则顺延
                    h = nextIndex(h, len); 
                tab[h] = e;
            }
        }
    }
    return i; // 返回entry为null的槽位置
}
// 下一个索引位置，模len
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0);
}
```

##### 3.set方法

set方法用于ThreadLocal.setInitialValue与ThreadLocal.set方法中，保存或者更新键为key、值为value的Entry。

```java
private void set(ThreadLocal<?> key, Object value) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1); // 计算数组索引

    for (Entry e = tab[i]; // 遍历
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        if (k == key) { // key匹配，则更新value并返回
            e.value = value;
            return;
        }
				// k为null，则对应entry为遗留entry，则该entry替换为键为key、值为value的新entry，并清理遗留节点
        if (k == null) { 
            replaceStaleEntry(key, value, i);
            return;
        }
    }
		// 在遍历entry时，key没命中且无遗留entry，则在null位置插入新entry
    tab[i] = new Entry(key, value);
    int sz = ++size;
    if (!cleanSomeSlots(i, sz) && sz >= threshold) // 清理遗留entry，并判断是否扩容
        rehash(); // 若无遗留entry被清除，且size到达了阈值threshold，则rehash
}
```

从上面也可以看出，如果hash冲突，会通过nextIndex方法再次计算位置。可见ThreadLocalMap解决冲突的方法是 **线性探测法**（不断加 1），而不是 HashMap 的 **链地址法**。

###### 3.1 replaceStaleEntry方法

replaceStaleEntry方法用于替换遗留entry为新entry，并执行清除两个entry为null的槽之间的遗留节点的操作。

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value, int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

  	// 回退查找key为null的遗留entry，查找最前一个无效的slot，直到遇到entry为null的槽
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;

  	// 从i开始往后一直遍历到数组最后一个Entry（线性探索）
    for (int i = nextIndex(staleSlot, len);
         (e = tab[i]) != null;
         i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        if (k == key) { // key命中
            e.value = value; // 更新value

            tab[i] = tab[staleSlot]; // 将此位置的e交换到staleSlot位置，此时i位置为无效的entry
            tab[staleSlot] = e;

            // Start expunge at preceding stale entry if it exists
          	// 如果之前回退查找没有找到比staleSlot位置更前的遗留entry，则更新slotToExpunge为i位置，
          	// 作为清理的起点
            if (slotToExpunge == staleSlot) 
                slotToExpunge = i;
          	// 从slotToExpunge开始做一次连续的清理
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

      	// 如果当前的slot已经无效，并且向前扫描过程中没有无效slot，则更新slotToExpunge为当前位置
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

	  // 如果key对应的entry不存在，则直接在staleSlot位置放一个新的entry
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

	  // 如果有任何一个无效的slot，则做一次清理
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
```

###### 3.2 cleanSomeSlots方法

这个方法有两处地方会被调用，用于清理无效的Entry。

- 插入的时候可能会被调用(ThreadLocalMap.set())
- 替换无效slot的时候可能会被调用(ThreadLocal.replaceStaleEntry方法)

区别是前者传入的n为元素个数(size)，后者为table的容量(len)。

```java
// 查找遗留的无效entry并删除，i位置肯定不是一个无效entry，从i位置之后开始扫描；n扫描次数控制
private boolean cleanSomeSlots(int i, int n) { 
    boolean removed = false; // 是否遗留的无效节点被删除了
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len); // i在任何情况下自己都不会是一个无效slot，所以从下一个开始判断
        Entry e = tab[i];
        if (e != null && e.get() == null) { // 发现无效节点
            n = len; // 扩大扫描控制因子
            removed = true;
            i = expungeStaleEntry(i); // 清理一个连续段
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

###### 3.3 rehash

rehash主要用于ThreadLocalMap.set方法中，在size到达了阈值threshold时，进行扩容操作。

```java
private void rehash() {
    expungeStaleEntries(); // 进行一次全面的清理无效entry的操作

    // Use lower threshold for doubling to avoid hysteresis
    if (size >= threshold - threshold / 4) // 在size到达了0.75*threshold，即0.5*数组容量时resize
        resize();
}

// 扩容为原来容量的两倍，并rehash
private void resize() {
    Entry[] oldTab = table;
    int oldLen = oldTab.length;
    int newLen = oldLen * 2; // 2倍
    Entry[] newTab = new Entry[newLen];
    int count = 0;

    for (int j = 0; j < oldLen; ++j) {
        Entry e = oldTab[j];
        if (e != null) {
            ThreadLocal<?> k = e.get();
            if (k == null) {
                e.value = null; // Help the GC
            } else {
                int h = k.threadLocalHashCode & (newLen - 1); // 计算索引
                while (newTab[h] != null)
                    h = nextIndex(h, newLen);
                newTab[h] = e; // 插入
                count++; // 个数+1
            }
        }
    }

    setThreshold(newLen); // 设置阈值为长度的2/3
    size = count; // 设置大小
    table = newTab;
}

// 清除所有无效的遗留entry
private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null) // 发现无效的entry
            expungeStaleEntry(j); // 执行一次连续的清理动作
    }
}
```

rehash 操作会执行一次全表的扫描清理工作，并在 size 大于等于 threshold 的四分之三时进行 resize。但注意threshold在setThreshold 的时候又取了三分之二：

```java
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

##### 3.remove方法

清除key对应的entry。

```java
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1); // 计算索引
    for (Entry e = tab[i]; // 遍历查找
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) { // 找到
            e.clear();
            expungeStaleEntry(i); // 从i开始执行一次清除无效entry的操作
            return;
        }
    }
}
```

### 四、ThreadLocal与内存泄漏

Thread、ThreadLocalMap、ThreadLocal总览图如下，引用关系：**Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value**

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/ThreadLocal-4.png?x-oss-process=style/markdown-pic)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/ThreadLocal-1.png?x-oss-process=style/markdown-pic)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/ThreadLocal-2.png)

Thread类有属性变量threadLocals （类型是ThreadLocal.ThreadLocalMap），也就是说每个线程有一个自己的ThreadLocalMap ，所以每个线程往这个ThreadLocal中读写隔离的，并且是互相不会影响的。**一个ThreadLocal只能存储一个Object对象，如果需要存储多个Object对象那么就需要多个ThreadLocal**。

如图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/ThreadLocal-3.png?x-oss-process=style/markdown-pic)

此外，Entry对ThreadLocal的引用为弱引用，用虚线表示。

##### 1.弱引用ThreadLocal

Entry对象弱引用ThreadLocal，是JDK为防止内存泄漏所做的工作。在这种情况下，如果只有ThreadLocalMap中的Entry的key指向ThreadLocal的时候，ThreadLocal会进行回收，即没有ThreadLocal的强引用。

ThreadLocal被垃圾回收后，在ThreadLocalMap里对应的Entry的键会变成null，但是Entry是强引用，那么Entry里面存储的Object，并没有办法进行回收，所以ThreadLocalMap又做了一些额外的回收工作。从上面对ThreadLocalMap的源码分析也可以看出，在set、getEntry、remove等方法调用的时候会进行无效entry的清理工作，如expungeStaleEntry、cleanSomeSlots等。

很多时候，使用的都是线程池的线程，程序不停止，线程基本不会销毁。由于线程的生命周期很长，如果往ThreadLocal里面set了很大的Object对象，且ThreadLocal被GC回收的情况下，前面所说的这些清理工作是在主动调用get、remove等操作的前提下，此时就需要线程在其他的ThreadLocal实例上操作才有可能清理无效Entry，这当然是不可能任何情况都成立的。所以**很多情况下需要使用者手动调用ThreadLocal的remove方法，手动删除不再需要的Entry，防止内存泄露**。

##### 2.ThreadLocal为static变量

阿里巴巴Java开发手册建议将ThreadLocal作为static变量，这有好的一面，也有不好的一面。ThreadLocal为static变量，优点是防止了ThreadLocal无意义的多实例创建情况；此外，当static时，ThreadLocal ref生命周期延长——ThreadLocalMap的key在线程生命期内始终有值——ThreadLocalMap的value在线程生命期内不释放，相对于不使用static变量，更加容易出现内存泄漏的情况，故线程池下，static修饰TheadLocal引用，必须(1)显式remove 或(2)手动设置ThreadLocal ref＝null。

因此，使用ThreadLocal的最佳实践是

```java
try {
    // 其它业务逻辑
} finally {
    threadLocal对象.remove();
}
```

### 五、总结

##### 1. ThreadLocalMap中无用的Entry被清理的情况

- Thread结束的时候；
- 插入元素时，发现stale entry，则会进行替换并清理；
- 插入元素时，ThreadLocalMap的 size 达到 threshold，并且没有任何stale entry的时候，会调用rehash方法清理并扩容；
- 调用ThreadLocalMap的remove方法或set(null)时。

##### 2.ThreadLocal的使用场景

- 每个线程需要有自己单独的实例；
- 实例需要在多个方法中共享，但不希望被多线程共享。

### 参考

- [Java进阶（七）正确理解Thread Local的原理与适用场景](http://www.jasongj.com/java/threadlocal/)
- [ThreadLocal和synchronized的区别?](https://www.zhihu.com/question/23089780/answer/62097840)
- [ThreadLocal的原理，源码深度分析及使用](https://www.cnblogs.com/barrywxx/p/10141169.html)
- [并发编程 ThreadLocal 源码深入分析](https://www.sczyh30.com/posts/Java/java-concurrent-threadlocal/)
- [手撕面试题ThreadLocal！！！](https://mp.weixin.qq.com/s?__biz=MzU2NjIzNDk5NQ==&mid=2247486666&idx=1&sn=ee9d72b115411940940f00171986e0db&chksm=fcaed6d6cbd95fc08cf5525e7974efc7753f9f018f2a79fd5386b9f0132b90645394e85c3568&mpshare=1&scene=1&srcid=#rd)
- [深入理解 ThreadLocal (这些细节不应忽略)](https://www.jianshu.com/p/56f64e3c1b6c)
- [Java面试必问，深入理解 ThreadLocal 实现原理与内存泄露](https://mp.weixin.qq.com/s?__biz=MzA5NDg3MjAwMQ==&mid=2457103267&idx=1&sn=a18ca7db8af546684b8d4465917856d0&chksm=87c8c30db0bf4a1be1ea0ca40a63327d2294cebdef55def3c791ce653562f061f6bc297a1f68&mpshare=1&scene=1&srcid=#rd)
- [ThreadLocal为什么要设计成private static](https://blog.csdn.net/silyvin/article/details/79551635)
- [JDK ThreadLocal API](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html)