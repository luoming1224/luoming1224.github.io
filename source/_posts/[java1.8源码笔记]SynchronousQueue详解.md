---
title: '[java1.8源码笔记]SynchronousQueue详解'
date: 2018-03-19 17:20:40
categories: 'java1.8源码笔记'
tags:
---
# 概述
SynchronousQueue是一种特殊的阻塞队列，不同于LinkedBlockingQueue、ArrayBlockingQueue等阻塞队列，其内部没有任何容量，任何的入队操作都需要等待其他线程的出队操作，反之亦然。任意线程（生产者线程或者消费者线程，生产类型的操作比如put，offer，消费类型的操作比如poll，take）都会等待直到获得数据或者交付完成数据才会返回，一个生产者线程的使命是将线程附着着的数据交付给一个消费者线程，而一个消费者线程则是等待一个生产者线程的数据。它们会在匹配到互斥线程的时候就会做数据交易，比如生产者线程遇到消费者线程时，或者消费者线程遇到生产者线程时，一个生产者线程就会将数据交付给消费者线程，然后共同退出。在java线程池newCachedThreadPool中就使用了这种阻塞队列。

<!--more-->

SynchronousQueue有一个fair选项，支持公平和非公平两种竞争机制，fair为true表示公平模式，否则表示非公平模式。公平模式使用先进先出队列（FIFO Queue）保存生产者或者消费者线程，非公平模式使用后进先出的栈(LIFO Stack)保存。

# 基本原理
当线程调用操作向阻塞队列SynchronousQueue中插入数据或者获取数据时，如果阻塞队列中队列或栈为空，或者队列或栈中的线程在等待与当前线程相同的操作（都为生产者插入数据或者都为消费者获取数据），则将当前线程封装为一个节点并插入队列或栈中，并挂起；如果队列或栈中的线程刚好在等待与当前线程互补的操作（队列或栈中为生产者，当前线程为消费者，或者反之亦然），则执行匹配操作，匹配成功的话则唤醒队列或栈中挂起的与之匹配的那个线程，完成数据交换。

# 构造函数
通过SynchronousQueue构造函数可以看出公平模式使用TransferQueue实现，非公平模式使用TransferStack实现。

	public SynchronousQueue(boolean fair) {
        transferer = fair ? new TransferQueue<E>() : new TransferStack<E>();
    }

TransferQueue和TransferStack均继承自内部抽象类Transferer，并实现唯一的抽象函数transfer，即使用SynchronousQueue阻塞队列交换数据都通过该transfer函数完成。

	abstract static class Transferer<E> {
        abstract E transfer(E e, boolean timed, long nanos);
    }

# 公平模式
公平模式使用一个FIFO队列保存线程，TransferQueue的结构如下所示：

	static final class TransferQueue<E> extends Transferer<E> {

        /** Node class for TransferQueue. */
        static final class QNode {
            volatile QNode next;          // next node in queue 指向队列中的下一个节点
            volatile Object item;         // CAS'ed to or from null 节点包含的数据
            volatile Thread waiter;       // to control park/unpark 等待在该节点上的线程
            final boolean isData;   // 表示该节点由生产者创建还是由消费者创建;true：生产者创建(由于生产者是放入数据)，false：消费者创建

            QNode(Object item, boolean isData) {
                this.item = item;
                this.isData = isData;
            }
			......
        }

        /** Head of queue */
        transient volatile QNode head;
        /** Tail of queue */
        transient volatile QNode tail;
        
        transient volatile QNode cleanMe;

        TransferQueue() {
            QNode h = new QNode(null, false); // initialize to dummy node.
            head = h;
            tail = h;
        }
		......
	}

可以看出TransferQueue同一个普通的双端队列，使用内部类QNode来封装待保存的线程作为队列的节点，QNode的next属性用于指向队列中下一个节点；TransferQueue分别有一个用于指向队列的头部（head）和尾部（tail）的指针。

队列初始化时会默认创建一个dummy node。

TransferQueue有三个重要的方法，transfer、awaitFulfill和clean，其中transfer是队列操作的入口函数，在put/take/poll/offer等接口中均是调用transfer函数来操作SynchronousQueue阻塞队列传递数据；另外两个方法在transfer函数中调用。

## transfer
transfer函数的基本算法为循环尝试执行以下两种操作之一：

