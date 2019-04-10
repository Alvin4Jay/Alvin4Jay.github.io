---
layout:     post
title:      LinkedHashMap源码分析(JDK 8)
subtitle:   LinkedHashMap
date:       2019-04-10
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JCF
---

# LinkedHashMap源码分析(JDK 8)

HashMap和双向链表合二为一即是LinkedHashMap。所谓LinkedHashMap，其落脚点在HashMap，因此更准确地说，它是一个将所有Entry节点链入一个双向链表的HashMap。由于LinkedHashMap是HashMap的子类，所以LinkedHashMap自然会拥有HashMap的所有特性。比如，LinkedHashMap的元素存取过程基本与HashMap基本类似，只是在细节实现上稍有不同。当然，这是由LinkedHashMap本身的特性所决定的，因为它额外维护了一个双向链表用于保持迭代顺序。此外，LinkedHashMap可以很好的支持LRU算法。

### 一、LinkedHashMap概述

HashMap 是 Java Collection Framework 的重要成员，也是Map族中最为常用的一种。不过遗憾的是，**HashMap是无序的，也就是说，迭代HashMap所得到的元素顺序并不是它们最初放置到HashMap的顺序**。HashMap的这一缺点往往会造成诸多不便，因为在有些场景中，我们的确需要用到一个可以保持插入顺序的Map。庆幸的是，JDK为我们解决了这个问题，它为HashMap提供了一个子类 —— LinkedHashMap。虽然LinkedHashMap增加了时间和空间上的开销，但是它通过维护一个额外的双向链表保证了迭代顺序。特别地，该迭代顺序可以是插入顺序，也可以是访问顺序。因此，根据链表中元素的顺序可以将LinkedHashMap分为：**保持插入顺序的LinkedHashMap 和 保持访问顺序的LinkedHashMap**，其中LinkedHashMap的默认实现是按插入顺序排序的。

本质上，HashMap和双向链表合二为一即是LinkedHashMap。所谓LinkedHashMap，其落脚点在HashMap，因此更准确地说，它是一个将所有Entry节点链入一个双向链表的HashMap。在LinkedHashMapMap中，所有put进来的Entry都保存在哈希表中，但由于它又额外定义了一个以head为头结点、tail为尾节点的双向链表，因此对于每次put进来Entry，除了将其保存到哈希表中对应的位置上之外，还会将其插入到双向链表的尾部。

更直观地，下图很好地还原了LinkedHashMap的原貌：HashMap和双向链表的密切配合和分工合作造就了LinkedHashMap。特别需要注意的是，**next用于维护HashMap各个桶中的Entry链，before、after用于维护LinkedHashMap的双向链表**，虽然它们的作用对象都是Entry，但是各自分离，是两码事儿。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/LinkedHashMap%E7%BB%93%E6%9E%84.jpg?x-oss-process=style/markdown-pic)

由于LinkedHashMap是HashMap的子类，所以LinkedHashMap自然会拥有HashMap的所有特性。比如，LinkedHashMap也最多只允许一条Entry的键为Null(多条会覆盖)，但允许多条Entry的值为Null。LinkedHashMap 也是 Map 的一个非同步的实现。此外，LinkedHashMap还可以用来实现LRU (Least recently used, 最近最少使用)算法，这个问题会在下文谈到。

### 二、LinkedHashMap在JDK中的定义

##### 1.类结构

LinkedHashMap继承于HashMap，其在JDK中的定义为：

```java
public class LinkedHashMap<K,V>
    extends HashMap<K,V>
    implements Map<K,V>
{
	...    
}
```

##### 2.成员变量

与HashMap相比，LinkedHashMap增加了三个属性用于保证迭代顺序，分别是 **双向链表头结点header** 、**尾节点tail** 和 **标志位accessOrder** (值为true时，表示按照访问顺序迭代；值为false时，表示按照插入顺序迭代)。

```java
// 双向链表头结点(最老)
transient LinkedHashMap.Entry<K,V> head;

// 双向链表尾结点(最年轻)
transient LinkedHashMap.Entry<K,V> tail;

// true表示按照访问顺序迭代，false时表示按照插入顺序
final boolean accessOrder;
```

##### 3.重要方法

以下是LinkedHashMap在put、remove、get操作过程中用到的三个回调方法实现。

