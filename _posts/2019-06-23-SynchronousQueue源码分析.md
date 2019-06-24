---
layout:     post
title:      SynchronousQueue源码分析
subtitle:   BlockingQueue
date:       2019-06-23
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Java基础
    - JCF
---

# SynchronousQueue源码分析

SynchronousQueue是一种特殊的阻塞队列，不同于LinkedBlockingQueue、ArrayBlockingQueue等阻塞队列，其内部没有任何容量，任何的入队操作都需要等待其他线程的出队操作，反之亦然。任意线程（生产者线程或者消费者线程，生产类型的操作比如put、offer，消费类型的操作比如poll、take）都会等待直到获得数据或者交付完成数据才会返回，一个生产者线程的使命是将线程附着着的数据交付给一个消费者线程，而一个消费者线程则是等待一个生产者线程的数据。它们在匹配到互补线程的时候就会做数据交换，比如生产者线程遇到消费者线程时，或者消费者线程遇到生产者线程时，一个生产者线程就会将数据交付给消费者线程，然后共同退出。在Java线程池**Executors.newCachedThreadPool**中就使用了这种阻塞队列。

SynchronousQueue的构造器有一个fair选项，支持公平和非公平两种线程竞争机制，fair为true表示公平模式，否则表示非公平模式。公平模式使用先进先出队列（FIFO Queue）保存生产者或者消费者线程，非公平模式使用后进先出的栈(LIFO Stack)保存。

### 一、SynchronousQueue的构造函数

```java
public SynchronousQueue() { // 默认非公平的策略
    this(false);
}

public SynchronousQueue(boolean fair) { // fair变量指定公平与非公平策略
    transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
}
```

通过SynchronousQueue的构造函数可以看出，公平模式使用TransferQueue实现，而非公平模式使用TransferStack实现。

```java
abstract static class Transferer<E> {
    abstract E transfer(E e, boolean timed, long nanos);
}
```

TransferQueue和TransferStack均继承自内部抽象类Transferer，并实现唯一的抽象函数transfer，即使用SynchronousQueue阻塞队列交换数据都通过该transfer函数完成。

### 二、公平模式 TransferQueue

公平模式使用一个FIFO队列保存线程，TransferQueue的结构如下所示:

```java
static final class TransferQueue<E> extends Transferer<E> {
    /** TransferQueue中的节点  */
    static final class QNode {
        volatile QNode next;          // next指向队列中的下一个节点
        volatile Object item;         // 数据
        volatile Thread waiter;       // 对应的线程
        final boolean isData;         // 是否为数据(表示该节点由生产者创建还是由消费者创建;true：生产者创建(由于生产者是放入数据)，false：消费者创建)

        QNode(Object item, boolean isData) {
            this.item = item;
            this.isData = isData;
        }
				// CAS设置next
        boolean casNext(QNode cmp, QNode val) {
            return next == cmp &&
                UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }
				// CAS设置item
        boolean casItem(Object cmp, Object val) {
            return item == cmp &&
                UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
        }
      	// 取消匹配，CAS设置item为 节点本身
        void tryCancel(Object cmp) {
            UNSAFE.compareAndSwapObject(this, itemOffset, cmp, this);
        }
				// 是否已取消
        boolean isCancelled() {
            return item == this;
        }
				// 是否已不在队列中
        boolean isOffList() {
            return next == this;
        }
    }

    /** 队列头 */
    transient volatile QNode head;
    /** 队列尾 */
    transient volatile QNode tail;
    
  	// 对应中断或超时节点的前继节点，这个节点存在的意义是标记, 它的下个节点要删除
    transient volatile QNode cleanMe;

    TransferQueue() {
      	// TransferQueue初始化时会构造一个dummy node，并且head一直是个dummy node
        QNode h = new QNode(null, false); // initialize to dummy node.
        head = h;
        tail = h;
    }

  	// 推进head，原head的next指向原head
    void advanceHead(QNode h, QNode nh) {
        if (h == head &&
            UNSAFE.compareAndSwapObject(this, headOffset, h, nh))
            h.next = h; // forget old next
    }

  	// 推进tail
    void advanceTail(QNode t, QNode nt) {
        if (tail == t)
            UNSAFE.compareAndSwapObject(this, tailOffset, t, nt);
    }

  	// CAS设置cleanMe变量
    boolean casCleanMe(QNode cmp, QNode val) {
        return cleanMe == cmp &&
            UNSAFE.compareAndSwapObject(this, cleanMeOffset, cmp, val);
    }

    @SuppressWarnings("unchecked")
    E transfer(E e, boolean timed, long nanos) {
        // ... 核心方法，后面详解
    }

    Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
        // ... 后面详解
    }

    void clean(QNode pred, QNode s) {
        // ... 后面详解
    }
}
```

