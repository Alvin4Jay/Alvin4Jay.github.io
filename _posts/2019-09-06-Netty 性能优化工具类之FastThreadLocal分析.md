---
layout:     post
title:      Netty FastThreadLocal分析
subtitle:   FastThreadLocalThread/InternalThreadLocalMap
date:       2019-09-06
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Netty
---

# Netty FastThreadLocal分析

Netty中FastThreadLocal用来代替ThreadLocal存放**线程本地变量**，从FastThreadLocalThread类型的线程中访问本地变量时，比使用ThreadLocal会有更好的性能。

FastThreadLocal使用InternalThreadLocalMap存放实际的数据。和ThreadLocal实现方式类似，FastThreadLocalThread中有一个InternalThreadLocalMap类型的字段threadLocalMap，这样一个线程对应一个InternalThreadLocalMap实例，该线程下所有的线程本地变量都会放在threadLocalMap中的数组indexedVariables中。下面详细分析FastThreadLocal的实现机制。

### 一、FastThreadLocal的创建

```java
public class FastThreadLocalTest {
    private static FastThreadLocal<Object> threadLocal = new FastThreadLocal<Object>() {
        @Override
        protected Object initialValue() throws Exception {
            return new Object();
        }
    };

    public static void main(String[] args) {
        new Thread(() -> {
            Object o = threadLocal.get();
            System.out.println(o);
        }).start();

        new Thread(() -> {
            Object o = threadLocal.get();
            System.out.println(o);
        }).start();
    }
}
```

如上所示是使用FastThreadLocal的例子，通过声明静态的FastThreadLocal实例，然后在两个线程中分别通过`threadLocal.get()`获取各自的线程本地变量。下面分析FastThreadLocal的创建过程。

```java
private final int index; // 每个FastThreadLocal实例对应一个唯一的index

public FastThreadLocal() {
    index = InternalThreadLocalMap.nextVariableIndex();
}
```

从FastThreadLocal的构造器可以看出，FastThreadLocal初始化时得到一个index变量，其值通过`InternalThreadLocalMap.nextVariableIndex()`获取:

```java
// UnpaddedInternalThreadLocalMap
static final AtomicInteger nextIndex = new AtomicInteger();

// InternalThreadLocalMap
public static int nextVariableIndex() {
    int index = nextIndex.getAndIncrement();
    if (index < 0) {
        nextIndex.decrementAndGet();
        throw new IllegalStateException("too many thread-local indexed variables");
    }
    return index;
}
```

可知，每当FastThreadLocal初始化时都会初始化index变量，并且在JVM进程中每个FastThreadLocal实例对应一个唯一的index。因此，下面将看到，每当一个FastThreadLocalThread线程通过FastThreadLocal获取线程本地变量时，都是通过FastThreadLocal对应的index变量值，在FastThreadLocalThread的threadLocalMap中的数组indexedVariables中以该index为索引得到的。

### 二、FastThreadLocal.get()实现

##### 1.获取InternalThreadLocalMap

```java
// FastThreadLocal
public final V get() {
    return get(InternalThreadLocalMap.get());
}

// InternalThreadLocalMap
public static InternalThreadLocalMap get() {
    Thread thread = Thread.currentThread();
    if (thread instanceof FastThreadLocalThread) {
        return fastGet((FastThreadLocalThread) thread);
    } else {
        return slowGet();
    }
}

private static InternalThreadLocalMap fastGet(FastThreadLocalThread thread) {
    InternalThreadLocalMap threadLocalMap = thread.threadLocalMap();
    if (threadLocalMap == null) {
        thread.setThreadLocalMap(threadLocalMap = new InternalThreadLocalMap());
    }
    return threadLocalMap;
}

private static InternalThreadLocalMap slowGet() {
    ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = UnpaddedInternalThreadLocalMap.slowThreadLocalMap;
    InternalThreadLocalMap ret = slowThreadLocalMap.get();
    if (ret == null) {
        ret = new InternalThreadLocalMap();
        slowThreadLocalMap.set(ret);
    }
    return ret;
}

// UnpaddedInternalThreadLocalMap
static final ThreadLocal<InternalThreadLocalMap> slowThreadLocalMap = new ThreadLocal<InternalThreadLocalMap>();
```