```java
// 节点e从哈希表移除之后(removeNode)的回调
void afterNodeRemoval(Node<K,V> e) { // unlink 将节点e从双向链表中移除
    LinkedHashMap.Entry<K,V> p =
        (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
    p.before = p.after = null;
    if (b == null)
        head = a;
    else
        b.after = a;
    if (a == null)
        tail = b;
    else
        a.before = b;
}

// 节点插入之后(putVal)的回调
void afterNodeInsertion(boolean evict) { // possibly remove eldest 可能移除最老的节点
    LinkedHashMap.Entry<K,V> first;
  	// removeEldestEntry(first)若为true，则移除最老的节点(头结点)
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true); // 将节点head从哈希表和双向链表删除
    }
}

// 节点访问之后(比如get)的回调
void afterNodeAccess(Node<K,V> e) { // move node to last 将节点移到双向链表的尾部
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) { // 首先accessOrder需要为true，即按照访问顺序排序
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

##### 4.节点类Entry<K, V>

LinkedHashMap采用的hash算法和HashMap相同，但是它重新定义了Entry。LinkedHashMap中的Entry增加了两个指针 before 和 after，它们分别用于维护双向链接列表。特别需要注意的是，**next用于维护HashMap各个桶中Entry的连接顺序，before、after用于维护Entry插入的先后顺序**，源代码如下：

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after; // 双向链表中节点的前后指针
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

##### 5.构造函数

LinkedHashMap 一共提供了五个构造函数，它们都是在HashMap的构造函数的基础上实现的，分别如下：

```java
// 构造一个指定初始容量和指定负载因子的空 LinkedHashMap，按照插入顺序迭代
public LinkedHashMap(int initialCapacity, float loadFactor) {
    super(initialCapacity, loadFactor);
    accessOrder = false;
}

// 构造一个指定初始容量和默认负载因子 (0.75)的空 LinkedHashMap，按照插入顺序迭代
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

// 构造一个具有 默认初始容量 (16)和默认负载因子(0.75)的空 LinkedHashMap，按照插入顺序迭代
public LinkedHashMap() {
    super();
    accessOrder = false;
}

// 构造一个与指定 Map 具有相同映射的 LinkedHashMap，按照插入顺序迭代
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super();
    accessOrder = false;
    putMapEntries(m, false);
}

