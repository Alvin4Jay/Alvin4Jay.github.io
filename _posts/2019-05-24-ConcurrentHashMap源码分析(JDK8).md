---
layout:     post
title:      ConcurrentHashMap源码分析(基于JDK8)
subtitle:   ConcurrentHashMap/HashMap/HashTable
date:       2019-05-24
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JUC
    - JCF
---

# ConcurrentHashMap源码分析(基于JDK8) #

[ConcurrentHashMap源码分析(JDK7)](https://xuanjian1992.top/2019/05/22/ConcurrentHashMap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(JDK7)/)这篇文章分析了ConcurrentHashMap在JDK7中的实现细节。JDK7中，ConcurrentHashMap采用了**分段锁**的机制，实现并发的更新操作，底层采用**数组+链表**的存储结构。其包含两个核心静态内部类**Segment和HashEntry**。一个 ConcurrentHashMap 实例中包含由若干个 Segment 对象组成的数组。特点如下：

- Segment继承ReentrantLock用来充当锁的角色，每个 Segment 对象守护每个散列映射表的若干个桶；
- HashEntry 用来封装映射表的键/值对；
- 每个桶是由若干个HashEntry对象链接起来的链表。

而在JDK8的实现中，已经抛弃了Segment分段锁机制，利用**CAS+Synchronized**来保证并发更新的安全性，底层采用**数组+链表+红黑树**的存储结构，如下图所示。下面详细分析JDK8 中ConcurrentHashMap的实现细节。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-04.png)

### 一、ConcurrentHashMap重要字段说明

(1)table：默认为null，初始化发生在第一次插入操作(put)，默认大小为16的数组，用来存储Node节点数据，扩容时大小总是2的幂次方。

(2)nextTable：默认为null，扩容时新生成的数组，其大小为原数组的两倍，这个变量是扩容时临时使用的。

(3)sizeCtl ：默认为0，用来控制table的初始化和扩容过程。

- -1 代表table正在初始化(见`initTable()`方法)；

- -N 表示有N-1个线程正在进行扩容操作；

  ![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-05.png?x-oss-process=style/markdown-pic)

  从上图可以看出，在扩容时，sizeCtl变量(32bit)可以分为**扩容标识符rs**和**扩容线程数**两部分，且扩容标识符最高位为1，扩容线程个数为0时最低位为1(从下面的源码分析可以看出)，因此由于扩容线程数计数位数(16bit)的限制，低16位最大值只能是65535。如果把扩容标识符整体看成是一个符号表示，那么若sizeCtl为-N，则表示有N-1个线程正在扩容。

- 其余情况：
	①如果table未初始化，表示table表示需要初始化的大小；②如果table已经初始化完成，则表示table的扩容阈值，默认是table大小的0.75倍，计算用公式`n - (n >>> 2)`。

(4)Node静态嵌套类:保存key，value及key的hash值的数据结构，普通节点。

```java
class Node<K,V> implements Map.Entry<K,V> {
  final int hash;
  final K key;
  volatile V val;
  volatile Node<K,V> next;
  ... 省略部分代码
}
```


其中value和next都用volatile修饰，保证并发的可见性。

(5)ForwardingNode静态嵌套类：一个特殊的Node节点，hash值为-1，其中存储nextTable的引用。只有table发生扩容的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为null或者已经被移动(transfer)。

```java
final class ForwardingNode<K,V> extends Node<K,V> {
  final Node<K,V>[] nextTable;
  ForwardingNode(Node<K,V>[] tab) {
      super(MOVED, null, null, null); // MOVED = -1
      this.nextTable = tab;
  }
}
```

(6)TreeBin静态嵌套类：一个特殊的Node节点，hash值为-2。在table某些桶位置，put操作后链表中节点过多时会转换成红黑树结构，此时用TreeBin替换整条链表，占位在这些桶上。TreeBin持有红黑树根节点root和TreeNode链表的引用。

```java
static final class TreeBin<K,V> extends Node<K,V> {
  TreeNode<K,V> root; // 红黑树根节点
  volatile TreeNode<K,V> first; // TreeNode链表第一个节点
  
  // 将TreeNode链表转换成红黑树结构
  TreeBin(TreeNode<K,V> b) {
    super(TREEBIN, null, null, null); // TREEBIN = -2
    this.first = b;
    ... 省略部分代码
	}    		
  
}  
```

(7)TreeNode静态嵌套类：TreeBin中用的节点。普通Node链表转为红黑树之前，会首先替换成TreeNode链表，再转为TreeBin(见`treeifyBin()/untreeify()`)。

```java
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;

    TreeNode(int hash, K key, V val, Node<K,V> next,
             TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }

   	...省略部分代码
}
```

### 二、ConcurrentHashMap初始化

```java
public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException(); // 参数检查
    if (initialCapacity < concurrencyLevel)   // Use at least as many bins
        initialCapacity = concurrencyLevel;   // as estimated threads
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size); // cap必须为2的幂次方
  	// 初始化只确定了sizeCtl的值，即table数组的大小，ConcurrentHashMap的初始容量
    this.sizeCtl = cap; 
}
```

**注意：**ConcurrentHashMap在构造函数中只会初始化sizeCtl值，并不会直接初始化table数组，而是延缓到第一次put操作。

### 三、table数组初始化

table数组的初始化操作会延缓到第一次put行为(见`putVal()`方法)。但是put是可以并发执行的，如何保证table只初始化一次？下面详细分析：

```java
// 并发的初始化table
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) { // table还未初始化
      	// sizeCtl小于0，说明table正在初始化，当前线程等待。从下面可以看出，多线程进入，
      	// 其中一个线程CAS成功，使得sizeCtl变为-1，从而拥有了初始化table的机会。
        if ((sc = sizeCtl) < 0)  
            Thread.yield(); // lost initialization race; just spin // 让出CPU，等待初始化完成
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { // 有且只有一个线程CAS成功修改sizeCtl为-1，进行table的初始化
            try {
                if ((tab = table) == null || tab.length == 0) { // 再次检查table
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY; // 2的n次 / 16
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n]; // 初始化
                    table = tab = nt; // 更新table
                    sc = n - (n >>> 2); // 算出扩容阈值
                }
            } finally {
                sizeCtl = sc; // sizeCtl设置为扩容阈值
            }
            break;
        }
    }
    return tab;
}
```

sizeCtl默认为0，如果ConcurrentHashMap实例化时有传参数，sizeCtl会是一个2的幂次方的值。所以执行第一次put操作的线程会执行Unsafe.compareAndSwapInt方法修改sizeCtl为-1，有且只有一个线程能够修改成功，其它线程通过Thread.yield()让出CPU时间片，等待table初始化完成。

### 四、put操作