可以看到，调用`FastThreadLocal.get()`时，首先获取当前线程实例对应的InternalThreadLocalMap。如果当前线程是FastThreadLocalThread实例，则通过fastGet()方法获取，否则通过slowGet()方法获取。fastGet()方法中直接获取FastThreadLocalThread的threadLocalMap变量，如果为null，则初始化。如果调用的是slowGet()，说明当前线程是普通的Thread，则通过JDK ThreadLocal `slowThreadLocalMap`变量获取与线程绑定的InternalThreadLocalMap。

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/FastThreadLocal-1.png)

##### 2.通过index索引获取值

在获取到线程的InternalThreadLocalMap实例之后，调用`get(InternalThreadLocalMap threadLocalMap)`方法获取线程本地变量。

```java
public final V get(InternalThreadLocalMap threadLocalMap) {
    Object v = threadLocalMap.indexedVariable(index);
    if (v != InternalThreadLocalMap.UNSET) {
        return (V) v;
    }

    return initialize(threadLocalMap);
}
```

get(InternalThreadLocalMap)方法中通过InternalThreadLocalMap.indexedVariable()方法获取线程本地变量：

```java
public Object indexedVariable(int index) {
    Object[] lookup = indexedVariables;
    return index < lookup.length? lookup[index] : UNSET;
}
```

`indexedVariables`是线程的InternalThreadLocalMap实例中的一个线程本地变量数组，**数组索引是FastThreadLocal的index变量值**。因此，通过indexedVariable()方法就可以获取到线程的InternalThreadLocalMap中某个FastThreadLocal对应的线程本地变量值。如下图所示：

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/netty/FastThreadLocal-2.png)



如果indexedVariable()方法返回的值不为UNSET，则直接返回该值。如果indexedVariable()方法返回UNSET，表明InternalThreadLocalMap中该index位置的线程本地变量未初始化，则调用initialize()方法进行初始化。

##### 3.初始化

```java
public final V get(InternalThreadLocalMap threadLocalMap) {
    Object v = threadLocalMap.indexedVariable(index);
    if (v != InternalThreadLocalMap.UNSET) {
        return (V) v;
    }

    return initialize(threadLocalMap);
}
```

```java
private V initialize(InternalThreadLocalMap threadLocalMap) {
    V v = null;
    try {
        v = initialValue(); // 获取初始值
    } catch (Exception e) {
        PlatformDependent.throwException(e);
    }

    threadLocalMap.setIndexedVariable(index, v);
    addToVariablesToRemove(threadLocalMap, this);
    return v;
}
protected V initialValue() throws Exception {
    return null;
}

```

initialize()方法中调用initialValue()方法获取初始化值，该方法可以在实例化FastThreadLocal时重写。在得到初始值v之后，将该值绑定到线程InternalThreadLocalMap的`indexedVariables`数组中。

```java
public boolean setIndexedVariable(int index, Object value) {
    Object[] lookup = indexedVariables;
    if (index < lookup.length) {
        Object oldValue = lookup[index];
        lookup[index] = value;
        return oldValue == UNSET;
    } else {
        expandIndexedVariableTableAndSet(index, value); // 扩容
        return true;
    }
}
// 扩容
private void expandIndexedVariableTableAndSet(int index, Object value) {
    Object[] oldArray = indexedVariables;
    final int oldCapacity = oldArray.length;

    // 找到大于等于index的2的幂次
    int newCapacity = index;
    newCapacity |= newCapacity >>>  1;
    newCapacity |= newCapacity >>>  2;
    newCapacity |= newCapacity >>>  4;
    newCapacity |= newCapacity >>>  8;
    newCapacity |= newCapacity >>> 16;
    newCapacity ++;

    Object[] newArray = Arrays.copyOf(oldArray, newCapacity); // copy
    Arrays.fill(newArray, oldCapacity, newArray.length, UNSET); // 填充其余部分元素为UNSET
    newArray[index] = value;
    indexedVariables = newArray;
}
```

通过setIndexedVariable()方法可以看出，如果该数组容量不够，还会进行扩容操作。

在initialize()方法的最后，调用了`addToVariablesToRemove(threadLocalMap, this);`代码:

