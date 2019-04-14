---
layout:     post
title:      WeakHashMap源码分析(JDK 8)
subtitle:   WeakHashMap
date:       2019-04-14
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JCF
---

# WeakHashMap源码分析

### 一、WeakHashMap简介

```java
public class WeakHashMap<K,V>
    extends AbstractMap<K,V>
    implements Map<K,V> {
    ...
}
```

WeakHashMap继承于AbstractMap，实现了Map接口。和HashMap一样，WeakHashMap也是一个散列表，它存储的内容也是键值对(key-value)映射，而且键和值都可以是null。WeakHashMap比HashMap多了一个引用队列：

```java
// Reference queue for cleared WeakEntries
private final ReferenceQueue<Object> queue = new ReferenceQueue<>();
```

对比JDK8 WeakHashMap和JDK7 HashMap的源码，可以发现WeakHashMap中方法的实现方式基本和JDK7 HashMap的一样，注意“基本”两个字，除了没有实现Cloneable和Serializable这两个标记接口，最大的区别在于在于expungeStaleEntries()这个方法，这个是整个WeakHashMap的精髓，稍后详细分析。

WeakHashMap使用弱引用(**WeakReference**)作为内部数据的存储方案。准确地说，WeakHashMap的键是“弱键”。即在WeakHashMap中，当某个键不再正常使用时，会从WeakHashMap中被自动移除。更精确地说，**对于一个给定的键，其映射的存在并不阻止垃圾回收器对该键的回收，这就使该键成为可终止的，被终止，然后被回收。某个键被终止时，它对应的键值对也就从映射中有效地移除了(expungeStaleEntries()方法)**。

### 二、WeakHashMap构造函数

- 默认构造函数，使用默认初始化容量：16，默认负载因子：0.75f——WeakHashMap()

- 指定“容量大小”的构造函数，使用默认负载因子：0.75f——WeakHashMap(int capacity)

- 指定“容量大小”和“负载因子”的构造函数——WeakHashMap(int capacity, float loadFactor)

- 参数为“Map”的构造函数——WeakHashMap(Map map)

### 三、WeakHashMap的属性

- Entry[] table
  table是一个Entry数组，而Entry实际上就是一个单向链表。哈希表的“key-value键值对”都是存储在Entry数组中的。

- private int size
  size是Hashtable的大小，它是Hashtable保存的键值对的数量。

- private int threshold
  threshold是阈值，用于判断是否需要调整容量。threshold=“容量*负载因子”。

- private final float loadFactor
  loadFactor就是负载因子，它是衡量哈希表能放多满的数值。

- int modCount
  modCount是用来实现fail-fast机制的，记录哈希表被结构性修改的次数。

- **private final ReferenceQueue queue = new ReferenceQueue()(重点)**

  引用队列：当弱引用所引用的对象Key被垃圾回收器回收时，该弱引用(Entry)将被垃圾回收器添加到与之关联的引用队列中。

### 四、关键实现(Entry)

```java
// Entry只引用V value，不引用K key，防止强引用
private static class Entry<K,V> extends WeakReference<Object> implements Map.Entry<K,V> {
  	// 无Key引用
    V value;
    final int hash;
    Entry<K,V> next;

    // Creates new entry.
    Entry(Object key, V value,
          ReferenceQueue<Object> queue,
          int hash, Entry<K,V> next) {
        super(key, queue); // key作为WeakReference指向的引用
        this.value = value;
        this.hash  = hash;
        this.next  = next;
    }
		// 其余代码省略
}
```

**Entry继承自WeakReference，同时初始化时将key传递给WeakReference(有关Reference、ReferenceQueue，参考文末文献)，因此只要key不被其他引用，在GC的时候，整个Entry就会被放入引用队列中。**

> 如果一个对象只具有弱引用，那就类似于可有可无的生活用品。只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程， 因此不一定会很快发现那些只具有弱引用的对象。 弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