put操作采用CAS+synchronized实现并发插入或更新操作。

```java
// put方法，返回旧值v
final V putVal(K key, V data, boolean onlyIfAbsent) {
    if (key == null || data == null) throw new NullPointerException(); // 参数null检查
    int hash = spread(key.hashCode()); // hashCode高低16位异或，减少碰撞，计算出hash
    int binCount = 0; // 桶中节点计数
    for (Node<K,V>[] tab = table;;) {  // 死循环
        Node<K,V> f; int n, i, fh;  
        if (tab == null || (n = tab.length) == 0) // table数组为空，还未初始化
            tab = initTable(); // 并发的初始化
        // i = (n - 1) & hash确定table数组中桶位置索引，等于取模运算hash%n
        // tabAt采用CAS获取table数组中的元素最新值(volatile读)
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) { // 这个桶位置为空null
            if (casTabAt(tab, i, null,   // 使用CAS插入新节点
                         new Node<K,V>(hash, key, data, null)))
                break;   // 节点CAS插入成功则线程退出，不成功则线程进入下一个循环
        }
      	// 当前节点f不为空且是ForwardingNode节点，表示ConcurrentHashMap当前正在扩容，
      	// 且这个位置其他线程已经转移节点完成，则当前线程帮助其他线程一起扩容、转移节点。
        else if ((fh = f.hash) == MOVED)  
            tab = helpTransfer(tab, f); // 帮助扩容，完成后线程进入下一个循环
        else { // 下面这种情况表示当前桶位置的节点正常
            V oldVal = null; // 旧值
	          // 获取当前节点的锁，避免当前线程插入节点的同时，其他线程进行扩容转移操作(扩容也是需要获取锁的)
            synchronized (f) {  
	              // 再次确认i索引处头节点是否是f，以防其他线程修改了(比如原头节点f被其他线程删除了，
              	// 桶的头结点发生变化)
                if (tabAt(tab, i) == f) {  
                    if (fh >= 0) {  // f的hash值>=0，表明是链表，而不是红黑树
                        binCount = 1; // 已经有1个节点f
                      	// 遍历链表，插入或者更新，同时统计节点数binCount
                        for (Node<K,V> e = f;; ++binCount) { 
                            K ek;
                          	// 找到key对应的映射，根据onlyIfAbsent变量确定是否更新value
                            if (e.hash == hash &&   
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) { 
                                oldVal = e.val;
                                if (!onlyIfAbsent) 
                                    e.val = data;
                                break;
                            }
                            Node<K,V> pred = e; // 前一个节点
                            if ((e = e.next) == null) { // 若key对应的映射找不到，则插入
                                pred.next = new Node<K,V>(hash, key,
                                                          data, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 如果是红黑树结构
                        Node<K,V> p;
                        binCount = 2;  // 直接置binCount为2
                      	// 以红黑树方式插入节点，如果返回null，表示key对应的映射原来不存在；
                      	// 若不为null，p表示原来key对应的映射
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, 
                                                       data)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) // 根据onlyIfAbsent变量确定是否更新value
                                p.val = data;
                        }
                    }
                }
            }
          	// 若put为更新操作，binCount在1到原链表长度之间；若为插入操作，binCount为原链表长度。
          	// 两种操作都>=1。binCount=0的情况为上述再次确认tabAt(tab, i) == f时不成立的case
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD) // 如果链表结点个数大于TREEIFY_THRESHOLD(8)
                    treeifyBin(tab, i); // 树化这个桶的节点
                if (oldVal != null)
                    return oldVal; // 如果是更新，直接返回旧值
                break;
            }
        }
    }
		// 插入操作，将当前ConcurrentHashMap的映射数量+1，检查是否需要扩容
    addCount(1L, binCount);
    return null;
}
```

##### 1.hash算法

```java
int hash = spread(key.hashCode());
// hashCode高低16位异或，减少碰撞
static final int spread(int h) {return (h ^ (h >>> 16)) & HASH_BITS;} 
```

##### 2.table初始化

```java
if (tab == null || (n = tab.length) == 0)
    tab = initTable();
```

若当前table尚未初始化，则进行并发的初始化操作，详见第三部分**tables数组初始化**

##### 3.定位桶索引位置

```java
int index = (n - 1) & hash // n是table的大小
```