```java
private static final int variablesToRemoveIndex = InternalThreadLocalMap.nextVariableIndex();

private static void addToVariablesToRemove(InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {
    Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
    Set<FastThreadLocal<?>> variablesToRemove;
    if (v == InternalThreadLocalMap.UNSET || v == null) {
        variablesToRemove = Collections.newSetFromMap(new IdentityHashMap<FastThreadLocal<?>, Boolean>());
        threadLocalMap.setIndexedVariable(variablesToRemoveIndex, variablesToRemove);
    } else {
        variablesToRemove = (Set<FastThreadLocal<?>>) v;
    }

    variablesToRemove.add(variable);
}
```

addToVariablesToRemove()方法中，通过variablesToRemoveIndex索引获取当前线程InternalThreadLocalMap线程本地变量数组中对应位置的值，该值是一个`Set<FastThreadLocal>`变量，并将当前FastThreadLocal实例放入该Set中。这个`Set<FastThreadLocal>`变量主要是用来FastThreadLocalThread在清理线程本地变量缓存的时候用的，后面会分析。

到此，FastThreadLocal.get()实现分析完毕。

### 三、FastThreadLocal.set()实现

```java
public final void set(V value) {
    if (value != InternalThreadLocalMap.UNSET) {
        set(InternalThreadLocalMap.get(), value);
    } else {
        remove();
    }
}
```

##### 1.获取InternalThreadLocalMap

假设设置的值不是UNSET，则先获取当前线程的InternalThreadLocalMap实例，上面已经分析过，这里不再详述。

```java
InternalThreadLocalMap.get()
```

##### 2.通过index索引设置值

```java
public final void set(InternalThreadLocalMap threadLocalMap, V value) {
    if (value != InternalThreadLocalMap.UNSET) {
        if (threadLocalMap.setIndexedVariable(index, value)) { // 返回true，表示新创建了一个线程本地变量
            addToVariablesToRemove(threadLocalMap, this);
        }
    } else {
        remove(threadLocalMap);
    }
}
```

首先调用`threadLocalMap.setIndexedVariable(index, value)`设置线程本地变量：

```java
public boolean setIndexedVariable(int index, Object value) {
    Object[] lookup = indexedVariables;
    if (index < lookup.length) {
        Object oldValue = lookup[index];
        lookup[index] = value;
        return oldValue == UNSET;
    } else {
        expandIndexedVariableTableAndSet(index, value); // 扩容
        return true;
    }
}
```

如果该方法返回true，表示设置线程本地变量成功，并且之前未设置过该index位置处的值。然后再调用`addToVariablesToRemove(threadLocalMap, this)`代码，该方法上面已经分析过，这里不再说明。

##### 3.remove对象

如果设置线程本地变量时传入的是UNSET，则调用remove()方法删除线程本地变量。

```java
public final void remove() {
     // InternalThreadLocalMap.getIfSet(): 可能返回null
    remove(InternalThreadLocalMap.getIfSet());
}
public final void remove(InternalThreadLocalMap threadLocalMap) {
    if (threadLocalMap == null) {
        return;
    }

    Object v = threadLocalMap.removeIndexedVariable(index); // 删除线程本地变量
    removeFromVariablesToRemove(threadLocalMap, this);

    if (v != InternalThreadLocalMap.UNSET) {
        try {
            onRemoval((V) v);
        } catch (Exception e) {
            PlatformDependent.throwException(e);
        }
    }
}
```

remove(InternalThreadLocalMap)方法中调用`threadLocalMap.removeIndexedVariable(index)`删除线程本地变量:

```java
// InternalThreadLocalMap
public Object removeIndexedVariable(int index) {
    Object[] lookup = indexedVariables;
    if (index < lookup.length) {
        Object v = lookup[index];
        lookup[index] = UNSET;
        return v;
    } else {
        return UNSET;
    }
}
```

如果removeIndexedVariable()方法返回的值不是UNSET，表示删除的线程本地变量之前已设置了有效值，则调用`removeFromVariablesToRemove(threadLocalMap, this)`:

```java
private static void removeFromVariablesToRemove(
        InternalThreadLocalMap threadLocalMap, FastThreadLocal<?> variable) {

    Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);

    if (v == InternalThreadLocalMap.UNSET || v == null) {
        return;
    }

    @SuppressWarnings("unchecked")
    Set<FastThreadLocal<?>> variablesToRemove = (Set<FastThreadLocal<?>>) v;
    variablesToRemove.remove(variable);
}
```