通过上面的分析，可知存储在WeakHashMap中的key随时都会面临被回收的风险，因此每次查询WeakHashMap时，都要确认当前WeakHashMap是否已经有key被回收了。当key被回收时，引用这个key的Entry对象就会被添加到引用队列中去，所以只要查询引用队列是否有Entry对象，就可以确认是否有key被回收了。WeakHashMap通过调用**expungeStaleEntries**方法来清除已经被回收的key所关联的Entry对象(通过get()、put()、size()调用expungeStaleEntries)。

```java
// 移除key被GC回收的Entry
private void expungeStaleEntries() {
  	// 从引用队列获取Entry，获取之后将对应Entry从WeakHashMap移除
    for (Object x; (x = queue.poll()) != null; ) {
        synchronized (queue) {
            @SuppressWarnings("unchecked")
                Entry<K,V> e = (Entry<K,V>) x;
            int i = indexFor(e.hash, table.length);

            Entry<K,V> prev = table[i];
            Entry<K,V> p = prev;
            while (p != null) {
                Entry<K,V> next = p.next;
                if (p == e) {
                    if (prev == e)
                        table[i] = next;
                    else
                        prev.next = next;
                    // Must not null out e.next;
                    // stale entries may be in use by a HashIterator
                    e.value = null; // Help GC
                    size--;
                    break;
                }
                prev = p;
                p = next;
            }
        }
    }
}
```

WeakHashMap在调用`put`和`get`方法之前，都会通过getTable间接调用expungeStaleEntries方法来清除已经被回收的key所关联的Entry对象。

**如果存放在WeakHashMap中的key都存在强引用，那么WeakHashMap就会退化成HashMap。如果在系统中希望通过WeakHashMap自动清除数据，请尽量不要在系统的其他地方强引用WeakHashMap的key，否则，这些key就不会被回收，WeakHashMap也就无法正常释放它们所占用的表项。**

### 五、实例分析

##### 1.Map存储数据

如果在一个普通的HashMap中存储一些比较大的值如下：

```java
// HashMap强引用，无法释放Entry内存
private static void hashMapTest() {
  // 结果：Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
  Map<Integer, Object> map = new HashMap<>();
  for (int i = 0; i < 10000; i++) {
    Integer integer = i;
    map.put(integer, new byte[i]);
  }
}
```

运行参数：-Xmx5M 

运行结果：

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

如果将HashMap换成WeakHashMap其余都不变：

```java
// WeakHashMap key弱引用，在内存不够时，回收部分Entry内存
private static void weakHashMapTest() {
  // 结果：正常
  Map<Integer, Object> map = new WeakHashMap<>();
  for (int i = 0; i < 10000; i++) {
    Integer integer = i;
    map.put(integer, new byte[i]);
  }
}
```

运行结果：（无任何报错）

这两段代码比较可以看到WeakHashMap的功效，如果在系统中需要一张很大的Map表，Map中的表项作为缓存使用，这也意味着即使没能从该Map中取得相应的数据，系统也可以通过候选方案获取这些数据。虽然这样会消耗更多的时间，但是不影响系统的正常运行。

在这种场景下，使用WeakHashMap是最合适的。因为WeakHashMap会在系统内存范围内，保存所有表项，而一旦内存不够，在GC时，没有被引用的表项又会很快被清除掉，从而避免系统内存溢出。

这里稍微改变一下上面的代码(加了一个List):

```java
// WeakHashMap Entry的key被强引用，在内存不够时，不会释放内存
private static void weakHashMapTestWithStrongRef() {
  // 结果：Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
  Map<Integer, Object> map = new WeakHashMap<>();
  List<Integer> list = new ArrayList<>(); // 强引用，无法回收entry
  for (int i = 0; i < 10000; i++) {
    Integer integer = i;
    list.add(integer); // 强引用key
    map.put(integer, new byte[i]);
  }
}
```

运行结果：

```java
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
```

如果存放在WeakHashMap中的key都存在强引用，那么WeakHashMap就会退化成HashMap。如果在系统中希望通过WeakHashMap自动清除数据，请尽量不要在系统的其他地方强引用WeakHashMap的key，否则，这些key就不会被回收，WeakHashMap也就无法正常释放它们所占用的表项。