##### 4.获取对应索引处的头结点f

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) { // 获取索引i处的节点
    return (Node<K,V>)U.getObjectVolatile(tab, ((long)i << ASHIFT) + ABASE);
}
```

在java内存模型中，我们已经知道每个线程都有一个工作内存，里面存储着table的副本，虽然table是volatile修饰的，但不能保证线程每次都拿到table中元素的最新值，`Unsafe.getObjectVolatile`可以直接获取指定内存的数据，保证了每次拿到的数据都是最新的。

##### 5.f为null的情况

```java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                    Node<K,V> c, Node<K,V> v) { // CAS设置索引i处的节点
    return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

如果f为null，说明table中这个位置第一次插入元素，利用`Unsafe.compareAndSwapObject`方法并发的插入Node节点。

- 如果CAS成功，说明Node节点已经插入，随后`addCount(1L, binCount)`方法会检查当前map是否需要进行扩容。
- 如果CAS失败，说明有其它线程提前插入了节点，当前线程进入下一个循环

##### 6.f的hash为-1的情况(helpTransfer)

```java
else if ((fh = f.hash) == MOVED) // MOVED=-1
    tab = helpTransfer(tab, f);
```

如果f的hash值为-1，说明当前f是ForwardingNode节点，意味着有其它线程正在扩容，则当前线程帮助一起进行扩容操作。

```java
// 帮助扩容。tab 原数组，f ForwardingNode节点
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc; // nextTab新数组，sc: sizeCtl
  	// 如果当前table不为null，f为ForwardingNode节点，nextTab新数组不为null，则进入下面的判断
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length); // 生成本次扩容的标识符，上面讲解sizeCtl字段时已介绍到
      	// 再次确认table、nextTable未发生变化，且sizeCtl<0，表明map当前正在扩容
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) { // 此时的sizeCtl可参考上面 扩容时sizeCtl字段组成 的介绍
          	// 如果存在以下情况，退出循环并返回
          	// 1.(sc >>> RESIZE_STAMP_SHIFT) != rs    sc >>> RESIZE_STAMP_SHIFT得到扩容标识符
          				sizeCtl发生了变化，表示本次扩容已经结束或者新的扩容已经开始
          	// 2.sc == (rs << RESIZE_STAMP_SHIFT) + 1
     // 此处JDK代码存在bug，见https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8214427
                  本次扩容已结束，扩容线程已经没了。因为从后面可以看到，发起扩容的线程会将
                  sizeCtl置为(rs << RESIZE_STAMP_SHIFT) + 2，后面每次扩容线程+1，sizeCtl置为
                  sizeCtl+1，因此这里sc == (rs << RESIZE_STAMP_SHIFT) + 1表明扩容线程已经没了，
                  扩容结束。
          	// 3.sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS
                  一起扩容的线程数已达到了最大值(sizeCtl低16位最大值65535)
          	// 4.transferIndex <= 0
                 由于扩容线程会对table数组进行分区扩容转移处理，而这是由transferIndex来分配的，
                 详见transfer方法。这个条件表明table已经没有区间分配给当前线程转移节点了。
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == (rs << RESIZE_STAMP_SHIFT) + 1 ||
                sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS || transferIndex <= 0)
                break;
          	// 上述情况都不成立，则尝试CAS将扩容线程数+1，进行扩容
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab); // 进行扩容，transfer方法下面分析
                break;
            }
        }
        return nextTab; // 返回新数组
    }
    return table;
}
// 生成本次扩容的标识符，n为扩容前数组的长度，即map容量。如n为16，则rs = 1000000000011011(二进制)
static final int resizeStamp(int n) { // RESIZE_STAMP_BITS=16
    return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
}
```

helpTransfer方法中线程在帮助扩容之前，首先生成扩容标识符rs，再判断sizeCtl<0，这样后才进入扩容操作或者根据情况直接退出。以下情况直接退出该方法：

- (sc >>> RESIZE_STAMP_SHIFT) != rs    

  sc >>> RESIZE_STAMP_SHIFT得到扩容标识符。sizeCtl发生了变化，表示本次扩容已经结束或者新的扩容已经开始

- sc == (rs << RESIZE_STAMP_SHIFT) + 1
  本次扩容已结束，扩容线程已经没了。因为从后面可以看到，发起扩容的线程会将sizeCtl置为(rs << RESIZE_STAMP_SHIFT) + 2，后面每次扩容线程+1，sizeCtl置为sizeCtl+1，因此这里sc == (rs << RESIZE_STAMP_SHIFT) + 1表明扩容线程已经没了，扩容结束。

- sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS
  一起扩容的线程数已达到了最大值(sizeCtl低16位最大值65535)

- transferIndex <= 0
  由于扩容线程会对table数组进行分区扩容转移处理，而这是由transferIndex来分配的，详见transfer方法。这个条件表明table已经没有区间分配给当前线程转移节点了。

##### 7.其他情况(hash>=0)

其余情况把新的Node节点按链表或红黑树的方式插入或者更新到合适的位置，这个过程采用内置同步锁实现并发。采用同步锁的作用：**避免当前线程插入节点的同时，其他线程对当前桶位置进行扩容转移操作(扩容也是需要获取锁的)，也即map在并发扩容的时候，是可以并发插入的**。

在节点f上进行同步，节点插入之前，再次利用`tabAt(tab, i) == f`判断，防止被其它线程修改。

1. 如果f.hash >= 0，说明f是链表结构的头结点，遍历链表，如果找到对应的node节点，则修改value，否则在链表尾部加入节点。
2. 如果f是TreeBin类型节点，说明f是红黑树根节点，则在树结构上遍历元素，更新或增加节点。
3. 如果链表中节点数`binCount >= TREEIFY_THRESHOLD(`默认是8)，则把链表转化为红黑树结构。

##### 8.树化链表结构(treeifyBin)

put插入操作后，如果链表过长，会将其转为红黑树结构，这是通过treeifyBin方法实现的。

```java
// 树化tab索引index处的链表结构
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
	      // 如果tab的长度，即map容量，目前小于最小树化容量64，则优先进行2倍的扩容操作，而不是树化操作
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY) 
            tryPresize(n << 1); // 优先2倍扩容
     		// 否则，先检查index索引处节点是否不为null，以及hash值是否>=0(普通Node链表)
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) { // 同步
              	// 再次检查索引index处节点是否发生变化，防止被其他线程修改
                if (tabAt(tab, index) == b) { 
                  	// Node链表在转为红黑树时，会先构造为TreeNode链表，
                  	// 再由TreeNode链表生成TreeBin红黑树结构
                    TreeNode<K,V> hd = null, tl = null; // TreeNode链表头、尾节点
                    for (Node<K,V> e = b; e != null; e = e.next) { // 生成TreeNode链表
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    }
                  	// 由TreeNode链表构造红黑树TreeBin，然后将index处设置为TreeBin
                    setTabAt(tab, index, new TreeBin<K,V>(hd)); 
                }
            }
        }
    }
}  
```

treeifyBin操作执行时，会先判断当前table长度是否小于最小树化容量64，是则优先进行扩容操作(tryPresize)，解决链表过长问题；否则先构造TreeNode链表，再由该链表生成红黑树结构。

tryPresize操作：

```java
// 优先扩容
private final void tryPresize(int size) {
    int c = (size >= (MAXIMUM_CAPACITY >>> 1)) ? MAXIMUM_CAPACITY :
        tableSizeFor(size + (size >>> 1) + 1); // 根据size，计算扩容后的容量
    int sc;
    while ((sc = sizeCtl) >= 0) { // sizeCtl>=0表示当前map未进行扩容，或者尚未初始化
        Node<K,V>[] tab = table; int n;
      	// table尚未初始化，对应 public ConcurrentHashMap(Map<? extends K, ? extends V> m)
      	// 构造函数中调用putAll方法，putAll再调用tryPresize(m.size());进行初始化table
        if (tab == null || (n = tab.length) == 0) { 
            n = (sc > c) ? sc : c; // 确定数组长度
          	// CAS设置sizeCtl=-1，表示table正在初始化
            if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { 
                try {
                    if (table == tab) {
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = nt; // 更新table
                        sc = n - (n >>> 2); // 计算sizeCtl
                    }
                } finally {
                    sizeCtl = sc;// 更新sizeCtl
                }
            }
        }
      	// 所需的扩容容量小于当前的sizeCtl阈值(无需扩容)，或者数组长度n已大于MAXIMUM_CAPACITY，
      	// 直接退出、返回
        else if (c <= sc || n >= MAXIMUM_CAPACITY)
            break;
        else if (tab == table) { // 再次确认table未发生变化
            int rs = resizeStamp(n); // 得到扩容标识符
            if (sc < 0) { // 其他线程正在扩容
                Node<K,V>[] nt;
              	// 如果存在以下情况，退出循环并返回
              	// 1.(sc >>> RESIZE_STAMP_SHIFT) != rs    
              		sc >>> RESIZE_STAMP_SHIFT得到扩容标识符
          				sizeCtl发生了变化，表示本次扩容已经结束或者新的扩容已经开始
          			// 2.sc == (rs << RESIZE_STAMP_SHIFT) + 1
                  本次扩容已结束，扩容线程已经没了。因为从后面可以看到，发起扩容的线程会将
                  sizeCtl置为(rs << RESIZE_STAMP_SHIFT) + 2，后面每次扩容线程+1，sizeCtl置为
                  sizeCtl+1，因此这里sc == (rs << RESIZE_STAMP_SHIFT) + 1表明扩容线程已经没了，
                  扩容结束。
          			// 3.sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS
                  一起扩容的线程数已达到了最大值(sizeCtl低16位最大值65535)
                // 4.(nt = nextTable) == null 
                  nextTable=null表示当前扩容操作已经完成，nextTable已被置为null
          			// 5.transferIndex <= 0
                 由于扩容线程会对table数组进行分区扩容转移处理，而这是由transferIndex来分配的，
                 详见transfer方法。这个条件表明table已经没有区间分配给当前线程转移节点了。
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs 
                    || sc == (rs << RESIZE_STAMP_SHIFT ) + 1 
                    || sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS 
                    || (nt = nextTable) == null 
                    || transferIndex <= 0)
                    break;
              	// 上述情况都不成立，则尝试CAS将扩容线程数+1，进行扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt); // 开始扩容操作
            }
        // 当前未扩容，则由当前线程发起扩容操作，通过CAS将sizeCtl设置为rs << RESIZE_STAMP_SHIFT) + 2
        // 即sizeCtl= 高16位扩容标识符 + 低16位00000000 0000 0010(二进制)
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null); // 开始扩容操作
        }
    }
}
```