可以看出TransferQueue使用了内部类QNode来封装待保存的线程作为队列的节点，QNode的next属性指向队列中的下一个节点；TransferQueue分别有一个用于指向队列的头部（head）和尾部（tail）的指针。队列初始化时会默认创建一个dummy node。

TransferQueue有三个重要的方法: transfer、awaitFulfill和clean，其中transfer是队列操作的入口函数，在put/take/poll/offer等接口中均是调用transfer函数来操作SynchronousQueue阻塞队列传递数据；另外两个方法在transfer函数中调用。

##### 1.transfer方法

transfer方法的基本算法为循环尝试执行以下两种操作之一：

1. 如果队列为空，或者队列中包含与该节点相同模式的节点（都为生产者或者都为消费者），尝试将节点加入队列中挂起并等待匹配，匹配成功则返回相应的值，若被取消则返回null。
2. 如果队列中包含与该节点互补的节点，则从队列头部开始，通过CAS操作等待节点(wait node)的item，尝试匹配，匹配成功，唤醒匹配的等待节点，并从队列中出队，并且返回匹配值。

transfer方法分析如下：

```java
// 参数e: 如果 e 不是 null(说明是生产者调用) ，将 item 交给消费者，并返回 e；反之，如果是 null
//（说明是消费者调用），将生产者提供的 item 返回给消费者。
E transfer(E e, boolean timed, long nanos) {
    QNode s = null; // constructed/reused as needed
  	// 当输入的是数据时，isData 就是 ture，表明这个操作是一个生成数据的操作；同理，
  	// 当调用者输入的是 null，则是在消费数据。
    boolean isData = (e != null);

    for (;;) {
        QNode t = tail;
        QNode h = head;
        if (t == null || h == null)         // 头/尾节点未初始化，自旋重来
            continue;                       

      	// ①队列为空, 或队列尾节点和自己类型相同(都是生产者或者消费者)
        if (h == t || t.isData == isData) { // empty or same-mode
            QNode tn = t.next; // 尾部节点的下一个节点
            if (t != tail)     // 如果t和tail不一样，说明tail被其他的线程改了，重来
                continue;
	          // 如果这个过程中又有新的线程插入队列，tail节点后又有新的节点，则修改tail指向新的节点
            if (tn != null) {
                advanceTail(t, tn);
                continue;
            }
            if (timed && nanos <= 0)  // 限时等待，但时间没了，直接返回null，如poll()/offer(E)方法
                return null;
            if (s == null) // 如果能走到这里，说明tail.next=null，这里的判断是避免重复创建Qnode对象
                s = new QNode(e, isData);
          	// CAS修改tail.next=s，即将节点s加入队列中;如果操作失败，说明t.next!=null，
          	// 有其他线程加入了节点，循环重新开始
            if (!t.casNext(null, s))        
                continue;
						// 修改队列尾指针tail指向节点s
            advanceTail(t, s);           
            Object x = awaitFulfill(s, e, timed, nanos); // 线程挂起、等待匹配
            if (x == s) {    // 说明等待节点s被取消了
                clean(t, s); // 如果节点被取消则从队列中清除，并且返回null
                return null;
            }
					
          	// 如果一切顺利，确实被其他线程唤醒了，其他线程也交换了数据。
            if (!s.isOffList()) {           // 节点s还在队列中
              	// 这一步是将s节点设置为head
              	// 注意这里传给advanceHead的参数是t，为什么？
                // 因为节点t是节点s的前驱节点，执行到这里说明节点s代表的线程被唤醒得到匹配，所以唤醒的
              	// 时候，节点s肯定是队列中第一个节点，前驱节点t是head指针.
                advanceHead(t, s);          // unlink if head
              	// 当x不是null时，表明对方线程是存放数据的。  
                if (x != null)              // and forget fields
                    s.item = s;
                s.waiter = null;
            }
          	// x不是null，表明对方线程是生产者，返回他生产的数据；如果是null，说明对方线程是消费者，
          	// 那自己就是生产者，返回自己的数据，表示成功。
            return (x != null) ? (E)x : e;
        // ②如果队列尾节点和当前节点类型不同，称之为互补，则进行匹配
        } else {                            // complementary-mode
          	// 公平模式，从队列头部开始匹配，由于head指向一个dummy节点，所以待匹配的节点为head.next
            QNode m = h.next;               // node to fulfill
          	// 如果下方这些判断没过，说明并发修改了，自旋重来
            if (t != tail || m == null || h != head)
                continue;                   // inconsistent read

            Object x = m.item;
            if (isData == (x != null) ||    // 并发情况下m已经和其他线程匹配过了
                x == m ||                   // m被取消了
                !m.casItem(x, e)) {         // 以上两种情况都不是，则进行CAS交换item
              	// 上述if的前两个条件任意满足，或者第三个条件CAS失败，到这里将m节点出队，并重试
                advanceHead(h, m);          // dequeue and retry
                continue;
            }
						// 匹配成功，修改head指针指向下一节点;同时唤醒匹配成功的线程
            advanceHead(h, m);              // successfully fulfilled
            LockSupport.unpark(m.waiter); // 唤醒等待节点的线程
	          // 如果x不是null，表明这是一次消费数据的操作，反之，这是一次生产数据的操作。
            return (x != null) ? (E)x : e;
        }
    }
}
```