> 要想WeakHashMap能够释放掉被回收的key关联的value对象，要尽可能的多调用下put/size/get等操作，因为这些方法会调用expungeStaleEntries方法，expungeStaleEntries方法是关键，而如果不操作WeakHashMap，以企图WeakHashMap“自动”释放内存是不可取的，这里的“自动”是指譬如：map.put(obj, new byte[10M])；之后obj=null了，之后再也没调用过map的任何方法，那么new出来的10M空间是不会释放的。

##### 2.WeakHashMap存入key=null，value为大对象

WeakHashMap的key可以为null，那么当put一个key为null，value为一个很大对象的时候，这个很大的对象怎么采用WeakHashMap的自带功能自动释放呢？

```java
// key(null)---value(大对象)
private static void nullKeyWithBigObject() throws InterruptedException {
   Map<Object, Object> map = new WeakHashMap<>();
   map.put(null, new byte[5*1024*600]);
   int i = 1;
   while (true) {
      System.out.println();
      TimeUnit.SECONDS.sleep(2);
      System.out.println(map.size());
      System.gc();
      System.out.println("==============第" + i++ + "次GC结束=============");
   }
}
```

运行参数：-Xmx5M -XX:+PrintGCDetails 
运行结果：

```java
1
[GC (System.gc()) [PSYoungGen: 1269K->496K(1536K)] 4433K->3731K(5632K), 0.0010877 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 496K->0K(1536K)] [ParOldGen: 3235K->3551K(4096K)] 3731K->3551K(5632K), [Metaspace: 3227K->3227K(1056768K)], 0.0072430 secs] [Times: user=0.02 sys=0.00, real=0.00 secs] 
==============第1次GC结束=============

1
[GC (System.gc()) [PSYoungGen: 40K->32K(1536K)] 3592K->3583K(5632K), 0.0007576 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 32K->0K(1536K)] [ParOldGen: 3551K->3486K(4096K)] 3583K->3486K(5632K), [Metaspace: 3228K->3228K(1056768K)], 0.0061292 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
==============第2次GC结束=============

1
[GC (System.gc()) [PSYoungGen: 20K->0K(1536K)] 3506K->3486K(5632K), 0.0008321 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 0K->0K(1536K)] [ParOldGen: 3486K->3486K(4096K)] 3486K->3486K(5632K), [Metaspace: 3228K->3228K(1056768K)], 0.0048328 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
==============第3次GC结束=============

1
[GC (System.gc()) [PSYoungGen: 20K->0K(1536K)] 3506K->3486K(5632K), 0.0008283 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 0K->0K(1536K)] [ParOldGen: 3486K->3486K(4096K)] 3486K->3486K(5632K), [Metaspace: 3228K->3228K(1056768K)], 0.0087827 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
==============第4次GC结束=============
```

可以看到在`map.put(null, new byte[5*1024*600]);`之后，相应的内存一直没有得到释放。

通过显式的调用`map.remove(null)`可以将内存释放掉，如下代码所示：

```java
// WeakHashMap (key(null)---value(大对象))，显式调用remove(null)方法
private static void nullKeyWithBigObject2() throws InterruptedException {
   Map<Integer,Object> map = new WeakHashMap<>();
   System.gc();
   System.out.println("===========gc:1=============");
   map.put(null,new byte[5*1024*600]);
   TimeUnit.SECONDS.sleep(5);
   System.gc();
   System.out.println("===========gc:2=============");
   TimeUnit.SECONDS.sleep(5);
   System.gc();
   System.out.println("===========gc:3=============");
   map.remove(null);
   TimeUnit.SECONDS.sleep(5);
   System.gc();
   System.out.println("===========gc:4=============");

}
```

运行参数：-Xmx5M -XX:+PrintGCDetails 
运行结果：