tryPresize操作中有如下情况：

- 判断table是否需要初始化(**针对构造器传入Map参数，初始化的场景**)，若需初始化，则进行初始化操作并更新sizeCtl；

- 所需的扩容容量若小于当前的sizeCtl阈值(无需扩容)，或者数组长度n已大于MAXIMUM_CAPACITY，直接退出、返回；

- 进行扩容操作。

  - 若当前未进行扩容，则由当前线程发起扩容操作。

    ```java
    else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null); // 开始扩容操作
    ```

  - 若map当前正在扩容，

    存在以下情况直接退出：

    - (sc >>> RESIZE_STAMP_SHIFT) != rs
    - sc == (rs << RESIZE_STAMP_SHIFT) + 1
    - sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS
    - (nt = nextTable) == null 
    - transferIndex <= 0

    否则，帮助扩容：

    ```java
    // 上述情况都不成立，则尝试CAS将扩容线程数+1，进行扩容
    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
       transfer(tab, nt); // 开始扩容操作
    ```

##### 9.addCount(1L, binCount)

put操作节点插入之后，需要更新map映射个数，并检查是否需要扩容，这通过addCount来实现。

```java
// 增加count计数，映射个数+x。check表示是否需要检查扩容。
// ①如果check<0，不检查扩容；
// ②在增加count计数时未发生竞争(uncontended=true)，1. check<=1，不检查扩容 2.check>1，检查扩容
private final void addCount(long x, int check) {
  	// 以下增加计数的逻辑可参考LongAdder: 			
  	// https://xuanjian1992.top/2019/02/22/LongAdder%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/
    CounterCell[] as; long b, s;
  	// CounterCell数组不为null或CAS更新baseCount失败，进入下面的逻辑。若CAS成功，直接进入后面的扩容检查
    if ((as = counterCells) != null || 
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true; // 是否没有竞争
      	// as为null或as数组没有元素或根据线程threadLocalRandomProbe得到数组元素为null
      	// 或用CAS递增得到的cell失败
        if (as == null || (m = as.length - 1) < 0 ||
      			// getProbe()获取当前线程的threadLocalRandomProbe值，与m求与运算，能获得一个
            // 最小0最大m的数，即得到CounterCell数组中元素的索引
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            // 根据得到的索引，取出CounterCell元素，进行CAS递增
            !(uncontended = U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
          	// 如果递增失败，即出现竞争，调用fullAddCount方法进行增加计数，此时不检查扩容，完成后直接退出
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1) // 上面CAS递增CounterCell的value成功，说明未出现竞争。check<=1，不检查扩容
            return;
        s = sumCount(); // check>1，计算map当前映射的个数，然后进入下面的扩容检查操作
    }
    if (check >= 0) { // check>=0，检查扩容。put操作默认检查是否扩容
        Node<K,V>[] tab, nt; int n, sc;
      	// 若当前映射个数s大于阈值sizeCtl，且table不为null，且当前容量小于MAXIMUM_CAPACITY，则扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
          	// 以下逻辑与tryPresize中的类似，不再介绍
            int rs = resizeStamp(n); 
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs 
                    || sc == (rs << RESIZE_STAMP_SHIFT ) + 1 
                    || sc == (rs << RESIZE_STAMP_SHIFT) + MAX_RESIZERS 
                    || (nt = nextTable) == null 
                    || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount(); // 重新统计映射个数，检查是否还需要扩容。有可能扩容期间，节点又插入很多。
        }
    }
}
```

addCount操作可分为两步，第一步增加count计数，第二步检查是否需要扩容。

- 增加count计数的逻辑与LongAdder类似，可参考[LongAdder源码分析](https://xuanjian1992.top/2019/02/22/LongAdder%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/)
- 检查扩容：若当前映射个数s大于阈值sizeCtl，且table不为null，且当前容量小于MAXIMUM_CAPACITY，则需要扩容。逻辑与tryPresize中的类似。扩容完之后，还需要重新统计映射个数，检查是否还需要扩容。因为ConcurrentHashMap支持扩容的时候并发插入，有可能扩容期间，节点又插入很多。

以上便是完整的put操作逻辑，还未针对真正的扩容操作transfer进行分析，下面结合图示，逐行分析其实现。

### 五、扩容操作transfer

当table容量不足的时候，即table中的节点(映射)数量达到容量阈值sizeCtl，需要对table进行扩容。

##### 1.扩容的情况

当往ConcurrentHashMap中成功插入一个key/value节点时，有可能触发扩容动作：

①如果新增节点之后，所在链表的元素个数达到了阈值 8，则会调用treeifyBin方法把链表转换成红黑树，不过在结构转换之前，会对数组长度进行判断，实现如下：
							![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-06.png?x-oss-process=style/markdown-pic)

如果数组长度n小于阈值MIN_TREEIFY_CAPACITY(使用扩容而不是树化)，默认是64，则会调用tryPresize方法把数组长度扩大到原来的两倍，并触发transfer方法，重新调整节点的位置。
							![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-07.png?x-oss-process=style/markdown-pic)

②新增节点之后，会调用addCount方法增加元素个数，并检查是否需要进行扩容，当数组元素个数达到阈值时，会触发transfer方法，重新调整节点的位置。
							![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-08.png?x-oss-process=style/markdown-pic)

##### 2.transfer实现

transfer方法实现了在并发的情况下，高效的从原始数组往新数组中移动元素，假设扩容之前节点的分布如下，这里区分蓝色节点和红色节点，是为了后续更好的分析：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-09.png?x-oss-process=style/markdown-pic)