1. 如果队列为空，或者队列中包含与该节点相同模式的节点（都为生产者或者都为消费者），尝试将节点加入队列中挂起并等待匹配，匹配成功则返回相应的值，若被取消则返回null。
2. 如果队列中包含与该节点互补模式的节点，则从队列头部通过CAS操作等待节点的item,尝试匹配，匹配成功，唤醒匹配的等待节点，并从队列中出队，并且返回匹配值。

transfer函数如下所示：

	E transfer(E e, boolean timed, long nanos) {
            QNode s = null; // constructed/reused as needed
            boolean isData = (e != null);

            for (;;) {
                QNode t = tail;
                QNode h = head;
                // 根据源码注释，在SynchronousQueue中不会出现这两种情况，因为在构造函数中对head和tail初始化了
                if (t == null || h == null)         // saw uninitialized value 根据源码注释，在SynchronousQueue中不会出现这两种情况，因为在
                    continue;                       // spin

                // 队列为空 或者 队列中线程与当前线程模式相同（队列中只存在一种模式的线程）
                if (h == t || t.isData == isData) { // empty or same-mode
                    QNode tn = t.next;
                    if (t != tail)                  // inconsistent read 被其他线程修改了tail指针，循环重新开始
                        continue;
                    if (tn != null) {               // lagging tail 如果这个过程中又有新的线程插入队列，tail节点后又有新的节点，则修改tail指向新的节点
                        advanceTail(t, tn);
                        continue;
                    }
                    if (timed && nanos <= 0)        // can't wait 表示非阻塞操作，不等待，模式相同说明没有可匹配的线程，直接返回null
                        return null;
                    if (s == null)
                        s = new QNode(e, isData);
                    // CAS修改tail.next=s，即将节点s加入队列中;如果操作失败说明t.next!=null,有其他线程加入了节点，循环重新开始
                    if (!t.casNext(null, s))        // failed to link in
                        continue;

                    // 修改队列尾指针tail指向节点s;这里没有用CAS操作，因为即使设置失败，其他线程会帮忙修改tail指针
                    advanceTail(t, s);              // swing tail and wait
                    Object x = awaitFulfill(s, e, timed, nanos);    //线程挂起等待匹配
                    if (x == s) {                   // wait was cancelled 如果线程被取消则从队列中清除，并且返回null
                        clean(t, s);
                        return null;
                    }

                    if (!s.isOffList()) {           // not already unlinked
                        // 注意这里传给advanceHead的参数是t，为什么？
                        // 因为节点t是节点s的前驱节点，执行到这里说明节点s代表的线程被唤醒得到匹配，所以唤醒的时候，节点s肯定是队列中第一个节点，前驱节点t是head指针
                        // 这里也并没有用for循环保证CAS操作成功，是因为唤醒线程s的线程也会执行advanceHead操作
                        advanceHead(t, s);          // unlink if head
                        if (x != null)              // and forget fields
                            s.item = s;
                        s.waiter = null;
                    }
                    return (x != null) ? (E)x : e;

                } else {                            // complementary-mode 如果为互补模式，则进行匹配
                    QNode m = h.next;               // node to fulfill 公平模式，从队列头部开始匹配，由于head指向一个dummy节点，所以待匹配的节点为head.next
                    // 如果出现以下三种情况之一，重新循环匹配：
                    // (1)t != tail; tail指针被修改
                    // (2)m == null; 待匹配的线程节点不存在
                    // (3)h != head; head指针被修改,m节点被其他线程匹配成功了
                    if (t != tail || m == null || h != head)
                        continue;                   // inconsistent read

                    Object x = m.item;
                    // 如果出现以下几种情况之一，则重新循环匹配：
                    // (1)isData == (x != null); 线程的模式相同（都为生产者或者都为消费者），无法匹配
                    // (2)x == m; 节点m代表的线程在等待被匹配过程中被取消了
                    // (3)!m.casItem(x, e); CAS操作修改item失败
                    if (isData == (x != null) ||    // m already fulfilled
                        x == m ||                   // m cancelled
                        !m.casItem(x, e)) {         // lost CAS
                        advanceHead(h, m);          // dequeue and retry
                        continue;
                    }

                    // 匹配成功，修改head指针指向下一节点;同时唤醒匹配成功的线程
                    advanceHead(h, m);              // successfully fulfilled
                    LockSupport.unpark(m.waiter);
                    // x != null; 表示等待匹配的线程为生产者，则当前线程为消费者，返回生产者的数据x
                    // x == null; 表示等待匹配的线程为消费者，则当前线程为生产者，当前线程返回自身的数据
                    return (x != null) ? (E)x : e;
                }
            }
        }

