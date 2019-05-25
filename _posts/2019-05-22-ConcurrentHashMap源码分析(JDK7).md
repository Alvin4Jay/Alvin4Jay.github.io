---
layout:     post
title:      ConcurrentHashMap源码分析(JDK7)
subtitle:   ConcurrentHashMap/HashMap/HashTable
date:       2019-05-22
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JUC
    - JCF
---

# ConcurrentHashMap源码分析(JDK7)

ConcurrentHashMap和HashMap一样，都是基于散列的容器，ConcurrentHashMap可以认为是一种线程安全的HashMap，它使用了一种完全不同的加锁策略来提高并发性和伸缩性(不同于HashTable的粗粒度锁)。ConcurrentHashMap并不是将每个方法在同一个锁上同步并使得每次只能有一个线程访问容器，而是使用一种粒度更细的加锁机制来实现更大程度的共享，这种机制称为“**分段锁**”。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-01.png?x-oss-process=style/markdown-pic)

ConcurrentHashMap继承自AbstractMap，实现了ConcurrentMap接口，内部主要的实体类有两个：

- Segment（桶，段）
- HashEntry（节点）

对应下面的图可以看出三者之间的关系，静态内部类HashEntry用来封装映射表的键值对，Segment充当锁的角色，每个 Segment 对象维护散列表的若干个桶。每个桶是由若干个 HashEntry 对象链接起来的链表。一个 ConcurrentHashMap 实例中包含由若干个 Segment 对象组成的数组。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-02.png?x-oss-process=style/markdown-pic)

### 一、主要的静态嵌套类

##### 1.Segment

```java
// 继承自ReentrantLock，Segment是一种类似于HashTable的特殊hash表
static final class Segment<K,V> extends ReentrantLock implements Serializable {

    private static final long serialVersionUID = 2249069246763182397L;

  	// 尝试获取锁失败后，最大的重试获取锁次数，重试次数过后若还没获取到锁，阻塞直到获取到锁
    static final int MAX_SCAN_RETRIES =
        Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

  	// 存储HashEntry的数组
    transient volatile HashEntry<K,V>[] table;

  	// 本segment中HashEntry的个数
    transient int count;

  	// 本segment中更新操作的次数
    transient int modCount;

  	// 阈值，当table中的HashEntry个数超过该值后，需要table扩容rehash
    transient int threshold;

  	// 负载因子
    final float loadFactor;

    Segment(float lf, int threshold, HashEntry<K,V>[] tab) {
        this.loadFactor = lf;
        this.threshold = threshold;
        this.table = tab;
    }
 		// ...   
}
```

segment结构示意图：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-03.jpg)

Segment是一个可重入锁，每个Segment相当于一个HashMap，是一个数组和链表结构，维持了一个HashEntry数组。

##### 2.HashEntry

```java
// HashEntry封装单个键值对
static final class HashEntry<K,V> {
    final int hash;
    final K key;
    volatile V value;
    volatile HashEntry<K,V> next;

    HashEntry(int hash, K key, V value, HashEntry<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

  	// 调用putOrderedObject进行写操作，因为setNext是在获得锁的情况下调用的，因此能保证写的可见性，
  	// 具有volatile写的语义。
    final void setNext(HashEntry<K,V> n) {
        UNSAFE.putOrderedObject(this, nextOffset, n);
    }

    // Unsafe 机制
    static final sun.misc.Unsafe UNSAFE;
    static final long nextOffset; // next变量内存偏移量
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class k = HashEntry.class;
            nextOffset = UNSAFE.objectFieldOffset
                (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

### 二、属性

```java
// 定位segment时用的掩码，如int j = (hash >>> segmentShift) & segmentMask; 
// j表示segment[]数组中的第j个segment
final int segmentMask;

// 定位segment时用的hash位移动量，如int j = (hash >>> segmentShift) & segmentMask; 
// j表示segment[]数组中的第j个segment
final int segmentShift;

// segment数组
final Segment<K,V>[] segments;