在上图中，第14个槽位插入新节点之后，链表元素个数已经达到了8，且数组长度为16，优先通过扩容来缓解链表过长的问题。treeifyBin、tryPresize操作的逻辑上面已经介绍，下面详细介绍真正的扩容实现transfer：

**transfer方法(源码加注释)**

transfer方法实现了并发的扩容操作，并且在并发扩容期间，其他put、get线程可以并发执行。

```java
// 一个过渡的table表  只有在扩容的时候才会使用
private transient volatile Node<K,V>[] nextTable;

private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;  // n为原table长度，stride表示每个扩容线程处理的桶区间大小
		// 计算每个扩容线程处理的桶区间大小，可见每个扩容线程处理扩容任务是不会出现竞争的，
  	// 因为他们处理的区间不会重叠
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range 划分table范围
		// nextTable未初始化，由发起扩容的线程初始化
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1]; // n << 1原来的两倍
            nextTab = nt; // 赋值给nextTab
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE; // 新建数组OOM， sizeCtl使用int最大值
            return;
        }
        nextTable = nextTab; // 更新nextTable变量
        transferIndex = n; // 扩容转移索引，初始化大小为原先数组大小
    }
    int nextn = nextTab.length; // 新数组长度
		// 初始化ForwardingNode节点，其中保存了新数组nextTable的引用，在处理完每个槽位的节点之后
  	// 当做占位节点，表示该槽位已经处理过了。并指示其他put线程在看到该节点后一起帮助加速扩容
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
		// 当前线程能否前进至下一个位置(--i)，继续转移下一个位置的节点。一般在转移完某个位置处的节点后
  	// advance变量置为true，表示可以前进至下一节点。初始值为true。
    boolean advance = true;
	  // 扩容结束标志
    boolean finishing = false; // to ensure sweep before committing nextTab
	  // 置i，bound=0，i表示当前处理的桶位置，bound表示当前线程目前所分配的处理区间的最小索引位置，即该线
  	// 程目前的任务中所能处理的最小索引边界。这两个值在下面while循环中进行初始分配。
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
	      // advance=true，则当前线程可以前进至下一个位置；advance=false，表示不能。
        while (advance) { 
            int nextIndex, nextBound;
          	// --i得到下一个位置的索引，如果此时小于最小边界bound，表明当前线程已经处理完分配的table区间
          	// 转移任务，那么就要在下面的if判断中继续领取处理的区间。若--i>=bound，则当前线程继续处理下一
          	// 个位置(--i)的节点转移，advance=false，表示目前不能再前进。
          	// finishing=true，表示当前扩容已经结束，advance=false，表示目前不能再前进。
            if (--i >= bound || finishing) 
                advance = false;
          	// 到这个if，说明当前线程已经处理完分配的table区间转移任务，
          	// 如果此时扩容转移索引<=0，表示整个table已经没有区间可以分配给当前线程继续转移了
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1; // 那么i置为-1，表示当前线程可以结束扩容了。advance=false，表示不能再前进了。
                advance = false;
            }
          	// 到这个if，说明当前线程可以领取table区间任务，进行扩容转移。
          	// 1.nextIndex-1表示当前线程目前领取的区间任务的起始桶位置。
          	// 2.nextBound = (nextIndex > stride ? nextIndex - stride : 0) 计算当前线程目前领
          	// 取的区间任务的最小索引位置边界。
          	// 3.CAS更新transferIndex，其他新加入的扩容线程或者已处理完区间任务的线程可根据此变量的
          	// 值判断是否还能分配任务
            else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
                bound = nextBound; // 当前线程目前领取的区间任务的最小索引位置边界
                i = nextIndex - 1; // 当前线程目前领取的区间任务的起始桶位置
                advance = false; // 需要处理完起始桶位置的节点，才能继续处理下一个位置的节点
            }
        }
      	// 1.i < 0 由上面(nextIndex = transferIndex) <= 0条件可知，此时当前线程已经完成扩容转移任
      	// 务，且整个table已经没有区间可以分配给当前线程继续转移了，因此线程可以退出了
      	// 2.i >= n || i + n >= nextn 这两个条件不清楚为什么会出现??
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {  // 如果已经完成全部的扩容操作，则更新相关的变量，再退出
                nextTable = null; // nextTable置为null，表示扩容已结束
                table = nextTab; // 更新table为最新的数组
                sizeCtl = (n << 1) - (n >>> 1); // 更新扩容阈值，相当于现在容量的0.75倍
                return;
            }
          	// 若扩容还未全部完成，则利用CAS更新sizeCtl，将sizeCtl减1，表示当前线程帮助扩容结束了，
          	// sizeCtl的低16位减一。
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
              	// 下面这个if条件如果不满足，即(sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT，
              	// 也即sc = resizeStamp(n) << RESIZE_STAMP_SHIFT + 2， 表示当前在CAS更新sizeCtl
              	// 之后，还剩余另外一个线程在扩容，当前线程是倒数第二个线程。其余情况直接退出。
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
              	// 到这里，说明当前线程是倒数第二个还在的线程，则由该线程重新检查一遍table表
                finishing = advance = true; //  finishing置为true，表示扩容完成
                i = n; // advance置为true，i置为n，重新检查table表
            }
        }
				// 如果遍历到的节点为空，则放入ForwardingNode节点，用来指示其他put线程在看到该节点后一起帮助
      	// 加速扩容
        else if ((f = tabAt(tab, i)) == null)
            advance = casTabAt(tab, i, null, fwd); // 置advance为true，前进至下一桶位置
				// 如果遍历到ForwardingNode节点，说明这个点已经被处理过了
	      // 此处判断发生在扩容完成后，recheck线程重新检查的时候。作用是告诉recheck线程，该位置已经处理完成
        else if ((fh = f.hash) == MOVED) 
            advance = true; // already processed 置advance为true，前进至下一桶位置
        else { // 下面是正常的节点转移操作
						// 对头节点f进行加锁，禁止插入节点的其他线程在该桶节点转移期间，在该桶位置进行节点插入操作
            synchronized (f) {
								// 检查i索引处的头结点是否发送变化
                if (tabAt(tab, i) == f) {
										// 分别保存hash值的第X位为0和1的节点
                    Node<K,V> ln, hn;
										// 如果fh>=0，证明这是一个Node链表
                    if (fh >= 0) {
						 						// 以下的逻辑在完成的工作是构造两个链表ln/hn
                      	// n为原数组的长度，为2的幂次方，fh&n计算fh的第X位(从0开始)为0或1。2的X次方为n
                        int runBit = fh & n; 
                      	// lastRun表示该桶位置上，链表从头遍历到尾部，runBit开始不再变化的起始节点
                        Node<K,V> lastRun = f; 
                        for (Node<K,V> p = f.next; p != null; p = p.next) { // 开始遍历
                            int b = p.hash & n; // 计算hash&n
	                          // 如果前后不相等，更新runBit、lastRun为当前值、节点
                            if (b != runBit) { 
                                runBit = b;
                                lastRun = p;
                            }
                        }
                      	// 遍历完后，runBit为0，说明lastRun之后的节点hash&n都为0
                        if (runBit == 0) { 
	                          // 则直接复用lastRun之后的节点，ln指向该lastRun指向的节点
                            ln = lastRun; 
                            hn = null; // hn置为null
                        }
                      	// 遍历完后，runBit为1，说明lastRun之后的节点hash&n都为1
                        else {
                          	// 则直接复用lastRun之后的节点，hn指向该lastRun指向的节点
                            hn = lastRun;
                            ln = null; // ln置为null
                        }
                      	// 重新遍历头节点与lastRun之间的节点，头插法插入到hn，ln
                      	// 注意：这里是新建节点构建了hn、ln的部分元素，并且复用了原链表lastRun之后
                      	// 的节点。说明原链表的指针还存在，可以并发的进行get等操作进行获取，不会有影响。
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0) // ph第X位为0，新建节点使用头插法插入到ln
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else // ph第X位为1，新建节点使用头插法插入到hn
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln); // 将ln挂在nextTable的i位置上
                        setTabAt(nextTab, i + n, hn);// 将hn挂在nextTable的i+n位置上
                        setTabAt(tab, i, fwd); // 在table的i位置上插入fwd节点,表示已经处理过该桶
                        advance = true; // 置advance为true，表示可以前进至下一个位置处理了。
                    }
                    else if (f instanceof TreeBin) { // 红黑树结构，与上面的处理过程类似
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                      	// hash&n为0的TreeNode链表头结点lo，尾节点loTail
                        TreeNode<K,V> lo = null, loTail = null;
												// hash&n为1的TreeNode链表头结点hi，尾节点hiTail
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0; // lo节点、hi节点个数
												// 遍历原链表，构造两个TreeNode链表
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null); // 新建TreeNode节点
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null) // lo尾部添加
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null) // hi尾部添加
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
												// 如果拆分后的lo链表的节点数量已经<=6个，就需要重新转化为Node链表;否则先判断
                      	// hc是否为0，是的话可以直接复用原t红黑树，否则需要构建红黑树。
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                      	// 如果拆分后的hn链表的节点数量已经<=6个，就需要重新转化为Node链表;否则先判断
                      	// hc是否为0，是的话可以直接复用原t红黑树，否则需要构建红黑树。
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln); // 将ln挂在nextTable的i位置上
                        setTabAt(nextTab, i + n, hn); // 将hn挂在nextTable的i+n位置上
                        setTabAt(tab, i, fwd); // 在table的i位置上插入fwd节点,表示已经处理过该桶
                        advance = true; // 置advance为true，表示可以前进至下一个位置处理了。
                    }
                }
            }
        }
    }
}
```