// 构造一个指定初始容量和指定负载因子的具有指定迭代顺序的LinkedHashMap
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```

### 三、LinkedHashMap的数据结构

LinkedHashMap = HashMap + 双向链表，也就是说，HashMap和双向链表合二为一即是LinkedHashMap。也可以这样理解，LinkedHashMap在HashMap的基础上，给HashMap的任意两个节点间加了两条连线(before指针和after指针)，使这些节点形成一个双向链表。在LinkedHashMapMap中，所有put进来的Entry都保存在HashMap中，但由于它又额外定义了一个以head为头结点、tail为尾节点的双向链表，因此对于每次put进来Entry还会将其插入到双向链表的尾部。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/java%E6%BA%90%E7%A0%81/LinkedHashMap%E7%BB%93%E6%9E%84.jpg?x-oss-process=style/markdown-pic)

### 四、LinkedHashMap的put、get与remove

##### 1.put(K, V)

LinkedHashMap没有对 put(K, V) 方法进行任何直接的修改，完全继承了HashMap的 put(K, V) 方法。

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

该方法中，LinkedHashMap对newNode、afterNodeAccess、afterNodeInsertion方法进行了重写，后两个方法前面已经介绍，newNode方法重写如下：

```java
// 新建一个LinkedHashMap.Entry，链接到双向链表尾部
Node<K,V> newNode(int hash, K key, V value, Node<K,V> e) {
    LinkedHashMap.Entry<K,V> p =
        new LinkedHashMap.Entry<K,V>(hash, key, value, e);
    linkNodeLast(p); // 链接到双向链表尾部
    return p;
}
```

因此put操作中，如果节点插入哈希表成功，则也会链接到双向链表的尾部，同时afterNodeInsertion方法会调用；如果节点已存在，则afterNodeAccess会调用。

##### 2.get(K)

LinkedHashMap中重写了HashMap中的get方法。

```java
public V get(Object key) {
    Node<K,V> e;
  	// 根据key获取对应的Entry，若没有这样的Entry，则返回null
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder) // 双向链表如果按照访问顺序排序，则将被访问的节点移到链表尾部
        afterNodeAccess(e);
    return e.value;
}
```

在LinkedHashMap的get方法中，通过HashMap中的getNode方法获取Entry对象。注意这里的afterNodeAccess方法，如果链表中元素的排序规则是按照插入的先后顺序排序的话，该方法什么也不做；如果链表中元素的排序规则是按照访问的先后顺序排序的话，则将e移到链表的末尾处，本文后面还会详细阐述这个问题。

##### 3.remove(K)

LinkedHashMap的remove方法，继承自HashMap的remove方法。

```java
final Node<K,V> removeNode(int hash, Object key, Object value,
                           boolean matchValue, boolean movable) {
    Node<K,V>[] tab; Node<K,V> p; int n, index;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (p = tab[index = (n - 1) & hash]) != null) {
        Node<K,V> node = null, e; K k; V v;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            node = p;
        else if ((e = p.next) != null) {
            if (p instanceof TreeNode)
                node = ((TreeNode<K,V>)p).getTreeNode(hash, key);
            else {
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key ||
                         (key != null && key.equals(k)))) {
                        node = e;
                        break;
                    }
                    p = e;
                } while ((e = e.next) != null);
            }
        }
        if (node != null && (!matchValue || (v = node.value) == value ||
                             (value != null && value.equals(v)))) {
            if (node instanceof TreeNode)
                ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
            else if (node == p)
                tab[index] = node.next;
            else
                p.next = node.next;
            ++modCount;
            --size;
            afterNodeRemoval(node);
            return node;
        }
    }
    return null;
}
```

可以看到，节点在从哈希表删除之后会调用afterNodeRemoval方法，将节点从双向链表移除。

##### 4.总结

LinkedHashMap 的put、get、remove过程基本与HashMap基本类似，只是在细节实现上稍有不同，这是由LinkedHashMap本身的特性所决定的，因为它要额外维护一个双向链表用于保持迭代(排序)顺序。

### 五、LinkedHashMap与LRU(最近最少使用)算法

LinkedHashMap区别于HashMap最大的一个不同点是，前者是有序的，而后者是无序的。为此，LinkedHashMap增加了三个属性用于保证顺序，分别是双向链表头结点header、尾节点tail和标志位accessOrder。header、tail是LinkedHashMap所维护的双向链表的头结点、尾节点，而accessOrder用于决定具体的迭代顺序。实际上，accessOrder标志位的作用并不像我们描述的这样简单。

当accessOrder标志位为true时，表示双向链表中的元素按照访问的先后顺序排列，可以看到，虽然Entry插入链表的顺序依然是按照其put到LinkedHashMap中的顺序，但put和get方法均有调用afterNodeAccess方法（put方法在key相同时会调用）。afterNodeAccess方法判断accessOrder是否为true，如果是，则将当前访问的Entry（put进来的Entry或get出来的Entry）移到双向链表的尾部（key不相同时，put新Entry时，会调用putVal，它会调用newNode方法，该方法同样将新插入的元素放入到双向链表的尾部，既符合插入的先后顺序，又符合访问的先后顺序，因为这时该Entry也被访问了）；当标志位accessOrder的值为false时，表示双向链表中的元素按照Entry插入LinkedHashMap到中的先后顺序排序，即每次put到LinkedHashMap中的Entry都放在双向链表的尾部，这样遍历双向链表时，Entry的输出顺序便和插入的顺序一致，这也是默认的双向链表的存储顺序。因此，当标志位accessOrder的值为false时，虽然也会调用afterNodeAccess方法，但不做任何操作。

前面介绍的LinkedHashMap的五种构造方法，前四个构造方法都将accessOrder设为false，说明默认是按照插入顺序排序的；而第五个构造方法可以自定义传入的accessOrder的值，因此可以指定双向链表中元素的排序规则。特别地，**当要用LinkedHashMap实现LRU算法时，就需要调用该构造方法并将accessOrder置为true**。

##### 1.put操作与标志位accessOrder

在put操作中，如果节点插入哈希表成功，则也会链接到双向链表的尾部，同时afterNodeInsertion方法会调用；如果节点已存在，则afterNodeAccess会调用。

afterNodeAccess方法：

```java
// 节点访问之后(比如get)的回调
void afterNodeAccess(Node<K,V> e) { // move node to last 将节点移到双向链表的尾部
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) { // 首先accessOrder需要为true，即按照访问顺序排序
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

LinkedHashMap重写了HashMap中的afterNodeAccess方法（HashMap中该方法为空），当调用父类的put方法时，在发现key已经存在时，会调用该方法；当调用自己的get方法时，也会调用到该方法。该方法提供了LRU算法的实现，它将最近使用的Entry放到双向循环链表的尾部。也就是说，当afterNodeAccess为true时，get方法和put方法都会调用afterNodeAccess方法使得最近使用的Entry移到双向链表的末尾；当accessOrder为默认值false时，从源码中可以看出afterNodeAccess方法什么也不会做。

afterNodeInsertion方法：

```java
// 节点插入之后(putVal)的回调
void afterNodeInsertion(boolean evict) { // possibly remove eldest 可能移除最老的节点
    LinkedHashMap.Entry<K,V> first;
  	// removeEldestEntry(first)若为true，则移除最老的节点(头结点)
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true); // 将节点head从哈希表和双向链表删除
    }
}
```

如果节点插入成功，afterNodeInsertion方法会调用。其中调用了removeEldestEntry方法：

```java
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

该方法是用来被重写的，一般地，如果用LinkedHashmap实现LRU算法，就要重写该方法。比如可以将该方法覆写为如果设定的内存已满，则返回true，这样当再次向LinkedHashMap中put Entry时，在调用的putVal(afterNodeInsertion)方法中便会将近期最少使用的节点删除掉(header节点)。

##### 2.get操作与标志位accessOrder

在LinkedHashMap中进行读取操作时，一样也会调用afterNodeAccess方法。

```java
public V get(Object key) {
    Node<K,V> e;
  	// 根据key获取对应的Entry，若没有这样的Entry，则返回null
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder) // 双向链表如果按照访问顺序排序，则将被访问的节点移到链表尾部
        afterNodeAccess(e);
    return e.value;
}
```

##### 3.LinkedListMap与LRU小结

使用LinkedHashMap实现LRU的必要前提是将accessOrder标志位设为true以便开启按访问顺序排序的模式。无论是put方法还是get方法，都会导致目标Entry成为最近访问的Entry，因此就把该Entry加入到了双向链表的末尾：get方法通过调用afterNodeAccess方法来实现；put方法在覆盖已有key的情况下，也是通过调用afterNodeAccess方法来实现，在插入新的Entry时，则是通过newNode方法中的linkNodeLast方法来实现。这样，便把最近使用的Entry放入到了双向链表的后面。多次操作后，双向链表前面的Entry便是最近没有使用的，这样当节点个数满的时候，删除最前面的Entry(head)即可，因为它就是最近最少使用的Entry。

### 六、使用LinkedHashMap实现LRU算法

下面使用LinkedHashMap实现一个符合LRU算法的数据结构，该结构最多可以缓存6个元素，但元素多于6个时，会自动删除最近最久没有被使用的元素，如下所示：

```java
// 使用LinkedHashMap实现LRU算法    
public class LRU<K,V> extends LinkedHashMap<K, V> implements Map<K, V>{
    private static final long serialVersionUID = 1L;