##### 2.awaitFulfill方法

当线程无法得到匹配时，则加入队列尾部，并调用awaitFulfill函数挂起线程，等待其他匹配的线程唤醒。由于线程挂起是一个耗时的操作，所以线程挂起前如果满足一定的条件（当前线程节点为队列中第一个节点，head后一个节点）则先进行自旋，检查是否得到匹配，如果在自旋过程中得到匹配则不需要挂起线程，否则挂起线程。awaitFulfill函数能够响应中断，在等待过程中如果被中断或者等待超时，则取消节点等待。

```java
Object awaitFulfill(QNode s, E e, boolean timed, long nanos) { // 等待匹配
    /* Same idea as TransferStack.awaitFulfill */
    final long deadline = timed ? System.nanoTime() + nanos : 0L; // 计算deadline
    Thread w = Thread.currentThread(); // 当前线程
    int spins = ((head.next == s) ? // 节点s若为第一个节点，即head后一个节点，则阻塞之前优先自旋
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0);
    for (;;) {
        if (w.isInterrupted()) // 如果已被中断，则取消等待
            s.tryCancel(e);
        Object x = s.item;
      	// 在进行线程阻塞->唤醒, 线程中断, 等待超时, 这时x!=e,直接return回去
        if (x != e)
            return x;
        if (timed) { // 限时等待
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) { // 如果没时间了，则取消等待
                s.tryCancel(e);
                continue; // 下次循环时退出
            }
        }
        if (spins > 0) // 如果还有自旋次数，则递减
            --spins;
        else if (s.waiter == null) // 如果自旋次数到了，且s的等待线程还没有赋值，则先赋值
            s.waiter = w;
        else if (!timed) // 如果自旋次数到了，且线程赋值过了，且没有限制时间，则阻塞wait
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold) // 限时等待，且剩余时间大于阈值，则阻塞对应时间
            LockSupport.parkNanos(this, nanos);
    }
}
```