transient Set<K> keySet;
transient Set<Map.Entry<K,V>> entrySet;
transient Collection<V> values;
```

### 三、构造器

```java
// initialCapacity初始容量，loadFactor负载因子，concurrencyLevel并发线程数的评估
public ConcurrentHashMap(int initialCapacity,
                         float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (concurrencyLevel > MAX_SEGMENTS)
        concurrencyLevel = MAX_SEGMENTS;
    // Find power-of-two sizes best matching arguments
    int sshift = 0; // sshift表示计算ssize时，ssize左移的次数，默认为4
    int ssize = 1; // segment数组的大小ssize为2的幂次，默认为16
    while (ssize < concurrencyLevel) {
        ++sshift;
        ssize <<= 1;
    }
    this.segmentShift = 32 - sshift; // 计算定位segment时用的hash位移动量
    this.segmentMask = ssize - 1; // 计算定位segment时用的掩码
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    int c = initialCapacity / ssize; // 计算每个segment的容量
    if (c * ssize < initialCapacity)
        ++c;
    int cap = MIN_SEGMENT_TABLE_CAPACITY;
    while (cap < c) // 每个segment的容量为2的幂次
        cap <<= 1;
    // 创建segments、segments[0]
    Segment<K,V> s0 =
        new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                         (HashEntry<K,V>[])new HashEntry[cap]);
    Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
  	// 调用UNSAFE.putOrderedObject将s0写入ss数组，由于segments变量是final变量，
  	// 因此能保证构造器中对该变量的初始化优先于其他地方对该变量的引用，保证可见性。
    UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
    this.segments = ss;
}
// 指定初始化容量和负载因子
public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, DEFAULT_CONCURRENCY_LEVEL);
}
// 指定初始化容量
public ConcurrentHashMap(int initialCapacity) {
    this(initialCapacity, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}
// 使用默认值
public ConcurrentHashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
}
// 指定map
public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this(Math.max((int) (m.size() / DEFAULT_LOAD_FACTOR) + 1,
                  DEFAULT_INITIAL_CAPACITY),
         DEFAULT_LOAD_FACTOR, DEFAULT_CONCURRENCY_LEVEL);
    putAll(m);
}
```

### 四、主要方法分析

##### 1.put(K key, V value)

**ConcurrentHashMap中的put操作**：

```java
public V put(K key, V value) { // key value不允许为null
    Segment<K,V> s;
    if (value == null)
        throw new NullPointerException();
    int hash = hash(key); // 计算hash值
	  // 确定segment的索引，使用hash的高位与segmentMask进行与操作
    int j = (hash >>> segmentShift) & segmentMask; 
  	// 此处使用UNSAFE.getObject获取segment[j]元素，非volatile。因为这里要做null判断，并且
  	// segment[j]一旦初始化就不会再变为null，所以这里使用getObject与getObjectVolatile效果
  	// 是一样的，因此这里使用getObject性能更好。
    if ((s = (Segment<K,V>)UNSAFE.getObject          // 非volatile读，在ensureSegment重新check
         (segments, (j << SSHIFT) + SBASE)) == null) 
	      // s如果为null，则需要重新check，确保s不为空(第一次访问segment时，创建segment对象)
        s = ensureSegment(j); 
    return s.put(key, hash, value, false);
}
// 计算k的hash值
private int hash(Object k) {
   int h = hashSeed;

   if ((0 != h) && (k instanceof String)) {
       return sun.misc.Hashing.stringHash32((String) k);
   }

   h ^= k.hashCode();

   // Spread bits to regularize both segment and index locations,
   // using variant of single-word Wang/Jenkins hash.
   h += (h <<  15) ^ 0xffffcd7d;
   h ^= (h >>> 10);
   h += (h <<   3);
   h ^= (h >>>  6);
   h += (h <<   2) + (h << 14);
   return h ^ (h >>> 16);
}
// 确保索引k处的segment存在，如果不存在，则创建并返回
private Segment<K,V> ensureSegment(int k) {
    final Segment<K,V>[] ss = this.segments;
    long u = (k << SSHIFT) + SBASE; // raw offset 内存偏移量
    Segment<K,V> seg;
    if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) { // 判断是否为null
        Segment<K,V> proto = ss[0]; // use segment 0 as prototype 以segment[0]为原型
        int cap = proto.table.length; // 容量
        float lf = proto.loadFactor; // 负载因子
        int threshold = (int)(cap * lf); // 阈值
        HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap]; // table数组
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
            == null) { // recheck 重新检查是否为null
            Segment<K,V> s = new Segment<K,V>(lf, threshold, tab); // 创建segment
            while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))
                   == null) { // CAS 写入
                if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))
                    break;
            }
        }
    }
    return seg;
}
```

**Segment中的put操作(加锁)**：

```java
// Segment.put()
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
	  // 尝试获取锁，若获取不到，则调用scanAndLockForPut获取锁
    HashEntry<K,V> node = tryLock() ? null : 
        scanAndLockForPut(key, hash, value);
  	// 到这里，说明已经获取到互斥锁
    V oldValue; // 旧值
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash; // 与运算，计算HashEntry桶位置
        HashEntry<K,V> first = entryAt(tab, index); // index对应桶位置的第一个节点
        for (HashEntry<K,V> e = first;;) { // 查找节点
            if (e != null) {
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) { // 找到对应key的节点
                    oldValue = e.value;
                    if (!onlyIfAbsent) { // onlyIfAbsent若为true，则若原映射存在，则值不替换
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;
            }
            else { // 节点未找到
                if (node != null) // 调用scanAndLockForPut时已创建了待插入的节点
                    node.setNext(first); // node的next指针指向first节点
                else // 否则创建一个新节点，next指针指向first节点
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1; // segment中节点元素总数+1
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node); // 若总数已超过阈值，则先扩容，扩容之后再将新节点插入
                else
                    setEntryAt(tab, index, node); // 如果不需要扩容，则直接使用头插法插入新节点
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue; // 返回旧值，可能为null
}

// 根据hash确定seg中HashEntry桶的位置
static final <K,V> HashEntry<K,V> entryForHash(Segment<K,V> seg, int h) {
    HashEntry<K,V>[] tab;
    return (seg == null || (tab = seg.table) == null) ? null :
        (HashEntry<K,V>) UNSAFE.getObjectVolatile
        (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
}
// 根据索引i获取数组tab的第i个元素(HashEntry)
static final <K,V> HashEntry<K,V> entryAt(HashEntry<K,V>[] tab, int i) {
    return (tab == null) ? null :
        (HashEntry<K,V>) UNSAFE.getObjectVolatile
        (tab, ((long)i << TSHIFT) + TBASE);
}
// 设置索引i处的节点为e
static final <K,V> void setEntryAt(HashEntry<K,V>[] tab, int i,
                                   HashEntry<K,V> e) {
  	// 因为setEntryAt操作是在获得互斥锁的情况下进行的，所以调用UNSAFE.putOrderedObject
  	// 即可保证volatile写语义
    UNSAFE.putOrderedObject(tab, ((long)i << TSHIFT) + TBASE, e);
}
```

put操作时需要注意的点：

- 插入新节点的时候，是头插法。
- 如果count > threshold，那么当前segment需要扩容。
- 若节点已存在，不一定更新，只有当onlyIfAbsent=false的时候才更新。

在put方法中，首先进行尝试加锁操作，tryLock失败后并不会按照常规思路阻塞当前线程，而是执行scanAndLockForPut方法，下面重点分析下这个方法做了什么工作。

```java
// put操作时，未获取到锁，则重试尝试获取锁，直到获取锁成功，或者重试次数到达上限后，阻塞获取锁直到成功。
// 这里如果未定位到节点，会创建一个待插入的节点返回
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);  // 根据hash得到对应桶位置的第一个HashEntry
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null; // 如果key未找到，则创建一个node，等到获取锁之后插入到table
    int retries = -1; // 当前重试获取锁的次数，在定位节点时为-1
    while (!tryLock()) { // 尝试获取锁，如果未成功，进入下面的逻辑
        HashEntry<K,V> f; // f变量在下面重新检查first节点时使用
        if (retries < 0) { // 还未定位到节点
            if (e == null) { // 如果未找到节点
                if (node == null) // 创建一个新节点，并将retries置为0
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key)) // 定位到节点，将retries置为0
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) { // 尝试获取锁次数加1，判断是否超过最大重试次数
            lock(); // 如果超过了，则阻塞直到获取锁成功
            break;
        }
        else if ((retries & 1) == 0 && // 每隔一次循环(retries为偶数)，检查first节点是否发生变化
                 (f = entryForHash(this, hash)) != first) {
            e = first = f; // 如果变化，则需要重新遍历
            retries = -1;
        }
    }
    return node;
}
```

scanAndLockForPut的作用：

- 当前线程获取不到锁的时候并没有闲着，而是查找key是否已经存在，如果当前链表中没有查到，则新建一个HashEntry对象。
- 新建HashEntry节点后，当retries <= MAX_SCAN_RETRIES时，不断通过tryLock尝试获取锁，retries >  MAX_SCAN_RETRIES，则调用lock()，此时若还获取不到锁，那么当前线程就被阻塞。
- 在检索key的时候，别的线程可能正在对segment进行修改，所以要做如下检查：

```java
else if ((retries & 1) == 0 && // 每隔一次循环(retries为偶数)，检查first节点是否发生变化
         (f = entryForHash(this, hash)) != first) {
    e = first = f; // 如果变化，则需要重新遍历
    retries = -1;
}
```

> 通过scanAndLockForPut方法，当前线程就可以在即使获取不到segment锁的情况下，完成潜在需要添加节点的实例化工作，当获取锁后，就可以直接将该节点插入链表即可。还实现了类似于自旋锁的功能，防止执行put操作的线程频繁阻塞，这些优化都提升了put操作的性能。

**Segment的rehash操作**：

```java
// table大小变为2倍，将原table中的各个节点移动到新table，并添加待插入节点node到新table
// segment的rehash是在获取到互斥锁的前提下进行的
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table; // old table
    int oldCapacity = oldTable.length; // 老的容量
    int newCapacity = oldCapacity << 1; // 新的容量，2倍老的容量
    threshold = (int)(newCapacity * loadFactor); // 更新阈值
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1; // 桶位置索引的掩码
    for (int i = 0; i < oldCapacity ; i++) { // 遍历old table，转移节点数据
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next; // 下一个节点
            int idx = e.hash & sizeMask; // 桶位置索引
            if (next == null)   //  Single node on list 链表上只有一个节点
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
              	// 由于table大小是2的幂次，且是2倍的扩容，所以原链表上的节点在扩容之后，要么在原
              	// 位置，要么在原位置+原table大小的位置。在一整条链表上，lastRun、lastIdx表示
              	// 某个节点，该节点之后的剩余节点，都会rehash到新table的同一个位置。这样最大程
              	// 度的复用了原table链表上的节点。
                HashEntry<K,V> lastRun = e; // 开始查找lastRun、lastIdx
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask; // 新table中的索引
                    if (k != lastIdx) { // 不相等，则更新lastIdx、lastRun
                        lastIdx = k;
                        lastRun = last;
                    }
                }
                newTable[lastIdx] = lastRun; // 直接复用原链表lastRun及其之后的节点
                // Clone remaining nodes 将e到lastRun之间的节点rehash到新table
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    V v = p.value;
                    int h = p.hash;
                    int k = h & sizeMask; // 计算index
                    HashEntry<K,V> n = newTable[k]; // 头插法插入节点
                    newTable[k] = new HashEntry<K,V>(h, p.key, v, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node 计算新添加node的索引
    node.setNext(newTable[nodeIndex]); // 头插法插入
    newTable[nodeIndex] = node;
    table = newTable;
}
```

##### 2.get(Object key)

```java
// 根据key，获取value(不加锁)
public V get(Object key) {
    Segment<K,V> s;
    HashEntry<K,V>[] tab;
    int h = hash(key); // 计算hash
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE; // 计算segment内存偏移
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
      	// (tab.length - 1) & h 获取table中桶的位置
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {
            K k;
          	// 找到对应key的节点，返回value
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```

①首先通过key的hash值，确定所属的segment的索引`(h >>> segmentShift) & segmentMask)`，以及该索引处segment相对于segment数组的内存偏移量u。

默认情况下segmentShift=28，segmentMask=15，h是hash后的值，为32位，h >>> segmentShift) & segmentMask)即让hash值的高4位参与到运算。

②通过UNSAFE类的getObjectVolatile方法获取segment

```java
s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u))
```

③通过hash值确定HashEntry桶的位置，遍历链表找出entry

```java
for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
         (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
     e != null; e = e.next) {
    K k;
    if ((k = e.key) == key || (e.hash == h && key.equals(k)))
        return e.value;
}
```

##### 3.remove(Object key) & remove(Object key, Object value)

```java
public V remove(Object key) { // 根据key，移除相应节点
    int hash = hash(key); // 计算hash
    Segment<K,V> s = segmentForHash(hash); // 得到对应的segment对象
    return s == null ? null : s.remove(key, hash, null); // 调用segment的remove方法
}