```java
[GC (Allocation Failure) [PSYoungGen: 1024K->496K(1536K)] 1024K->520K(5632K), 0.0006676 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (Allocation Failure) [PSYoungGen: 1508K->512K(1536K)] 1532K->637K(5632K), 0.0007752 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (System.gc()) [PSYoungGen: 853K->512K(1536K)] 979K->685K(5632K), 0.0005943 secs] [Times: user=0.01 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 512K->0K(1536K)] [ParOldGen: 173K->527K(4096K)] 685K->527K(5632K), [Metaspace: 3317K->3317K(1056768K)], 0.0045081 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
===========gc:1=============
[GC (Allocation Failure) [PSYoungGen: 1024K->480K(1536K)] 4551K->4015K(5632K), 0.0005182 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[GC (System.gc()) [PSYoungGen: 528K->384K(1536K)] 4063K->3919K(5632K), 0.0014862 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 384K->0K(1536K)] [ParOldGen: 3535K->3809K(4096K)] 3919K->3809K(5632K), [Metaspace: 3847K->3847K(1056768K)], 0.0072400 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
===========gc:2=============
[GC (System.gc()) [PSYoungGen: 19K->32K(1536K)] 3829K->3841K(5632K), 0.0008839 secs] [Times: user=0.00 sys=0.00, real=0.01 secs] 
[Full GC (System.gc()) [PSYoungGen: 32K->0K(1536K)] [ParOldGen: 3809K->3809K(4096K)] 3841K->3809K(5632K), [Metaspace: 3847K->3847K(1056768K)], 0.0035518 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
===========gc:3=============
[GC (System.gc()) [PSYoungGen: 19K->32K(1536K)] 3829K->3841K(5632K), 0.0006995 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
[Full GC (System.gc()) [PSYoungGen: 32K->0K(1536K)] [ParOldGen: 3809K->809K(4096K)] 3841K->809K(5632K), [Metaspace: 3847K->3847K(1056768K)], 0.0063985 secs] [Times: user=0.01 sys=0.00, real=0.01 secs] 
===========gc:4=============
Heap
 PSYoungGen      total 1536K, used 30K [0x00000007bfe00000, 0x00000007c0000000, 0x00000007c0000000)
  eden space 1024K, 2% used [0x00000007bfe00000,0x00000007bfe07a60,0x00000007bff00000)
  from space 512K, 0% used [0x00000007bff00000,0x00000007bff00000,0x00000007bff80000)
  to   space 512K, 0% used [0x00000007bff80000,0x00000007bff80000,0x00000007c0000000)
 ParOldGen       total 4096K, used 809K [0x00000007bfa00000, 0x00000007bfe00000, 0x00000007bfe00000)
  object space 4096K, 19% used [0x00000007bfa00000,0x00000007bfaca560,0x00000007bfe00000)
 Metaspace       used 3853K, capacity 4572K, committed 4864K, reserved 1056768K
  class space    used 435K, capacity 460K, committed 512K, reserved 1048576K
```

分析：

- 在WeakHashMap中，put的key为null时，实际放入的是NULL_KEY(调用maskNull方法进行了转换)，即：private static final Object NULL_KEY = new Object()，是一个静态常量。 

  ```java
  private static final Object NULL_KEY = new Object();
  
  // 若key=null，将key转换为NULL_KEY
  private static Object maskNull(Object key) {
      return (key == null) ? NULL_KEY : key;
  }
  
  // 将NULL_KEY转换为null
  static Object unmaskNull(Object key) {
      return (key == NULL_KEY) ? null : key;
  }
  ```

- 在WeakHashMap中，由于传给WeakReference的只有key和queue，即gc只会在key可以回收时将整个Entry引用放入引用队列，对应WeakHashMap中的Entry的移除，则是在expungeStaleEntries这个私有方法中进行的。 

- 而static的就不在gc之列，所以key也就不会被gc，所以key对应的Entry也不会从WeakHashMap删除，即不会被回收。

- 通过调用remove方法显式删除键值对后，最终table[k]设为null，此时大对象游离，所以被回收。