## awaitFulfill
当线程无法得到匹配时，则加入队列尾部，并调用awaitFulfill函数挂起线程，等待其他匹配的线程唤醒。由于线程挂起是一个耗时的操作，所以线程挂起前如果满足一定的条件（当前线程节点为队列中第一个节点）则先进行自旋，检查是否得到匹配，如果在自旋过程中得到匹配则不需要挂起线程，否则挂起线程。awaitFulfill函数能够响应中断，在挂起过程中如果被中断或者等待超时，则取消节点等待。

	Object awaitFulfill(QNode s, E e, boolean timed, long nanos) {
            /* Same idea as TransferStack.awaitFulfill */
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Thread w = Thread.currentThread();
            // 自旋循环次数，如果是队列中的第一个节点则自旋，否则自旋次数为0（不自旋）
            int spins = ((head.next == s) ?
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);
            for (;;) {
                if (w.isInterrupted())  //支持响应中断异常
                    s.tryCancel(e);     // 中断异常，设置s.item=s
                Object x = s.item;
                // 如果s的item不等于e，有三种情况：
                // a.等待被取消了，此时x==s
                // b.匹配上了，此时x==另一个线程传过来的值
                // c.线程被打断，会取消，此时x==s
                // 不管是哪种情况都不要再等待了，返回即可
                if (x != e)
                    return x;
                if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) {
                        s.tryCancel(e); //等待超时，设置s.item=s，表示该节点已经取消
                        continue;
                    }
                }
                if (spins > 0)
                    --spins;
                else if (s.waiter == null)
                    s.waiter = w;
                else if (!timed)
                    LockSupport.park(this);
                else if (nanos > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanos);
            }
        }

## clean
当等待中的线程因被中断或者等待超时而被唤醒时，需要取消该线程的等待，并且从队列中删除。clean函数的实现思路为：

1. 如果被取消的节点不是队列的尾部节点，则直接通过修改其前驱节点的next指针指向其后继节点，将被取消的节点从队列中删除。
2. 如果被取消的节点是队列的尾部节点，则用cleanMe指针指向其前驱节点，等待以后再清除。