// 根据hash值，得到对应的segment对象
private Segment<K,V> segmentForHash(int h) {
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;
    return (Segment<K,V>) UNSAFE.getObjectVolatile(segments, u);
}

public boolean remove(Object key, Object value) { // 根据key与value(全部匹配)，移除相应节点
    int hash = hash(key); // 计算hash
    Segment<K,V> s;
    return value != null && (s = segmentForHash(hash)) != null &&
        s.remove(key, hash, value) != null;
}
```

以上两个remove方法调用的是Segment的remove方法(加锁)：

```java
// 移除节点。如果value为null，只匹配key；否则key，value都必须匹配
final V remove(Object key, int hash, Object value) {
    if (!tryLock()) // 尝试加锁
        scanAndLock(key, hash); // 失败后，重试尝试加锁，直到获取到锁
  	// 到这里说明已经获取锁
    V oldValue = null; // 旧值
    try {
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash; // 根据hash计算桶位置
        HashEntry<K,V> e = entryAt(tab, index); // 得到对应桶位置的第一个节点
        HashEntry<K,V> pred = null; // 前一个节点
        while (e != null) { // 遍历查找节点
            K k;
            HashEntry<K,V> next = e.next; // 下一个节点
            if ((k = e.key) == key ||
                (e.hash == hash && key.equals(k))) { // 找到节点
                V v = e.value;
              	// value为null或者value不为null，但是value匹配
                if (value == null || value == v || value.equals(v)) {
                    if (pred == null) // 移除的是第一个节点
                        setEntryAt(tab, index, next);
                    else
                        pred.setNext(next); // 移除的是中间节点
                    ++modCount;
                    --count;
                    oldValue = v;
                }
                break;
            }
            pred = e; // 循环遍历
            e = next;
        }
    } finally {
        unlock();
    }
    return oldValue;
}