##### 3.clean方法

```java
// 对中断的或等待超时的节点s进行清除操作。pred为前驱节点
void clean(QNode pred, QNode s) {
    s.waiter = null; // forget thread
   
    while (pred.next == s) { // Return early if already unlinked
        QNode h = head;
        QNode hn = h.next;   // Absorb cancelled first node as head
        if (hn != null && hn.isCancelled()) { // 检查第一个节点的状态
            advanceHead(h, hn); // 如果第一个节点hn中断或者超时, 则出队
            continue; // 继续检查
        }
        QNode t = tail;      // Ensure consistent read for tail
        if (t == h) // 队列为空，返回
            return;
        QNode tn = t.next;
        if (t != tail) // 其他线程改变了tail, continue重来
            continue;
        if (tn != null) { // 有新的节点加入，则修改tail指向新的节点
            advanceTail(t, tn);
            continue;
        }
      	// 如果被取消的节点s不是队尾节点，则直接修改其前驱节点的next指针，将节点s从队列中删除
        if (s != t) {        // If not tail, try to unsplice
            QNode sn = s.next;
            if (sn == s || pred.casNext(s, sn)) // sn == s表示s已不在队列中，直接返回
                return;
        }
      	// 如果s是队尾节点，则用cleanMe节点指向其前驱节点，等待以后s不是队尾时再从队列中清除
        // 如果 cleanMe == null，则直接将pred赋值给cleanMe即可
        // 否则，说明之前有一个节点等待被清除，并且用cleanMe指向了其前驱节点，所以现在需要将其从队列中清除
        QNode dp = cleanMe;
        if (dp != null) {    // Try unlinking previous cancelled node
            QNode d = dp.next; // 待删除节点
            QNode dn; // 待删除节点的下一个节点
            if (d == null ||               // d is gone or 待删除节点不存在
                d == dp ||                 // d is off list or 待删除节点已不在队列中
                !d.isCancelled() ||        // d not cancelled or 待删除节点未取消
                // 以上3种情况，直接将cleanMe置为null。下面这种情况CAS删除节点d，再将cleanMe置为null
                (d != t &&                 // d not tail and // d不是尾部节点
                 (dn = d.next) != null &&  //   has successor // d有后继节点
                 dn != d &&                //   that is on list // d在队列中
                 dp.casNext(d, dn)))       // d unspliced // 删除d
                casCleanMe(dp, null); // 将cleanMe置为null
          	// 下面若成立, 说明清除节点s成功, 直接return, 不然的话要再次loop, 在下面设置这次的
          	// cleanMe，然后再返回
            if (dp == pred)
                return;      // s is already saved node
        } else if (casCleanMe(null, pred)) 
          	// 原来的cleanMe是null, 则将pred标记为cleamMe，为下次清除s节点做标识
            return;          // Postpone cleaning s
    }
}
```

这个clean方法都是由节点线程中断或等待超时时调用的, 清除时分两种情况讨论:

1. 删除的节点不是queue尾节点，这时直接pred.casNext(s, s.next) 方式来进行删除；
2. 删除的节点是队尾节点
  1) 此时cleanMe == null，则前继节点pred标记为 cleanMe，为下次删除做准备；
  2) 此时cleanMe != null，先删除上次需要删除的节点，然后将cleanMe置null, 然后再将pred赋值给cleanMe。

### 三、非公平模式 TransferStack

非公平模式使用一个LIFO栈保存线程，TransferStack的结构如下所示:

```java
static final class TransferStack<E> extends Transferer<E> {
    
    /* Modes for SNodes, ORed(或) together in node fields */
    /** 该值表示节点是一个未匹配的消费者 */
    static final int REQUEST    = 0;
    /** 该值表示节点是一个未匹配的生产者 */
    static final int DATA       = 1;
    /** 该值表示节点正在匹配生产者或消费者 */
    static final int FULFILLING = 2;

  	// 判断节点是否正在匹配
    static boolean isFulfilling(int m) { return (m & FULFILLING) != 0; }

    /** TransferStacks中的节点 */
    static final class SNode {
        volatile SNode next;        // 下一个节点，指向栈底(先入栈的节点)
        volatile SNode match;       // 与之匹配的节点
        volatile Thread waiter;     // 线程
        Object item;                // 数据/null
        int mode; // 模式，上面三个值或者 "或||" 的结果

        SNode(Object item) {
            this.item = item;
        }
				// CAS设置next
        boolean casNext(SNode cmp, SNode val) {
            return cmp == next &&
                UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
        }

       	// 尝试匹配并唤醒等待线程
        boolean tryMatch(SNode s) {
            if (match == null &&
                UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
                Thread w = waiter;
                if (w != null) {    // waiters need at most one unpark
                    waiter = null;
                    LockSupport.unpark(w); // 唤醒线程
                }
                return true;
            }
            return match == s;
        }

      	// 尝试取消匹配
        void tryCancel() {
            UNSAFE.compareAndSwapObject(this, matchOffset, null, this);
        }
        boolean isCancelled() { // 是否已取消
            return match == this;
        }
    }

  	// 栈顶节点(头部)
    volatile SNode head;
		// CAS设置头部节点
    boolean casHead(SNode h, SNode nh) {
        return h == head &&
            UNSAFE.compareAndSwapObject(this, headOffset, h, nh);
    }

  	// 创建或者重置节点
    static SNode snode(SNode s, Object e, SNode next, int mode) {
        if (s == null) s = new SNode(e);
        s.mode = mode;
        s.next = next;
        return s;
    }

    @SuppressWarnings("unchecked")
    E transfer(E e, boolean timed, long nanos) {
      	// ... 后面分析
    }

    SNode awaitFulfill(SNode s, boolean timed, long nanos) {
        // ... 后面分析
    }

  	// 如果s为头结点，或者头结点正在匹配，则自旋
    boolean shouldSpin(SNode s) {
        SNode h = head;
        return (h == s || h == null || isFulfilling(h.mode));
    }

    void clean(SNode s) {
        // ... 后面分析
    }
}
```

TransferStack是一个栈结构，使用内部类SNode来封装待保存的线程作为栈中节点，SNode的next属性用于指向栈中下一个节点。TransferStack有一个用于指向栈顶的指针head，且没有初始化为dummy node。

TransferStack中同样有三个重要的方法: transfer、awaitFulfill和clean，其作用和TransferQueue类似。

##### 1.transfer

transfer方法基本算法为循环尝试以下三种操作之一：

1. 如果栈为空，或者栈中包含的节点与该节点为同一模式（都为REQUEST或都为DATA），则尝试将节点入栈并等待匹配，匹配成功返回相应的值，如果被取消则返回null。
2. 如果栈中包含互补模式的节点，则**尝试入栈一个模式包含FULFILLING的节点**，并且匹配相应的处于等待中的栈中节点，匹配成功，将成功匹配的两个节点都出栈，并返回相应的值。
3. 如果栈顶为一个模式包含FULFILLING的节点，则帮助其执行匹配和出栈操作，然后在循环执行自己的匹配操作。帮助其他线程匹配操作和自身执行匹配操作代码基本一致，除了不返回匹配的值。

非公平操作和公平操作有一点不一样，非公平操作时，模式不同得到匹配时也需要先将节点入栈，最后将匹配的两个节点一起出栈；公平操作中，得到匹配时，不需要将节点加入队列，直接从队列头部匹配一个节点。