为什么这里要采用cleanMe标记而不是直接删除尾部节点？防止在删除尾部节点的同时有其他节点加入队列，导致尾部节点后面的节点全部一起被误删。
![](https://i.imgur.com/kSNl2MV.jpg)

如图所示，假如被删除的节点为图中节点d，且为队列中的尾部节点，删除过程中刚好有另外一个节点dn加入队列。加入队列有两步操作，(1)设置d.next=dn；(2)设置队列尾部指针tail=dn。而将节点d从队列尾部删除也有两步操作，(1)设置队列尾部指针tail=dp；(2)设置其前驱节点dp.next=null。

假如出现如下操作顺序：(1) d.next = dn; (2) tail = dp; (3) dp.next = null; (4) tail = dn；就会发生异常操作，将刚加入队列的节点dn一起从队列中删除了，而且队列尾部指针也出错了。所以防止类似的误删，在clean函数中采用cleanMe指针指向尾部节点的前驱节点，等待以后再删除该节点。

	void clean(QNode pred, QNode s) {
            s.waiter = null; // forget thread
            /*
             * At any given time, exactly one node on list cannot be
             * deleted -- the last inserted node. To accommodate this,
             * if we cannot delete s, we save its predecessor as
             * "cleanMe", deleting the previously saved version
             * first. At least one of node s or the node previously
             * saved can always be deleted, so this always terminates.
             */
            while (pred.next == s) { // Return early if already unlinked
                QNode h = head;
                QNode hn = h.next;   // Absorb cancelled first node as head
                // 从队列头部开始遍历，遇到被取消的节点则将其出队
                if (hn != null && hn.isCancelled()) {
                    advanceHead(h, hn);
                    continue;
                }
                QNode t = tail;      // Ensure consistent read for tail
                if (t == h)     // 表示队列为空
                    return;
                QNode tn = t.next;
                if (t != tail)
                    continue;
                if (tn != null) {     // 有新的节点加入，帮助其加入队列尾
                    advanceTail(t, tn);
                    continue;
                }
                // 如果被取消的节点s不是队尾节点，则修改其前驱节点的next指针，将节点s从队列中删除
                if (s != t) {        // If not tail, try to unsplice
                    QNode sn = s.next;
                    if (sn == s || pred.casNext(s, sn))
                        return;
                }
                // 如果s是队尾节点，则用cleanMe节点指向其前驱节点，等待以后s不是队尾时再从队列中清除
                // 如果 cleanMe == null，则直接将pred赋值给cleanMe即可
                // 否则，说明之前有一个节点等待被清除，并且用cleanMe指向了其前驱节点，所以现在需要将其从队列中清除
                QNode dp = cleanMe;
                if (dp != null) {    // Try unlinking previous cancelled node
                    QNode d = dp.next;
                    QNode dn;
                    // 以下四种情况下将cleanMe置为null，前面三种是特殊情况，最后一个是利用CAS操作正确的将待清楚的节点从队列中清除
                    // d == null; 表示待清除的节点不存在
                    // d == dp; dp被清除
                    // !d.isCancelled(); d没有被取消，说明哪里出错了
                    if (d == null ||               // d is gone or
                        d == dp ||                 // d is off list or
                        !d.isCancelled() ||        // d not cancelled or
                        (d != t &&                 // d not tail and
                         (dn = d.next) != null &&  //   has successor
                         dn != d &&                //   that is on list
                         dp.casNext(d, dn)))       // d unspliced
                        casCleanMe(dp, null);
                    if (dp == pred)     // 说明已经被其他线程更新了
                        return;      // s is already saved node
                } else if (casCleanMe(null, pred))
                    return;          // Postpone cleaning s
            }
        }

# 非公平模式
非公平模式使用一个LIFO栈保存线程，TransferStack的结构如下所示：

	static final class TransferStack<E> extends Transferer<E> {
       
        /* Modes for SNodes, ORed together in node fields */ // 节点代表的含义
        /** Node represents an unfulfilled consumer */
        static final int REQUEST    = 0;    // 消费者请求数据 (0x0000 0000)
        /** Node represents an unfulfilled producer */
        static final int DATA       = 1;    // 生产者生产数据 (0x0000 0001)
        /** Node is fulfilling another unfulfilled DATA or REQUEST */
        static final int FULFILLING = 2;    // 正在匹配中... (0x0000 0010)

        /** Returns true if m has fulfilling bit set. */
        static boolean isFulfilling(int m) { return (m & FULFILLING) != 0; }

        /** Node class for TransferStacks. */
        static final class SNode {
            volatile SNode next;        // next node in stack  指向栈中（栈底方向）上一个节点，先入栈的节点
            volatile SNode match;       // the node matched to this  与之匹配的节点
            volatile Thread waiter;     // to control park/unpark  等待在该节点上的线程
            Object item;                // data; or null for REQUESTs
            int mode;
            // Note: item and mode fields don't need to be volatile
            // since they are always written before, and read after,
            // other volatile/atomic operations.

            // 其他内部基本同TransferQueue，不同之处是当匹配到一个节点时并非是将被匹配的节点出栈，
            // 而是将匹配的节点入栈，然后同时将匹配上的两个节点一起出栈。

            SNode(Object item) {
                this.item = item;
            }

            boolean casNext(SNode cmp, SNode val) {
                return cmp == next &&
                    UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
            }

            /**
             * Tries to match node s to this node, if so, waking up thread.
             * Fulfillers call tryMatch to identify their waiters.
             * Waiters block until they have been matched.
             *
             * @param s the node to match
             * @return true if successfully matched to s
             */
            boolean tryMatch(SNode s) {
                if (match == null &&
                    UNSAFE.compareAndSwapObject(this, matchOffset, null, s)) {
                    Thread w = waiter;
                    if (w != null) {    // waiters need at most one unpark
                        waiter = null;
                        LockSupport.unpark(w);
                    }
                    return true;
                }
                // 如果被其他线程帮助完成匹配操作，则有可能上面的match != null 或者 CAS操作失败
                // 这个时候只要检查match是否等于s，如果是则同样表示匹配成功
                return match == s;
            }
        }

        /** The head (top) of the stack */
        volatile SNode head;    // 指向栈顶，并且没有初始化为dummy节点

        static SNode snode(SNode s, Object e, SNode next, int mode) {
            if (s == null) s = new SNode(e);
            s.mode = mode;
            s.next = next;
            return s;
        }
		......
	}

TransferStack是一个栈结构，使用内部类SNode来封装待保存的线程作为栈中节点，SNode的next熟悉用于指向栈中上一个节点。TransferStack有一个用于指向栈顶的指针head，且没有初始化为dummy node。

TransferStack中同样有三个重要的方法，transfer、awaitFulfill和clean，其作用和TransferQueue类似。

## transfer
transfer函数基本算法为循环尝试以下三种操作之一：

1. 如果栈为空，或者栈中包含的节点与该节点为同一模式（都为REQUEST或都为DATA），则尝试将节点入栈并等待匹配，匹配成功返回相应的值，如果被取消则返回null。
2. 如果栈中包含互补模式的节点，则尝试入栈一个模式包含FULFILLING的节点，并且匹配相应的处于等待中的栈中节点，匹配成功，将成功匹配的两个节点都出栈，并返回相应的值。
3. 如果栈顶为一个模式包含FULFILLING的节点，则帮助其执行匹配和出栈操作，然后在循环执行自己的匹配操作。帮助其他线程匹配操作和自身执行匹配操作代码基本一致，除了不返回匹配的值。

非公平操作和公平操作有一点不一样，非公平操作时，模式不同得到匹配时也需要先将节点入栈，最后将匹配的两个节点一起出栈；公平操作中，得到匹配时，不需要将节点加入队列，直接从队列头部匹配一个节点。

	E transfer(E e, boolean timed, long nanos) {
            SNode s = null; // constructed/reused as needed
            int mode = (e == null) ? REQUEST : DATA;

            for (;;) {
                SNode h = head;
                if (h == null || h.mode == mode) {  // empty or same-mode 栈为空 或者 模式相同
                    if (timed && nanos <= 0) {      // can't wait   非阻塞模式，不等待，直接返回null
                        if (h != null && h.isCancelled())   // 清除被取消的节点
                            casHead(h, h.next);     // pop cancelled node
                        else
                            return null;
                    } else if (casHead(h, s = snode(s, e, h, mode))) {  // 入栈
                        SNode m = awaitFulfill(s, timed, nanos);        // 挂起等待匹配
                        if (m == s) {               // wait was cancelled   如果被取消等待则清除并返回null
                            clean(s);
                            return null;
                        }
                        // 运行到这里说明得到匹配被唤醒，从栈顶将匹配的两个节点一起出栈（修改栈顶指针）
                        if ((h = head) != null && h.next == s)
                            casHead(h, s.next);     // help s's fulfiller
                        return (E) ((mode == REQUEST) ? m.item : s.item);
                    }
                } else if (!isFulfilling(h.mode)) { // try to fulfill 模式不同，尝试匹配
                    if (h.isCancelled())            // already cancelled
                        casHead(h, h.next);         // pop and retry
                    else if (casHead(h, s=snode(s, e, h, FULFILLING|mode))) {   // 将节点入栈，且模式标记为正在匹配中...
                        for (;;) { // loop until matched or waiters disappear
                            SNode m = s.next;       // m is s's match
                            if (m == null) {        // all waiters are gone 说明栈中s之后无元素了，重新进行最外层的循环
                                casHead(s, null);   // pop fulfill node
                                s = null;           // use new node next time
                                break;              // restart main loop
                            }
                            // 将s设置为m的匹配节点，并更新栈顶为m.next，即将s和m同时出栈
                            SNode mn = m.next;
                            if (m.tryMatch(s)) {
                                casHead(s, mn);     // pop both s and m
                                return (E) ((mode == REQUEST) ? m.item : s.item);
                            } else                  // lost match
                                s.casNext(m, mn);   // help unlink
                        }
                    }
                } else {                            // help a fulfiller 如果其他线程正在匹配，则帮助其匹配
                    SNode m = h.next;               // m is h's match
                    if (m == null)                  // waiter is gone
                        casHead(h, null);           // pop fulfilling node
                    else {
                        SNode mn = m.next;
                        if (m.tryMatch(h))          // help match
                            casHead(h, mn);         // pop both h and m
                        else                        // lost match
                            h.casNext(m, mn);       // help unlink
                    }
                }
            }
        }

## awaitFulfill
awaitFulfill函数功能同TransferQueue中同名函数，当无法匹配时加入栈后挂起线程，挂起前满足一定的条件则先自旋。

		SNode awaitFulfill(SNode s, boolean timed, long nanos) {
            final long deadline = timed ? System.nanoTime() + nanos : 0L;
            Thread w = Thread.currentThread();
            int spins = (shouldSpin(s) ?
                         (timed ? maxTimedSpins : maxUntimedSpins) : 0);    // 计算自旋次数
            for (;;) {
                if (w.isInterrupted())
                    s.tryCancel();
                SNode m = s.match;
                // s.match ！= null,有几种情况：
                // (1)因中断或超时被取消了，此时s.match=s
                // (2)匹配成功了，此时s.match=另一个节点
                if (m != null)
                    return m;
                if (timed) {
                    nanos = deadline - System.nanoTime();
                    if (nanos <= 0L) {
                        s.tryCancel();
                        continue;
                    }
                }
                if (spins > 0)
                    spins = shouldSpin(s) ? (spins-1) : 0;
                else if (s.waiter == null)
                    s.waiter = w; // establish waiter so can park next iter
                else if (!timed)
                    LockSupport.park(this);
                else if (nanos > spinForTimeoutThreshold)
                    LockSupport.parkNanos(this, nanos);
            }
        }

		// s是栈顶节点，或者栈顶节点不存在，或者栈顶节点正在匹配
		boolean shouldSpin(SNode s) {
            SNode h = head;
            return (h == s || h == null || isFulfilling(h.mode));
        }

## clean
clean函数功能同TransferQueue中同名函数，当节点因被中断或者等待超时而被取消时，从栈中删除该节点。对应栈顶节点则直接修改栈顶指针的指向，栈中节点则修改其前驱节点的next指针。

		void clean(SNode s) {
            s.item = null;   // forget item
            s.waiter = null; // forget thread

            SNode past = s.next;
            if (past != null && past.isCancelled())     // 找到s之后第一个未被取消的节点
                past = past.next;

            // Absorb cancelled nodes at head
            SNode p;
            while ((p = head) != null && p != past && p.isCancelled()) //如果栈顶被取消了，更改栈顶指针，将取消的节点从栈中删除
                casHead(p, p.next);

            // Unsplice embedded nodes 把p和past之间所有被取消的节点移除栈中
            while (p != null && p != past) {
                SNode n = p.next;
                if (n != null && n.isCancelled())
                    p.casNext(n, n.next);
                else
                    p = n;
            }
        }

# SynchronousQueue常用操作
## void put(E e)
想SynchronousQueue队列中插入数据，如果没有相应的线程接收数据，则put操作一直阻塞，直到有消费者线程接收数据。
	
	public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        if (transferer.transfer(e, false, 0) == null) {
            Thread.interrupted();
            throw new InterruptedException();
        }
    }