// remove操作时，未获取到锁，则重试尝试获取锁，直到获取锁成功，或者重试次数到达上限后，阻塞获取锁直到成功。
private void scanAndLock(Object key, int hash) {
    // similar to but simpler than scanAndLockForPut 逻辑与scanAndLockForPut类似
    HashEntry<K,V> first = entryForHash(this, hash); // 根据hash得到对应桶位置的第一个HashEntry
    HashEntry<K,V> e = first; 
    int retries = -1; // 当前重试获取锁的次数，在定位节点时为-1
    while (!tryLock()) { // 尝试获取锁，如果未成功，进入下面的逻辑
        HashEntry<K,V> f; // f变量在下面重新检查first节点时使用
        if (retries < 0) { // 还未定位到节点
            if (e == null || key.equals(e.key)) // 如果未找到节点，或者找到节点，将retries置为0
                retries = 0;
            else
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) { // 尝试获取锁次数加1，判断是否超过最大重试次数
            lock(); // 如果超过了，则阻塞直到获取锁成功
            break;
        }
        else if ((retries & 1) == 0 && // 每隔一次循环(retries为偶数)，检查first节点是否发生变化
                 (f = entryForHash(this, hash)) != first) { 
            e = first = f; // 如果变化，则需要重新遍历
            retries = -1;
        }
    }
}
```

##### 4.isEmpty()

该方法用于判断ConcurrentHashMap是否为空。(不加锁)

```java
public boolean isEmpty() {
    long sum = 0L; // sum记录所有segment的modCount之和
    final Segment<K,V>[] segments = this.segments;
    for (int j = 0; j < segments.length; ++j) {
        Segment<K,V> seg = segmentAt(segments, j); // 根据索引j，得到对应的segment
        if (seg != null) {
            if (seg.count != 0) // 如果有一个segment的count不为0，表明map不为空，返回false
                return false;
            sum += seg.modCount; // 累加modCount
        }
    }
  	// 重新检查modCount
    if (sum != 0L) { // recheck unless no modifications
        for (int j = 0; j < segments.length; ++j) {
            Segment<K,V> seg = segmentAt(segments, j);
            if (seg != null) {
                if (seg.count != 0) // 如果有一个segment的count不为0，表明map不为空，返回false
                    return false;
                sum -= seg.modCount; // 减去modCount
            }
        }
      	// 在第二次检查后，若发现sum不为0，表明此时map中存在数据的并发增删改(存在数据)，返回false
        if (sum != 0L) 
            return false;
    }
    return true;
}
// 根据索引j，得到对应的segment
static final <K,V> Segment<K,V> segmentAt(Segment<K,V>[] ss, int j) {
    long u = (j << SSHIFT) + SBASE;
    return ss == null ? null :
        (Segment<K,V>) UNSAFE.getObjectVolatile(ss, u);
}
```

isEmpty()检查segments的count是否为0，不为0直接返回false。其次统计modCount之和，如果在两次统计期间，发生了数据的并发增删改，则返回false，否则返回true。

##### 5.size()

size()方法计算ConcurrentHashMap中映射的个数。(可能加锁)

```java
static final int RETRIES_BEFORE_LOCK = 2; // 在对segment数组加锁之前的重试统计次数