    public LRU(int initialCapacity, float loadFactor, boolean accessOrder) {
        super(initialCapacity, loadFactor, accessOrder);
    }

    // 重写LinkedHashMap中的removeEldestEntry方法，当LRU中元素多余6个时，删除最不经常使用的元素
    @Override
    protected boolean removeEldestEntry(java.util.Map.Entry<K, V> eldest) {
        if(size() > 6){
            return true;
        }
        return false;
    }

    public static void main(String[] args) {
        LRU<Character, Integer> lru = new LRU<Character, Integer>(16, 0.75f, true);

        String s = "abcdefghijkl";
        for (int i = 0; i < s.length(); i++) {
            lru.put(s.charAt(i), i);
        }
        System.out.println("LRU中key为h的Entry的值为： " + lru.get('h'));
        System.out.println("LRU的大小 ：" + lru.size());
        System.out.println("LRU ：" + lru);
    }
}

// 输出
LRU中key为h的Entry的值为： 7
LRU的大小 ：6
LRU ：{g=6, i=8, j=9, k=10, l=11, h=7}
```

### 七、迭代器

```java
// 迭代器基类
abstract class LinkedHashIterator {
    LinkedHashMap.Entry<K,V> next; // 下一个返回的节点
    LinkedHashMap.Entry<K,V> current; // 当前已经返回的节点
    int expectedModCount; // 结构修改计数

    LinkedHashIterator() {
        next = head;
        expectedModCount = modCount;
        current = null;
    }

    public final boolean hasNext() {
        return next != null;
    }

  	// 下一个节点
    final LinkedHashMap.Entry<K,V> nextNode() {
        LinkedHashMap.Entry<K,V> e = next;
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        if (e == null)
            throw new NoSuchElementException();
        current = e;
        next = e.after; // 按照双向链表的顺序迭代
        return e;
    }

    public final void remove() {
        Node<K,V> p = current;
        if (p == null)
            throw new IllegalStateException();
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
        current = null;
        K key = p.key;
        removeNode(hash(key), key, null, false, false); // 移除节点
        expectedModCount = modCount;
    }
}

final class LinkedKeyIterator extends LinkedHashIterator
    implements Iterator<K> {
    public final K next() { return nextNode().getKey(); }
}

final class LinkedValueIterator extends LinkedHashIterator
    implements Iterator<V> {
    public final V next() { return nextNode().value; }
}

final class LinkedEntryIterator extends LinkedHashIterator
    implements Iterator<Map.Entry<K,V>> {
    public final Map.Entry<K,V> next() { return nextNode(); }
}
```

### 参考

- [Map 综述（二）：彻头彻尾理解 LinkedHashMap](<https://blog.csdn.net/justloveyou_/article/details/71713781>)
- [LinkedHashMap就这么简单【源码剖析】](<https://zhuanlan.zhihu.com/p/35559602>)