```java
E transfer(E e, boolean timed, long nanos) {
    
    SNode s = null; // constructed/reused as needed
    int mode = (e == null) ? REQUEST : DATA; // 模式：请求数据或者生产数据

    for (;;) {
        SNode h = head;
        if (h == null || h.mode == mode) {  // empty or same-mode 栈为空 或者 模式相同
            if (timed && nanos <= 0) {      // can't wait // 限时等待，且时间没了，返回null
                if (h != null && h.isCancelled()) // 清除被取消的节点
                    casHead(h, h.next);     // pop cancelled node
                else
                    return null; // 返回null
            } else if (casHead(h, s = snode(s, e, h, mode))) { // 入栈s，变为栈顶
                SNode m = awaitFulfill(s, timed, nanos); // 挂起、等待匹配
                if (m == s) { // s如果被取消等待则清除并返回null
                    clean(s);
                    return null;
                }
              	// 运行到这里说明得到匹配被唤醒，从栈顶将匹配的两个节点一起出栈（修改栈顶指针）
                if ((h = head) != null && h.next == s)
                    casHead(h, s.next);     // help s's fulfiller
                return (E) ((mode == REQUEST) ? m.item : s.item); // 返回值
            }
        // 模式不同且头结点未正在匹配，则尝试匹配  
        } else if (!isFulfilling(h.mode)) { // try to fulfill
            if (h.isCancelled())            // 头结点已经被取消，则出栈并重试
                casHead(h, h.next);        
            else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) { // 入栈s，变为栈顶
							// 循环直到匹配或者等待节点没了
              for (;;) { // loop until matched or waiters disappear
                    SNode m = s.next;       // m是s的匹配节点
                    if (m == null) {        // 说明栈中s之后无元素了，重新进行最外层的循环
                        casHead(s, null);   // pop fulfill node
                        s = null;           // use new node next time
                        break;              // restart main loop
                    }
                		// 将s设置为m的匹配节点，并更新栈顶为m.next，即将s和m同时出栈
                    SNode mn = m.next;
                    if (m.tryMatch(s)) {
                        casHead(s, mn);     // pop both s and m
                        return (E) ((mode == REQUEST) ? m.item : s.item); // 返回
                    } else                  // lost match
                        s.casNext(m, mn);   // help unlink 匹配失败，则m出栈
                }
            }
        // 头部节点正在匹配，则当前线程帮助其匹配 
        } else {                            // help a fulfiller
            SNode m = h.next;               // m is h's match 头节点的匹配节点m
            if (m == null)                  // waiter is gone 无匹配节点
                casHead(h, null);           // pop fulfilling node 则头节点出栈
            else {
              	// 将h设置为m的匹配节点，并更新栈顶为m.next，即将h和m同时出栈
                SNode mn = m.next;
                if (m.tryMatch(h))          // help match
                    casHead(h, mn);         // pop both h and m
                else                        // lost match
                    h.casNext(m, mn);       // help unlink 匹配失败，则m出栈
            }
        }
    }
}
```

##### 2.awaitFulfill

awaitFulfill方法功能同TransferQueue中同名方法，当无法匹配时加入栈后挂起线程，挂起前满足一定的条件则先自旋。

```java
SNode awaitFulfill(SNode s, boolean timed, long nanos) {
   
    final long deadline = timed ? System.nanoTime() + nanos : 0L; // deadline计算
    Thread w = Thread.currentThread(); // 线程
    int spins = (shouldSpin(s) ?
                 (timed ? maxTimedSpins : maxUntimedSpins) : 0); // 判断自旋与否
    for (;;) {
        if (w.isInterrupted()) // 如果被中断，则取消等待
            s.tryCancel();
        SNode m = s.match;
      	// s.match ！= null,有几种情况：
        // (1)因中断或超时被取消了，此时s.match=s
        // (2)匹配成功了，此时s.match=另一个节点
        if (m != null)
            return m;
        if (timed) {
            nanos = deadline - System.nanoTime();
            if (nanos <= 0L) { // 限时等待，且时间没了，则取消等待，在下次循环退出
                s.tryCancel();
                continue;
            }
        }
        if (spins > 0) // 剩余自旋次数大于0，则先自旋
            spins = shouldSpin(s) ? (spins-1) : 0;
        else if (s.waiter == null) // 自选剩余次数没了，且waiter变量还未赋值，则先赋值
            s.waiter = w; // establish waiter so can park next iter
        else if (!timed) // waiter变量已赋值，且未限时，则直接阻塞等待
            LockSupport.park(this);
        else if (nanos > spinForTimeoutThreshold)// 为限时等待且剩余时间大于阈值，则先阻塞等待
            LockSupport.parkNanos(this, nanos);
    }
}
```