在当前线程的InternalThreadLocalMap的`indexedVariables`数组中，取出variablesToRemoveIndex索引对应的`Set<FastThreadLocal>`集合，并删除当前的FastThreadLocal variable引用。这个`Set<FastThreadLocal>`变量主要是用来FastThreadLocalThread在清理线程本地变量缓存的时候用的，后面会分析。

在删除线程本地变量之后，将调用onRemove()方法:

```java
if (v != InternalThreadLocalMap.UNSET) {
    try {
        onRemoval((V) v);
    } catch (Exception e) {
        PlatformDependent.throwException(e);
    }
}
protected void onRemoval(@SuppressWarnings("UnusedParameters") V value) throws Exception { }
```

用户可以重写此方法，在该方法调用时做一些资源清理工作。

### 四、FastThreadLocal.removeAll()实现

在通过DefaultThreadFactory创建FastThreadLocalThread线程时，Runnbale任务被包装为了DefaultRunnableDecorator实例。

```java
// DefaultThreadFactory
public Thread newThread(Runnable r) {
    // name: nioEventLoopGroup-poolId-nextId
    // 创建的线程实体: FastThreadLocalThread
    Thread t = newThread(new DefaultRunnableDecorator(r), prefix + nextId.incrementAndGet());
    try {
        if (t.isDaemon()) {
            if (!daemon) {
                t.setDaemon(false);
            }
        } else {
            if (daemon) {
                t.setDaemon(true);
            }
        }

        if (t.getPriority() != priority) {
            t.setPriority(priority);
        }
    } catch (Exception ignored) {
        // Doesn't matter even if failed to set.
    }
    return t;
}

protected Thread newThread(Runnable r, String name) {
    return new FastThreadLocalThread(threadGroup, r, name);
}

private static final class DefaultRunnableDecorator implements Runnable {

    private final Runnable r;

    DefaultRunnableDecorator(Runnable r) {
        this.r = r;
    }

    @Override
    public void run() {
        try {
            r.run();
        } finally {
            FastThreadLocal.removeAll(); // 移除NIO线程的所有本地变量副本
        }
    }
}
```

可以看到，在Runnable任务运行结束后，将调用`FastThreadLocal.removeAll()`方法，移除FastThreadLocalThread线程的所有线程本地变量副本。

```java
public static void removeAll() {
    InternalThreadLocalMap threadLocalMap = InternalThreadLocalMap.getIfSet();
    if (threadLocalMap == null) {
        return;
    }

    try {
        Object v = threadLocalMap.indexedVariable(variablesToRemoveIndex);
        if (v != null && v != InternalThreadLocalMap.UNSET) {
            @SuppressWarnings("unchecked")
            Set<FastThreadLocal<?>> variablesToRemove = (Set<FastThreadLocal<?>>) v;
            FastThreadLocal<?>[] variablesToRemoveArray =
                    variablesToRemove.toArray(new FastThreadLocal[variablesToRemove.size()]);
            for (FastThreadLocal<?> tlv: variablesToRemoveArray) {
                tlv.remove(threadLocalMap); // 删除线程threadLocalMap上的所有缓存
            }
        }
    } finally {
        InternalThreadLocalMap.remove();
    }
}
```

从FastThreadLocal.removeAll()方法可以看出，这个时候variablesToRemoveIndex索引起到了作用。通过variablesToRemoveIndex拿到了当前线程InternalThreadLocalMap中的`indexedVariables`数组中该索引处的`Set<FastThreadLocal>`集合。这个集合中的FastThreadLocal引用，与当前线程的InternalThreadLocalMap中的各个本地变量相对应，因此可以通过这些FastThreadLocal引用，删除线程InternalThreadLocalMap中的所有线程本地变量。最后再将线程的threadLocalMap变量设置为null:

```java
InternalThreadLocalMap.remove();

public static void remove() {
    Thread thread = Thread.currentThread();
    if (thread instanceof FastThreadLocalThread) {
        ((FastThreadLocalThread) thread).setThreadLocalMap(null);
    } else {
        slowThreadLocalMap.remove();
    }
}
```

至此，有关FastThreadLocal的内容分析完毕。

### 参考文章

- [惊：FastThreadLocal吞吐量居然是ThreadLocal的3倍！！！](http://www.jiangxinlingdu.com/interview/2019/07/01/fastthreadlocal.html)
- [Netty 之线程本地变量 FastThreadLocal](http://anyteam.me/netty-FastThreadLocal/)