public int size() {
    // Try a few times to get accurate count. On failure due to
    // continuous async changes in table, resort to locking.
    final Segment<K,V>[] segments = this.segments;
    int size; // KV映射的总个数
    boolean overflow; // true if size overflows 32 bits 是否上溢
    long sum;         // sum of modCounts  modCount总数
    long last = 0L;   // previous sum   modCount前一轮统计的总数
    int retries = -1; // first iteration isn't retry  第一个迭代统计不算重试
    try {
        for (;;) { // 循环多次统计
          	// 在第一次统计以及后续的两次重试统计中，数据都存在并发的增删改，导致统计结果前后不一致
          	// 则在retries达到RETRIES_BEFORE_LOCK(2)时，尝试对每个segment加锁(segment不存在
          	// ，则先创建)，且只会加锁一次，然后进行统计。
            if (retries++ == RETRIES_BEFORE_LOCK) {
                for (int j = 0; j < segments.length; ++j)
                    ensureSegment(j).lock(); // force creation 加锁
            }
            sum = 0L; // 初始化sum、size、overflow
            size = 0;
            overflow = false;
            for (int j = 0; j < segments.length; ++j) { // 统计
                Segment<K,V> seg = segmentAt(segments, j);
                if (seg != null) {
                    sum += seg.modCount; // 累加modCount
                    int c = seg.count; // 单个seg中的映射个数
                    if (c < 0 || (size += c) < 0)
                        overflow = true; // c小于0或者size+c小于0，表示上溢
                }
            }
          	// 若本次统计的modCount总数与上次统计的modCount总数相同，则表示在此期间没有数据的增删改，
          	// 直接正常退出
            if (sum == last) 
                break;
            last = sum; // 否则将本次统计的modCount总数赋值给last，更新，继续统计
        }
    } finally {
      	// 如果在计算size时进行了加锁，此时retries > RETRIES_BEFORE_LOCK(2)
        if (retries > RETRIES_BEFORE_LOCK) {
            for (int j = 0; j < segments.length; ++j)
                segmentAt(segments, j).unlock(); // 则依次释放锁
        }
    }
  	// 如果size溢出，返回Integer.MAX_VALUE；否则返回正常值
    return overflow ? Integer.MAX_VALUE : size; 
}
```

size()方法在统计KV映射的个数时，采取的策略是：首先进行一次正常的统计和两次重试统计，如果在此期间数据存在并发的增删改，导致统计数据前后不一致，则对segment数组加锁，然后再统计，直到两次数据统计一致，返回size。因此**size()方法在调用时可能导致当前线程阻塞(因为可能要获取segment的锁，而segment的锁有可能已被其他线程获得)和其他线程阻塞(因为size统计时获得了锁，其他线程无法获得锁进行增删改操作)**。

### 参考

- [juc系列-ConcurrentHashMap](https://www.jianshu.com/p/fadc5bc01e23)
- [JDK 1.7之 ConcurrentHashMap 源码分析](https://blog.csdn.net/crazy1235/article/details/76795383)
- [JDK ConcurrentHashMap](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/ConcurrentHashMap.html)