##### 3.clean

clean方法功能同TransferQueue中同名方法，当节点因被中断或者等待超时而被取消时，从栈中删除该节点。

```java
void clean(SNode s) {
    s.item = null;   // forget item
    s.waiter = null; // forget thread

    SNode past = s.next;
    if (past != null && past.isCancelled())  // 找到s之后第一个未被取消的节点past
        past = past.next;

    // Absorb cancelled nodes at head
    SNode p;
    while ((p = head) != null && p != past && p.isCancelled())
        casHead(p, p.next); // 如果栈顶被取消了，更改栈顶指针，将取消的节点从栈中删除

    // Unsplice embedded nodes 把p和past之间所有被取消的节点移除栈中，包括s
    while (p != null && p != past) {
        SNode n = p.next;
        if (n != null && n.isCancelled())
            p.casNext(n, n.next);
        else
            p = n;
    }
}
```

### 四、SynchronousQueue常用操作

##### 1.put(E)

```java
public void put(E e) throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, false, 0) == null) {
        Thread.interrupted();
        throw new InterruptedException();
    }
}
```

向SynchronousQueue队列中插入数据，如果没有相应的线程接收数据，则put操作一直阻塞，直到有消费者线程接收数据。可以看到transfer方法的第二个参数timed为false，表示不会超时、一直阻塞。

##### 2.offer(E, long, TimeUnit)

```java
public boolean offer(E e, long timeout, TimeUnit unit)
    throws InterruptedException {
    if (e == null) throw new NullPointerException();
    if (transferer.transfer(e, true, unit.toNanos(timeout)) != null)
        return true;
    if (!Thread.interrupted())
        return false;
    throw new InterruptedException();
}
```

带超时时间的插入数据，如果没有相应的线程接收数据，则等待timeout时间，如果等待超时还未传递完数据，则返回false。

##### 3.offer(E) 

```java
public boolean offer(E e) {
    if (e == null) throw new NullPointerException();
    return transferer.transfer(e, true, 0) != null;
}
```

非阻塞操作插入数据，如果没有相应的线程接收数据，则直接返回false。可以看到transfer函数的第二个参数timed为true，第三个参数时间为0，即超时时间为0，也就是不等待。

##### 4.take()

```java
public E take() throws InterruptedException {
    E e = transferer.transfer(null, false, 0);
    if (e != null)
        return e;
    Thread.interrupted();
    throw new InterruptedException();
}
```

阻塞操作从SynchronousQueue队列中获取数据，如果没有获取到数据，则一直阻塞。

##### 5.poll(long, TimeUnit)

```java
public E poll(long timeout, TimeUnit unit) throws InterruptedException {
    E e = transferer.transfer(null, true, unit.toNanos(timeout));
    if (e != null || !Thread.interrupted())
        return e;
    throw new InterruptedException();
}
```

带超时时间的从SynchronousQueue队列中获取数据。

##### 6.poll()

```java
public E poll() {
    return transferer.transfer(null, true, 0);
}
```

非阻塞操作从SynchronousQueue队列中获取数据。

### 参考文献

- [SynchronousQueue 源码分析 (基于Java 8)](https://www.jianshu.com/p/95cb570c8187)
- [并发编程之 SynchronousQueue 核心源码分析](https://juejin.im/post/5ae754c7f265da0ba76f8534)
- [[java1.8源码笔记]SynchronousQueue详解](https://luoming1224.github.io/2018/03/19/[java1.8%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0]SynchronousQueue%E8%AF%A6%E8%A7%A3/)
- [JDK8 API](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/SynchronousQueue.html)