可以看到transfer函数的第二个参数timed为false，表示不会超时，一直阻塞。

## boolean offer(E e, long timeout, TimeUnit unit)
带超时时间的插入数据，如果没有相应的线程接收数据，则等待timeout时间，如果等待超时还未传递完数据，则返回false

## boolean offer(E e)
非阻塞操作插入数据，如果没有相应的线程接收数据，则直接返回false

	public boolean offer(E e) {
        if (e == null) throw new NullPointerException();
        return transferer.transfer(e, true, 0) != null;
    }
可以看到transfer函数的第二个参数timed为true，第三个参数时间为0，即超时时间为0，也就是不等待。

## E take()
阻塞操作从SynchronousQueue队列中获取数据，如果没有获取到数据则一直阻塞。

## E poll(long timeout, TimeUnit unit)
带超时时间的从SynchronousQueue队列中获取数据

## E poll()
非阻塞操作从SynchronousQueue队列中获取数据

# 参考链接

[1] [【Java并发编程】17、SynchronousQueue源码分析](https://www.cnblogs.com/wangzhongqiu/archive/2018/03/06/8514910.html)

[2] [SynchronousQueue 源码分析 (基于Java 8)](https://www.jianshu.com/p/95cb570c8187)

[3] [Java阻塞队列SynchronousQueue详解](https://www.jianshu.com/p/376d368cb44f)

[4] [ynchronousQueue(同步队列)](http://shift-alt-ctrl.iteye.com/blog/1840385)

[5] [java并发之SynchronousQueue实现原理](http://blog.csdn.net/yanyan19880509/article/details/52562039)