1.根据当前数组长度n，新建一个两倍长度的数组nextTable；

```java
if (nextTab == null) {            // initiating
    try {
        @SuppressWarnings("unchecked")
        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
        nextTab = nt;
    } catch (Throwable ex) {      // try to cope with OOME
        sizeCtl = Integer.MAX_VALUE;
        return;
    }
    nextTable = nextTab;
    transferIndex = n;
}
```

2.初始化ForwardingNode节点，其中保存了新数组nextTable的引用，在处理完每个槽位的节点之后当做占位节点，表示该槽位已经处理过了；

```java
int nextn = nextTab.length;
ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
boolean advance = true;
boolean finishing = false; // to ensure sweep before committing nextTab 
```

3.通过for自循环处理每个槽位中的链表元素，默认advace为true，通过CAS设置transferIndex属性值，并初始化i和bound值，i表示当前处理的槽位序号，bound表示需要处理的槽位边界，先处理槽位15的节点；

```java
for (int i = 0, bound = 0;;) {
    Node<K,V> f; int fh;
    while (advance) {
        int nextIndex, nextBound;
        if (--i >= bound || finishing)
            advance = false;
        else if ((nextIndex = transferIndex) <= 0) {
            i = -1;
            advance = false;
        }
        else if (U.compareAndSwapInt
                 (this, TRANSFERINDEX, nextIndex,
                  nextBound = (nextIndex > stride ?
                               nextIndex - stride : 0))) {
            bound = nextBound;
            i = nextIndex - 1;
            advance = false;
        }
    }
  ...
```

4.在当前假设条件下，槽位15中没有节点，则通过CAS插入在第二步中初始化的ForwardingNode节点，用来指示其他put线程在看到该节点后一起帮助加速扩容。

```java
else if ((f = tabAt(tab, i)) == null)
    advance = casTabAt(tab, i, null, fwd);
```

5.如果遍历到ForwardingNode节点，说明这个点已经被处理过了。这个判断发生在扩容完成后，recheck线程重新检查的时候。作用是告诉recheck线程，该位置已经处理完成。

```java
else if ((fh = f.hash) == MOVED)
    advance = true; // already processed
```

6.处理槽位14的节点，是一个链表结构，先定义两个变量节点ln和hn(lowNode和highNode)，分别保存hash值的第X位为0和1的节点，具体实现如下：

```java
synchronized (f) {
    if (tabAt(tab, i) == f) {
        Node<K,V> ln, hn;
        if (fh >= 0) {
            int runBit = fh & n; // 重要
            Node<K,V> lastRun = f;
            for (Node<K,V> p = f.next; p != null; p = p.next) {
                int b = p.hash & n;
                if (b != runBit) {
                    runBit = b;
                    lastRun = p;
                }
            }
            if (runBit == 0) {
                ln = lastRun;
                hn = null;
            }
            else {
                hn = lastRun;
                ln = null;
            }
            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                int ph = p.hash; K pk = p.key; V pv = p.val;
                if ((ph & n) == 0)
                    ln = new Node<K,V>(ph, pk, pv, ln);
                else
                    hn = new Node<K,V>(ph, pk, pv, hn);
            }
            setTabAt(nextTab, i, ln);
            setTabAt(nextTab, i + n, hn);
            setTabAt(tab, i, fwd);
            advance = true;
        }
```

使用fh&n可以快速把链表中的元素区分成两类(为什么要分为两类？见[ConcurrentHashMap 扩容分析拾遗](https://www.jianshu.com/p/dc69edc17b32))，A类是hash值的第X位为0，B类是hash值的第X位为1，并通过lastRun记录链表上从头遍历到尾部，第X位的值不在发生变化的起始节点，A类和B类节点可以分散到新数组的槽位14和30(14+16)中，在原数组的槽位14中，蓝色节点第X为0，红色节点第X为1，把链表拉平显示如下：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-10.png?x-oss-process=style/markdown-pic)

- 通过遍历链表，记录runBit和lastRun，分别为1和节点6，所以设置hn为节点6，ln为null；

- 重新遍历链表，以lastRun节点为终止条件，根据第X位的值继续构造ln链表和hn链表：

	![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/CHM-11.png?x-oss-process=style/markdown-pic)

最后，通过CAS把ln链表设置到新数组的i位置，hn链表设置到i+n的位置。

7.如果该槽位是红黑树结构，则构造TreeNode链表lo和hi，遍历红黑树中的节点，同样根据hash&n算法，把节点分为两类，分别插入到lo和hi为头的TreeNode链表中，根据lo和hi链表中的元素个数分别生成ln和hn节点，其中ln节点的生成逻辑如下：

- 如果lo链表的元素个数小于等于UNTREEIFY_THRESHOLD，默认为6，则通过untreeify方法把树节点TreeNode链表转化成普通Node节点链表；

- 否则判断hi链表中的元素个数是否等于0：如果等于0，表示lo链表中包含了所有原始节点，直接复用原红黑树即可，则设置原始红黑树给ln，否则根据lo链表重新构造红黑树。

  ```java
  else if (f instanceof TreeBin) {
      TreeBin<K,V> t = (TreeBin<K,V>)f;
      TreeNode<K,V> lo = null, loTail = null;
      TreeNode<K,V> hi = null, hiTail = null;
      int lc = 0, hc = 0;
      for (Node<K,V> e = t.first; e != null; e = e.next) {
          int h = e.hash;
          TreeNode<K,V> p = new TreeNode<K,V>
              (h, e.key, e.val, null, null);
          if ((h & n) == 0) {
              if ((p.prev = loTail) == null)
                  lo = p;
              else
                  loTail.next = p;
              loTail = p;
              ++lc;
          }
          else {
              if ((p.prev = hiTail) == null)
                  hi = p;
              else
                  hiTail.next = p;
              hiTail = p;
              ++hc;
          }
      }
      ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
          (hc != 0) ? new TreeBin<K,V>(lo) : t;
      hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
          (lc != 0) ? new TreeBin<K,V>(hi) : t;
      setTabAt(nextTab, i, ln);
      setTabAt(nextTab, i + n, hn);
      setTabAt(tab, i, fwd);
      advance = true;
  }
  ```

最后，同样的通过CAS把ln设置到新数组的i位置，hn设置到i+n位置。

### 六、get操作

```java
// 根据key获取value
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
		// 计算hash
    int h = spread(key.hashCode());
		// 根据hash确定桶位置，并判断桶位置头节点是否为null
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
				// 如果key对应的节点为头节点，直接返回
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
				// hash小于0，①可能是红黑树，在红黑树中寻找 ②可能是ForwardingNode节点，则
      	// 在ForwardingNode节点对应的nextTable新数组中查找
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
				// 否则遍历链表，找到对应的节点就返回其值
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null; // 找不到，返回null
}
```

- 计算key的hash值，并得到桶位置以及头结点，判断该头节点是否是key对应的节点，是则返回；
- 如果hash<0，表明桶位置①可能是红黑树，则在红黑树中寻找 ②可能是ForwardingNode节点，则在ForwardingNode节点对应的nextTable新数组中查找；
- 否则遍历链表，找到对应的节点就返回其值。

①TreeBin查找:

```java
final Node<K,V> find(int h, Object k) {
    if (k != null) {
        for (Node<K,V> e = first; e != null; ) {
            int s; K ek;
            if (((s = lockState) & (WAITER|WRITER)) != 0) {
                if (e.hash == h &&
                    ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                e = e.next;
            }
            else if (U.compareAndSwapInt(this, LOCKSTATE, s,
                                         s + READER)) {
                TreeNode<K,V> r, p;
                try {
                    p = ((r = root) == null ? null :
                         r.findTreeNode(h, k, null));
                } finally {
                    Thread w;
                    if (U.getAndAddInt(this, LOCKSTATE, -READER) ==
                        (READER|WAITER) && (w = waiter) != null)
                        LockSupport.unpark(w);
                }
                return p;
            }
        }
    }
    return null;
}
```

②ForwardingNode节点对应的nextTable新数组中查找

```java
Node<K,V> find(int h, Object k) {
    // loop to avoid arbitrarily deep recursion on forwarding nodes
    outer: for (Node<K,V>[] tab = nextTable;;) { // tab为新数组
        Node<K,V> e; int n;
        if (k == null || tab == null || (n = tab.length) == 0 ||
            // n为新数组长度，(n - 1) & h定位到新数组中的桶位置
            (e = tabAt(tab, (n - 1) & h)) == null)  // 获取头结点e
            return null;
        for (;;) {
            int eh; K ek;
            if ((eh = e.hash) == h &&
                ((ek = e.key) == k || (ek != null && k.equals(ek)))) // 头结点命中
                return e;
            if (eh < 0) { // hash < 0
                if (e instanceof ForwardingNode) { // e为ForwardingNode节点
                    tab = ((ForwardingNode<K,V>)e).nextTable; // 更新tab为e指向的nextTable
                    continue outer; // 使用循环而不是递归，继续查找
                }
                else // 红黑树结构，在红黑树中查找
                    return e.find(h, k);
            }
            if ((e = e.next) == null)
                return null;
        }
    }
}
```

可见在并发扩容时，完全可以使用get操作进行查询，如果查询时由节点hash<0得到当前map正在扩容，则直接到扩容的新数组nextTable中查找；如果hash>=0，也有可能当前桶节点正在转移，即还未转移完成，未将ForwardingNode放置到桶上，根据上述transfer方法构造ln、hn链表的逻辑，可见这种情况下依然可以获取到元素，因为原链表的指针关系并未发生变化。

### 七、remove/replace操作

remove/replace操作，底层使用的是replaceNode方法。

```java
public V remove(Object key) { // 根据key移除映射
    return replaceNode(key, null, null);
}
public boolean remove(Object key, Object value) { // 根据key，value移除映射
    if (key == null)
        throw new NullPointerException();
    return value != null && replaceNode(key, null, value) != null;
}
public V replace(K key, V value) { // 将key对应映射的值替代为value
    if (key == null || value == null)
        throw new NullPointerException();
    return replaceNode(key, value, null);
}
// 将key、oldValue对应映射的值替代为newValue
public boolean replace(K key, V oldValue, V newValue) { 
    if (key == null || oldValue == null || newValue == null)
        throw new NullPointerException();
    return replaceNode(key, newValue, oldValue) != null;
}

// 移除节点或者替换节点的值 cv表示匹配节点的当前值；value表示替换的值，为null，表示删除
final V replaceNode(Object key, V value, Object cv) {
    int hash = spread(key.hashCode()); // 计算hash
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        if (tab == null || (n = tab.length) == 0 ||
            (f = tabAt(tab, i = (n - 1) & hash)) == null) // 定位到的桶位置头结点为null，返回null
            break;
        else if ((fh = f.hash) == MOVED) // 正在扩容，则帮助扩容
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null; // 旧值
            boolean validated = false;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // Node链表
                        validated = true;
                        for (Node<K,V> e = f, pred = null;;) { // 遍历查找
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) { // 命中
                                V ev = e.val; // 当前值
	                              // 需要匹配的值cv为null，表示不需要匹配值，或者值匹配
                                if (cv == null || cv == ev ||
                                    (ev != null && cv.equals(ev))) { 
                                    oldVal = ev;
                                    if (value != null) // value不为null，更新
                                        e.val = value;
                                    else if (pred != null) // 替换的值为null，表示删除
                                        pred.next = e.next;
                                    else
                                        setTabAt(tab, i, e.next);
                                }
                                break;
                            }
                            pred = e;
                            if ((e = e.next) == null)
                                break;
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树结构
                        validated = true;
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> r, p;
                        if ((r = t.root) != null &&
                            (p = r.findTreeNode(hash, key, null)) != null) {
                            V pv = p.val; // 逻辑与上述Node链表处理逻辑相同
                            if (cv == null || cv == pv ||
                                (pv != null && cv.equals(pv))) {
                                oldVal = pv;
                                if (value != null)
                                    p.val = value;
                                else if (t.removeTreeNode(p))
                                    setTabAt(tab, i, untreeify(t.first));
                            }
                        }
                    }
                }
            }
            if (validated) {
                if (oldVal != null) { // 映射存在，旧值不为null
                    if (value == null) // value为null，表示删除节点
                        addCount(-1L, -1); // 映射计数-1，不需要检查扩容
                    return oldVal;
                }
                break;
            }
        }
    }
    return null;
}
```

### 八、size()/isEmpty()方法

JDK8中的size()/isEmpty()逻辑相对于[JDK7中的逻辑]([https://xuanjian1992.top/2019/05/22/ConcurrentHashMap%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90(JDK7)/](https://xuanjian1992.top/2019/05/22/ConcurrentHashMap源码分析(JDK7)/)简单的多。

##### 1.size()

不加锁。

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 : // 瞬时值可能小于0，则返回0
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}
// 统计map中的映射个数
final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount; // 基础值
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value; // 将CounterCell数组中元素的各个value域值累加
        }
    }
    return sum;
}
```

##### 2.isEmpty()

```java
public boolean isEmpty() {
    return sumCount() <= 0L; // 忽略瞬时负值
}
```

### 九、红黑树的构造

如果链表结构中元素超过TREEIFY_THRESHOLD阈值，默认为8个，则把链表转化为红黑树，提高遍历查询效率。

```java
if (binCount != 0) {
    if (binCount >= TREEIFY_THRESHOLD) // 8
        treeifyBin(tab, i); // 树化
    if (oldVal != null)
        return oldVal;
    break;
}
// 构造树
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1); // 如果数组大小小于64, 优先扩容
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) { // 再次确认
                    TreeNode<K,V> hd = null, tl = null;
                    for (Node<K,V> e = b; e != null; e = e.next) {
                        TreeNode<K,V> p =
                            new TreeNode<K,V>(e.hash, e.key, e.val,
                                              null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    } // 构造TreeNode链表结构
                    setTabAt(tab, index, new TreeBin<K,V>(hd)); // 生成TreeBin结构，设置到index处
                }
            }
        }
    }
}
```

可以看出，生成TreeBin的代码块是同步的，进入同步代码块之后，再次验证table中index位置元素是否被修改过。

1、根据table中index位置的Node链表，重新生成一个hd为头结点的TreeNode链表。

2、根据TreeNode链表，生成TreeBin树结构，并把树结构的root节点写到table的index位置上。

### 十、问题

##### 1.JDK8/JDK7 ConcurrentHashMap实现比较

参考[占小狼——谈谈ConcurrentHashMap1.7和1.8的不同实现](https://www.jianshu.com/p/e694f1e868ec)

角度：

- 数据结构
- ConcurrentHashMap初始化
- put操作逻辑
- size()逻辑

##### 2.为什么将链表拆成两份的时候，0 在低位，1 在高位？

参考[莫那一鲁道——ConcurrentHashMap 扩容分析拾遗](https://www.jianshu.com/p/dc69edc17b32)

### 参考文献

- [JDK ConcurrenHashMap doc](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ConcurrentHashMap.html)
- [深入浅出ConcurrentHashMap1.8](https://www.jianshu.com/p/c0642afe03e0)
- [谈谈ConcurrentHashMap1.7和1.8的不同实现](https://www.jianshu.com/p/e694f1e868ec)
- [深入分析ConcurrentHashMap1.8的扩容实现](https://www.jianshu.com/p/f6730d5784ad)
- [ConcurrentHashMap的红黑树实现分析(待看TODO)](https://www.jianshu.com/p/23b84ba9a498)
- [并发编程之 ConcurrentHashMap（JDK 1.8） putVal 源码分析](https://www.jianshu.com/p/77fda250bddf)
- [并发编程——ConcurrentHashMap#helpTransfer() 分析](https://www.jianshu.com/p/39b747c99d32)
- [并发编程——ConcurrentHashMap#transfer() 扩容逐行分析](https://www.jianshu.com/p/2829fe36a8dd)
- [ConcurrentHashMap 扩容分析拾遗](https://www.jianshu.com/p/dc69edc17b32)
- [并发编程——ConcurrentHashMap#addCount() 分析](https://www.jianshu.com/p/749d1b8db066)
- [ConcurrentHashMap 源码阅读小结](https://www.jianshu.com/p/29d8e66bc3bf)
- [HashMap源码解析](https://xuanjian1992.top/2019/03/31/HashMap源码解析/)
- [LongAdder源码分析](https://xuanjian1992.top/2019/02/22/LongAdder源码分析/)
- [并发包中ThreadLocalRandom类原理剖析](https://www.jianshu.com/p/9c2198586f9b)