---
title: '[java1.8源码笔记]AbstractQueuedSynchronizer详解'
date: 2018-02-09 14:20:50
categories: 'java1.8源码笔记'
tags:
---
# 概述
AbstractQueuedSynchronizer又称队列同步器（简称AQS），它是用来构建锁和其他同步组件的基础框架，在ReentrantLock、ReentrantReadWriteLock、ThreadPoolExecutor等类中均有应用。AQS内部通过一个int类型的成员变量state来控制同步状态，当state=0时，则说明没有任何线程占有共享资源的锁，当state=1时，则说明有线程目前正在使用共享变量，其他线程必须加入同步等待队列进行等待，AQS内部通过内部类Node构成FIFO的同步队列来完成线程获取锁的排队工作。

<!--more-->

AQS同时利用内部类ConditionObject构建条件等待队列，当线程调用await()方法后，线程将会加入条件等待队列中，等待其他线程唤醒，当有其他线程调用signal()或signalAll()方法后，线程将会从条件等待队列中转移到同步等待队列中进行锁竞争。

在AQS中涉及到两种FIFO队列，一种是同步等待队列，当线程请求锁而必须等待时将会加入同步等待队列中等待其他线程释放锁；另一种是条件等待队列（一个锁中可以有多个条件等待队列），线程调用await()方法释放锁后将加入条件等待队列。
#内部类介绍
## Node
Node为AQS中构成同步等待队列和条件等待队列的节点，Node用thread域包装了正在等待的线程对象

	static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        // 标记一个等待队列中的节点所代表的线程正处于共享模式下等待锁
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        // 标记一个等待队列中的节点所代表的线程正处于独占模式下等待锁
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        // 表示节点所代表的线程已取消等待锁或条件
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        // 表示该节点的后继节点需要唤醒
        // 节点插入队列时，节点代表的线程睡眠前会将前一个节点的waitStatus置为SIGNAL
        // 当前一个节点释放锁时，如果其waitStatus置为SIGNAL，则会唤醒其后下一个节点线程
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        // 表示节点代表的线程正处于条件等待队列中等待signal信号
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        // 在共享模式下使用，表示同步状态能够无条件向后传播（可能有点难以理解，到后面源码解释中会理解更清晰）
        static final int PROPAGATE = -3;

        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        // 表示节点Node的状态，可能的取值包含：CANCELLED、SIGNAL、CONDITION、PROPAGATE和0
        // 在同步等待队列中的节点初始值为0,在条件等待队列中的节点初始值为CONDITION
        // 在同步等待队列中不同的模式下，有不同的取值
        // 在独占模式下，取值为CANCELLED、SIGNAL、0中之一
        // 在共享模式下，取值为CANCELLED、SIGNAL、PROPAGATE和0中之一
        // 在条件等待队列中，取值为CANCELLED、CONDITION中之一
        volatile int waitStatus;

        /**
         * Link to predecessor node that current node/thread relies on
         * for checking waitStatus. Assigned during enqueuing, and nulled
         * out (for sake of GC) only upon dequeuing.  Also, upon
         * cancellation of a predecessor, we short-circuit while
         * finding a non-cancelled one, which will always exist
         * because the head node is never cancelled: A node becomes
         * head only as a result of successful acquire. A
         * cancelled thread never succeeds in acquiring, and a thread only
         * cancels itself, not any other node.
         */
        // 用于同步等待队列中，指向前驱节点
        volatile Node prev;

        /**
         * Link to the successor node that the current node/thread
         * unparks upon release. Assigned during enqueuing, adjusted
         * when bypassing cancelled predecessors, and nulled out (for
         * sake of GC) when dequeued.  The enq operation does not
         * assign next field of a predecessor until after attachment,
         * so seeing a null next field does not necessarily mean that
         * node is at end of queue. However, if a next field appears
         * to be null, we can scan prev's from the tail to
         * double-check.  The next field of cancelled nodes is set to
         * point to the node itself instead of null, to make life
         * easier for isOnSyncQueue.
         */
        // 用于同步等待等列中，指向后继节点
		// 线程取消等待时，节点next会指向自己
        volatile Node next;

        /**
         * The thread that enqueued this node.  Initialized on
         * construction and nulled out after use.
         */
        // 加入队列中的线程，Node节点为线程的包装
        volatile Thread thread;

        /**
         * Link to next node waiting on condition, or the special
         * value SHARED.  Because condition queues are accessed only
         * when holding in exclusive mode, we just need a simple
         * linked queue to hold nodes while they are waiting on
         * conditions. They are then transferred to the queue to
         * re-acquire. And because conditions can only be exclusive,
         * we save a field by using special value to indicate shared
         * mode.
         */
        // 在条件等待队列中，用于指向下一个节点
        // 在同步等待队列中，用于标记该节点所代表的线程在独占模式下还是共享模式下获取锁
        Node nextWaiter;

        /**
         * Returns true if node is waiting in shared mode.
         */
        final boolean isShared() {
            return nextWaiter == SHARED;
        }

        /**
         * Returns previous node, or throws NullPointerException if null.
         * Use when predecessor cannot be null.  The null check could
         * be elided, but is present to help the VM.
         *
         * @return the predecessor of this node
         */
        final Node predecessor() throws NullPointerException {
            Node p = prev;
            if (p == null)
                throw new NullPointerException();
            else
                return p;
        }

        Node() {    // Used to establish initial head or SHARED marker
        }

        Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }

Node类中域的含义见以上代码注释

# AQS数据结构
如前所述，AQS中涉及到两种FIFO队列
## 同步等待队列
AQS中同步等待队列为一个FIFO双向链表结构，如下图所示：
![](https://i.imgur.com/A83df1d.jpg)
队列包含一个head和tail引用分别指向队列的头部和尾部节点，头节点为一个dummy node，当head和tail引用都指向同一个节点时，表示队列为空。

队列节点Node各个字段详细含义见上面Node内部类介绍，在同步等待队列中有特殊含义的几个域含义如下：

* volatile int waitStatus;
	> 表示节点Node的状态，可能的取值包含：CANCELLED、SIGNAL、CONDITION、PROPAGATE和0
	> 
	> 在同步等待队列中不同的模式下，有不同的取值：
	> 
	> * 在独占模式下，取值为CANCELLED、SIGNAL、0中之一
	> 
	> * 在共享模式下，取值为CANCELLED、SIGNAL、PROPAGATE和0中之一
* volatile Node prev;
	> 指向前驱节点
* volatile Node next;
	> 指向后继节点；线程取消等待时，节点next会指向自己
* Node nextWaiter;
	> 在同步等待队列中，用于标记该节点所代表的线程在独占模式下还是共享模式下获取锁

## 条件等待队列
AQS中的条件等待队列为一个FIFO单向链表，如下图所示：
![](https://i.imgur.com/2qstinh.jpg)
队列包含一个firstWaiter和lastWaiter引用分别指向队列的头部和尾部节点，没有dummy node，当firstWaiter和lastWaiter指向null时，表示队列为空。

在条件等待队列中有特殊含义的几个域含义如下：

* volatile int waitStatus;
	> 在条件等待队列中，取值为CANCELLED、CONDITION中之一
* Node nextWaiter;
	> 指向下一个节点

# 独占模式下锁获取与释放
## 独占模式获取锁
线程获取锁过程：线程首先尝试获取锁，如果获取成功则直接返回；否则将线程封装成Node节点，加入同步等待队列尾部；将前驱节点的waitStatus标记为SIGNAL(-1)后线程睡眠，等待前驱节点所代表的线程释放锁后唤醒该线程。

独占模式获取锁的函数包含如下几个：

* public final void acquire(int arg);
	> 独占模式获取锁，忽略中断
* public final void acquireInterruptibly(int arg);
	> 独占模式获取锁，中断时抛出异常
* protected boolean tryAcquire(int arg);
	> 由子类实现
* public final boolean tryAcquireNanos(int arg, long nanosTimeout) throws InterruptedException;
	> 独占模式获取锁，中断时抛出异常，超时返回false

### acquire函数
独占模式下线程获取锁的方法为acquire()函数，该函数忽略中断，即线程在aquire过程中，中断此线程是无效的。

	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

（1）首先调用tryAcquire函数，线程尝试获取锁，如果获取成功则直接返回。tryAcquire函数在AbstractQueuedSynchronizer源码中默认会抛出一个异常，即需要子类去重写此函数完成自己的逻辑，在ReentrantLock等类中将会看到此方法的实现。

（2）如果获取失败，则调用addWaiter函数，将线程封装成一个Node节点并插入同步等待队列尾部；Node.EXCLUSIVE代表独占模式。

（3）接着调用acquireQueued函数，将前驱节点的waitStatus标记为SIGNAL后睡眠，等待前驱节点释放锁后唤醒，被唤醒后则继续尝试获取锁。

（4）如果线程睡眠过程中产生中断，则调用selfInterrupt函数让线程自我中断一下，设置中断标志，将中断传递给外层。

下面看看addWaiter函数的具体实现

	// 此处在尾部插入node时，先设置node的prev，再CAS修改队列tail指向，修改成功再设置前一个节点的next域
    // 意味着在队列中，如果某个node的prev!=null，并不一定表示node已经成功插入队列中，
    // 如果某个node的前一个节点的next!=null，则该node一定位于队列中。
    private Node addWaiter(Node mode) {
		// 将线程封装为Node节点
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            // CAS操作在队列尾部插入封装线程的Node节点
            if (compareAndSetTail(pred, node)) {
                pred.next = node;   // 插入成功后，修改前一个节点的next指向当前节点
                return node;
            }
        }
        enq(node);
        return node;
    }
	
	private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize 第一次插入队列时需要先初始化
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

此函数先将获取锁的线程封装成Node节点；接着判断同步等待队列是否初始化，如果已初始化，则直接CAS操作将Node节点插入队列尾部，如果插入成功则直接返回；否则调用enq函数在for循环中CAS操作插入队列；enq函数中先判断队列是否初始化，如果未初始化，则先初始化队列。

再看看acquireQueued函数实现

	final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                // 获取前驱节点，如果前驱节点为同步等待队列的头head，则尝试获取锁
                final Node p = node.predecessor();
                /**
                 *每次线程获取锁成功时，会将包含当前线程的Node节点设置为队列的head（即下面调用的setHead函数）
                 *所以当前驱节点等于head时，说明前驱线程为当前正拥有锁，或者刚刚释放锁并且唤醒了当前节点
                 *所以此时调用 tryAcquire尝试一下能否获取锁
                 */
                if (p == head && tryAcquire(arg)) {
                    setHead(node);  //设置成功获取锁的node节点为队列头head
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
				// 如果前驱节点不是队列head或者获取锁失败，，则设置前驱节点waitStatus为SIGNAL，并睡眠
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

（1）acquireQueued函数在一个for循环中尝试获取锁，检查node的前驱节点是否为队列头head，如果是则尝试获取锁，如果获取锁成功，则将当前node设为队列头head；

（2）如果前驱节点不是队列head或者获取锁失败，则设置前驱节点waitStatus为SIGNAL，并睡眠

（3）如果获取锁过程中出错，则调用cancelAcquire取消线程获取锁

（4）如果线程睡眠过程中，产生了中断，记录中断返回给调用函数，调用函数会让线程自我中断一下，设置中断标志，将中断传递给外层。

调用shouldParkAfterFailedAcquire函数设置前驱节点的waitStatus为SIGNAL

	private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }

shouldParkAfterFailedAcquire方法的作用是：

* 确定线程是否需要park;
* 跳过被取消的结点;
* 设置前继的waitStatus为SIGNAL.

其实应该理解，该函数在确定线程是否需要park()睡眠前，可能会在for循环中调用进入好几次，
该函数分为如下三步：

（1）如果正在获取锁的线程其前驱节点的waitStatus==SIGNAL，即前驱已经设置好，直接睡眠当前线程，等待前驱释放锁时唤醒；什么情况下会设置好了呢，在第三步，也有可能是在被唤醒后，在竞争锁的过程中失败

（2）如果前驱节点的waitStatus>0，即已经被取消，则循环检查跳过所有已经被取消的线程，其实这一步应该重新获取新的前驱pred节点的waitStatus，然后检查waitStatus是否等于SIGNAL，如果是返回true，否则设置为SIGNAL，但是这样导致整个if语句中包含了很多本函数中（1）（3）步已经有的重复冗余代码

（3）else语句中将前驱节点设置为SIGNAL，其实这里完全可以检查一下是否设置成功，如果设置成功则返回true，将线程睡眠，一来多了一步冗余的检查，二来为了在睡眠前再次尝试获取一下锁，确认在睡眠前无法获取到锁；

所以通常情况下，前驱节点没有设置SIGNAL，进入第三步，然后第二次进入该函数，进入第一步，所以至少进入该函数两次；如果前驱节点已经被取消，则要进入三次该函数。

如果前驱节点的waitStatus为SIGNAL，则调用parkAndCheckInterrupt函数睡眠当前线程，等待前驱节点唤醒当前线程。被唤醒后，继续进入acquireQueued函数的for循环重新尝试获取锁。

	private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
		// 线程睡眠后，可能因为被其他线程唤醒，也可能是因为被中断而醒过来
        // 返回是否被中断
        return Thread.interrupted();
    }
parkAndCheckInterrupt函数返回在睡眠过程中是否被中断过。

以上就是独占模式下获取锁的逻辑。

### acquireInterruptibly函数
acquireInterruptibly表示可中断获取锁，获取锁的逻辑与acquire基本一致，唯一的不同是如果线程睡眠时，是因为被中断而醒过来则直接抛出中断异常

	public final void acquireInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (!tryAcquire(arg))
            doAcquireInterruptibly(arg);
    }
（1）先检查一下线程是否被中断过，如果是则直接抛出中断异常；

（2）调用tryAcquire尝试获取锁，如果获取成功则直接返回；

（3）如果获取失败，则调用doAcquireInterruptibly，将线程封装成节点加入同步等待队列，然后睡眠

	private void doAcquireInterruptibly(int arg)
        throws InterruptedException {
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
					// 与acquire唯一的不同之处是，如果线程因为被中断而醒过来则抛出中断异常
                    throw new InterruptedException();
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

### tryAcquireNanos函数
tryAcquireNanos尝试以独占模式获取锁，如果被中断则抛出中断异常，如果在指定时间内没有获取到锁，则表示获取锁超时，返回false

获取锁的逻辑与acquire基本一致，不同之处为可被中断，以及设置有超时时间

	public final boolean tryAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        return tryAcquire(arg) ||
            doAcquireNanos(arg, nanosTimeout);
    }

	private boolean doAcquireNanos(int arg, long nanosTimeout)
            throws InterruptedException {
        if (nanosTimeout <= 0L)
            return false;
        // 计算超时的最后期限
        final long deadline = System.nanoTime() + nanosTimeout;
        // 封装成Node节点插入队列尾部
        final Node node = addWaiter(Node.EXCLUSIVE);
        boolean failed = true;
        try {
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return true;
                }
                nanosTimeout = deadline - System.nanoTime();
                if (nanosTimeout <= 0L)
                    return false;   // 如果已经超过设置的超时期限，获取锁超时
                if (shouldParkAfterFailedAcquire(p, node) &&
                    nanosTimeout > spinForTimeoutThreshold)
                    // 如果设置了前一个节点waitStatus为SIGNAL，且超时时间大于1000ns，则睡眠
                    LockSupport.parkNanos(this, nanosTimeout);
                if (Thread.interrupted())
                    throw new InterruptedException();   // 如果因为被中断而从睡眠中醒过来，则抛出中断异常
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

## 独占模式释放锁
释放锁的过程为：线程释放占有的锁，并唤醒后继线程
### release函数

	public final boolean release(int arg) {
    	// tryReease由子类实现，通过设置state值来达到同步的效果。
    	if (tryRelease(arg)) {
        	Node h = head; //每个队列中的线程在获取到锁以后，会将队列中包含该线程的Node节点设置为head
							//所以head指向的Node即为当前占有锁的线程，也就是当前正在进行释放锁操作的线程
       	 	// waitStatus为0说明是初始化的空队列，说明该节点后面没有需要唤醒的线程，即没有在排队等待获取锁的线程
			//每个在队列中排队等待获取锁的线程，在调用park()睡眠前，会先将队列中其前驱节点的waitStatus设置为SIGNAL(-1)
        	if (h != null && h.waitStatus != 0)
            	// 唤醒后续的结点
            	unparkSuccessor(h);
        	return true;
    	}
    	return false;
	} 

	private void unparkSuccessor(Node node) {
        /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
        /**
         * 在独占模式下，waitStatus<0此时为SIGNAL，将SIGNAL标志清除
         */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
        /**
         * 如果node.next.waitStatus > 0，表示该node.next线程取消了获取锁，则从队列尾部遍历，找到队列中第一个没有取消的线程
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            //这里并没有从头向尾寻找，而是相反的方向寻找，为什么呢？
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        // 如果存在，唤醒该后继线程
        if (s != null)
            LockSupport.unpark(s.thread);
    }

在unparkSuccessor中为什么是从尾部往前找第一个没有取消的线程？因为在同步等待队列中的节点随时有可能被中断或者等待超时而被取消，被取消的节点的waitStatus设置为CANCELLED，并且其next引用指向自己（详细见cancelAcquire函数实现）。

CANCELLED结点的next指针为什么要指向它自己，为什么不指向真正的next结点？为什么不为NULL？第一个问题的答案是这种被CANCELLED的结点最终会被GC回收，如果指向next结点，GC无法回收。对于第二个问题的回答，JDK中next域的注释中有这么一句话： The next field of cancelled nodes is set to point to the node itself instead of null, to make life easier for isOnSyncQueue.大至意思是为了使isOnSyncQueue方法更新简单。isOnSyncQueue方法判断一个结点是否在同步队列，实现如下：

	final boolean isOnSyncQueue(Node node) {
		// node.waitStatus == Node.CONDITION表示还在条件等待队列中
        if (node.waitStatus == Node.CONDITION || node.prev == null)
            return false;
        if (node.next != null) // If has successor, it must be on queue
            return true;
        /*
         * node.prev can be non-null, but not yet on queue because
         * the CAS to place it on queue can fail. So we have to
         * traverse from tail to make sure it actually made it.  It
         * will always be near the tail in calls to this method, and
         * unless the CAS failed (which is unlikely), it will be
         * there, so we hardly ever traverse much.
         */
		// 可能是在队列尾部（node.next==null），也可能不在队列中，只能从尾部往前找
        return findNodeFromTail(node);
    }

在addWaiter中将节点node插入队列时，先设置node.prev指向队列当前尾部节点，再设置node节点为队列的尾部tail，如果设置成功，则将旧的尾部节点的next指向节点node。所以如果node.prev！=null，并不一定说明node已经在队列中，因为CAS设置node为队列的尾部tail操作有可能失败；但是如果node.prev==null，则说明肯定不在队列中，所以isOnSyncQueue可以直接返回false。

同理，node.next!=null，说明有后继节点，那该节点肯定也在队列中；对于被取消的节点，节点虽然被取消，也还在队列中，所以其next指向自身可以加快查找效率；所以如果节点取消获取锁时，将node.next设为null，就只能进一步调用findNodeFromTail函数，从尾部往前查找，进一步确认node是否在队列中；

	private boolean findNodeFromTail(Node node) {
        Node t = tail;
        for (;;) {
            if (t == node)
                return true;
            if (t == null)
                return false;
            t = t.prev;
        }
    }

从尾部往前查找node是否在同步等待队列中，这里从尾部开始查找是同样的原因，因为可能存在被取消的节点；另一方面，在isOnSyncQueue中进入该函数，表示要查找的node可能在队列尾部或不存在，所以如果存在，从尾部查找也刚好可以更快速的找到。

## 线程取消获取锁
线程在获取锁过程中，可能因为出错、被中断或超时而取消获取锁。
### cancelAcquire函数

	// 取消线程获取锁，清除因中断/超时而放弃获取锁的线程节点
	private void cancelAcquire(Node node) {
        // Ignore if node doesn't exist
        if (node == null)
            return;

        node.thread = null;     // 线程引用清空

        // Skip cancelled predecessors
        Node pred = node.prev;
        while (pred.waitStatus > 0)
            node.prev = pred = pred.prev;   //若前驱节点是CANCELLED，则跳过继续往前找

        // predNext is the apparent node to unsplice. CASes below will
        // fail if not, in which case, we lost race vs another cancel
        // or signal, so no further action is necessary.
        Node predNext = pred.next;      // 如果前驱节点中没有CANCELLED节点，此处preNext即为node本身

        // Can use unconditional write instead of CAS here.
        // After this atomic step, other Nodes can skip past us.
        // Before, we are free of interference from other threads.
        node.waitStatus = Node.CANCELLED;   // 设置waitStatus值为CANCELLED，标记节点已取消

        // If we are the tail, remove ourselves.
        // 如果被取消的node是尾部节点，则设置tail指针指向前驱节点，并且设置前驱节点的next指针为null
        if (node == tail && compareAndSetTail(node, pred)) {
            compareAndSetNext(pred, predNext, null);
        } else {
            // If successor needs signal, try to set pred's next-link
            // so it will get one. Otherwise wake it up to propagate.
            int ws;
            if (pred != head &&
                ((ws = pred.waitStatus) == Node.SIGNAL ||       //如果前驱已经设置好为SIGNAL 或者 若未设置好，且waitStatus<=0，则设置前驱节点waitStatus为SIGNAL
                 (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
                pred.thread != null) {
                Node next = node.next;
                if (next != null && next.waitStatus <= 0)
                    compareAndSetNext(pred, predNext, next);    //修改前驱节点的next指向
            } else {
                // 如果前驱节点是head，或者CAS设置前驱节点waitStatus=SIGNAL失败，或者前驱节点thread==null（可能因为取消，可能因为变成head）
                // 则唤醒
                unparkSuccessor(node);
            }

            node.next = node; // help GC    设置next指向自身
        }
    }

被取消的节点最终在同步等待队列中的状态如下图所示：
![](https://i.imgur.com/GnAahj2.jpg)

# 共享模式下锁获取与释放
## 共享模式获取锁
线程共享模式下获取锁过程：线程首先尝试获取锁，如果获取成功则直接返回；否则将线程封装成Node节点，加入同步等待队列尾部；将前驱节点的waitStatus标记为SIGNAL(-1)后线程睡眠，等待前驱节点所代表的线程释放锁后唤醒该线程。共享与独占获取锁的主要区别在于， 在共享方式下获取锁成功会判断是否需要继续唤醒后继节点继续获取共享的锁。

共享模式下获取锁主要包含以下几个函数：

* public final void acquireShared(int arg)；
	> 不响应中断获取锁
* public final void acquireSharedInterruptibly(int arg)；
	> 响应中断获取锁，线程在获取锁过程中若被中断则抛出中断异常
* public final boolean tryAcquireSharedNanos(int arg, long nanosTimeout)；
	> 响应中断及超时获取锁，线程获取锁过程中若被中断则抛出异常，若获取超时，则返回false，获取锁失败

### acquireShared函数

	public final void acquireShared(int arg) {
        if (tryAcquireShared(arg) < 0)
            doAcquireShared(arg);
    }

	private void doAcquireShared(int arg) {
		// 加入队列的节点标记为Node.SHARED，标记为共享节点
        final Node node = addWaiter(Node.SHARED);
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head) {
                    int r = tryAcquireShared(arg);
                    if (r >= 0) {
						// 与独占模式下获取锁唯一的区别就在下面这个setHeadAndPropagate函数
                        setHeadAndPropagate(node, r);
                        p.next = null; // help GC
                        if (interrupted)
                            selfInterrupt();
                        failed = false;
                        return;
                    }
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

tryAcquireShared由子类实现
setHeadAndPropagate函数实现如下所示：

	private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        /*
         * 尝试唤醒后继的结点：<br />
         * propagate > 0说明许可还有能够继续被线程acquire;<br />
         * 或者 之前的head被设置为PROPAGATE(PROPAGATE可以被转换为SIGNAL)说明需要往后传递;<br />
         * 或者为null,我们还不确定什么情况。 <br />
         * 并且 后继结点是共享模式或者为如上为null。
         * <p>
         * 上面的检查有点保守，在有多个线程竞争获取/释放的时候可能会导致不必要的唤醒。<br />
         * http://www.cnblogs.com/zhanjindong/p/java-concurrent-package-aqs-AbstractQueuedSynchronizer.html
         */
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            // 后继结是共享模式或者s == null（不知道什么情况）
            // 如果后继是独占模式，那么即使剩下的许可大于0也不会继续往后传递唤醒操作
            // 即使后面有结点是共享模式。
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }

setHeadAndPropagate设置队列的头，并且判断如果后继节点为共享模式，则继续唤醒后继节点。

	//当该节点是队列中唯一一个节点时，状态为0，直接通过CAS操作将状态改成PROPAGATE
    // （之所以要修改而不保持为0是因为，若下一个节点是shared的节点，则可以直接获取锁。当下一个节点获取锁成功，
    // 调用setHeadAndPropagate，该方法中第一个if可能propagate为0(N)，但是waitStatus状态小于0（Y），则还是可以直接进入外层if中），
    // 否则，该节点状态为SIGNAL，CAS修改状态为0，之后调用unparkSuccessor唤醒后继节点（独占的和共享的节点都要唤醒）。
	private void doReleaseShared() {
        /*
         * Ensure that a release propagates, even if there are other
         * in-progress acquires/releases.  This proceeds in the usual
         * way of trying to unparkSuccessor of head if it needs
         * signal. But if it does not, status is set to PROPAGATE to
         * ensure that upon release, propagation continues.
         * Additionally, we must loop in case a new node is added
         * while we are doing this. Also, unlike other uses of
         * unparkSuccessor, we need to know if CAS to reset status
         * fails, if so rechecking.
         */
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                //首节点的状态是0，则通过CAS设置为PROPAGATE，表示如果新连接在当前节点后的node的状态如果是shared，则无条件获取锁。
                //考虑这样一种情况，当前节点node是一个shared状态的节点，并且是tail,则当前节点node的状态为0。
                // 当node成功获取锁后，调用setHeadAndPropagate设置当前节点为head后，
                // （此时node的前一个节点waitstatus<0,可以进入外层的if语句，next为空，则可以进入内层if语句）
                // 调用doReleaseShared进行释放锁后的处理，此时当前节点就可以通过下面代码将状态设置为PROPAGATE了。
                // 若此后有一个新的shared状态的node添加到队尾，则可以直接获取到锁，再次进入setHeadAndPropagate时，
                // 当前节点的waitStatus为PROPAGATE（propagate可能等于0），保证能进入外层的if语句。
                //PROPAGATE，表示下一次（当前节点获得锁时下一个节点还没有添加到队列中），shared状态节点将会无条件传播共享状态。
                //参考链接 http://blog.csdn.net/u011470552/article/details/76483532
                else if (ws == 0 &&
                         !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head)                   // loop if head changed
                break;
        }
    }

## 共享模式释放锁
### releaseShared函数
	
	public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }

tryReleaseShared由子类实现

# Condition的实现原理
## Object对象监视器方法与Condition概述
在java并发编程中，线程间的通信与协作有两种常见的方式：一种是利用Object对象的监视器方法，如wait()、notify()；另一种就是利用Condition，主要函数有await()、signal()；

两者的共同之处是调用wait()、await()等函数的线程必须拥有锁。调用Object对象的wait()、notify()等方法的线程必须拥有这个对象的monitor（即锁），因此调用wait()方法必须在同步块或者同步方法中进行（synchronized块或者synchronized方法）。调用Condition的await()和signal()方法，都必须在lock保护之内，就是说必须在lock.lock()和lock.unlock之间才可以使用，Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition() 。

然而，与synchronized的等待唤醒机制相比Condition具有更多的灵活性以及精确性，这是因为notify()在唤醒线程时是随机(同一个锁)，而Condition则可通过多个Condition实例对象建立更加精细的线程控制，对于一个锁，我们可以为多个线程间建立不同的Condition。

## Condition实现原理概述
Condition的具体实现类是AQS的内部类ConditionObject，前面我们分析过AQS中存在两种队列，一种是同步等待队列，一种是条件等待队列，而条件等待队列就相对于Condition而言的。每个Condition对象都对应着一个条件等待队列，也就是说如果一个锁上创建了多个Condition对象（注：使用Condition前必须获得锁），那么也就存在多个条件等待队列。条件等待队列是一个FIFO的队列，在队列中每一个节点都包含了一个线程的引用，而该线程就是Condition对象上等待的线程。

当一个线程调用了await()相关的方法，那么该线程将会释放锁，并构建一个Node节点封装当前线程加入到条件等待队列中进行等待，直到被唤醒、中断、超时才从队列中移出。当一个线程调用了signal()相关的方法，那么该线程会将条件等待队列中的节点转移至同步等待队列中等待获取锁。

## 常用函数介绍
* void await() throws InterruptedException;
	> 可中断的条件等待
* long awaitNanos(long nanosTimeout) throws InterruptedException;
	> 实现定时的条件等待，响应中断
* boolean await(long time, TimeUnit unit) throws InterruptedException;
	> 实现定时的条件等待，响应中断
* boolean awaitUntil(Date deadline) throws InterruptedException;
	> 实现绝对定时的条件等待，响应中断
* void awaitUninterruptibly();
	> 实现不可中断的条件等待
* void signal();
	> 将等待时间最长的线程（如果存在）从此条件的条件等待队列中移动到拥有锁的同步等待队列。
* void signalAll();
	> 将所有线程从此条件的条件等待队列移动到拥有锁的同步等待队列中。

## 源码实现
### await()函数

		public final void await() throws InterruptedException {
            // 线程是否被中断，如果是则抛出异常
            if (Thread.interrupted())
                throw new InterruptedException();
            // 将调用await()的线程封装成Node节点，并加入条件等待队列中
            Node node = addConditionWaiter();
            // 释放线程占有的锁
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            // 如果不在同步等待队列中则睡眠等待
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                // 判断睡眠过程中是否被中断过
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            // 重新获取锁
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            // 在signal函数中将节点转移至同步等待队列中时，会将node.nextWaiter 置为 null
            // node.nextWaiter != null表示等待被取消，则清楚条件等待队列中的已取消节点
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            // 处理异常
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }

await()方法的主要内容为：将线程封装为Node节点加入条件等待队列中；释放线程占有的锁，并唤醒同步等待队列中后继节点的线程；挂起等待，直到出现在同步等待队列中，然后重新尝试获取锁

### transferAfterCancelledWait函数

	// 该函数的功能为：线程等待条件过程中被中断后，调用此函数将节点从条件等待队列转移至同步等待队列
    // 同时返回是调用signal前中断还是调用signal后被中断
	final boolean transferAfterCancelledWait(Node node) {
        // 如果节点状态为CONDITION，则说明还在条件等待队列中，则转移至同步等待队列，且说明为调用signal函数前被中断
        if (compareAndSetWaitStatus(node, Node.CONDITION, 0)) {
            enq(node);
            return true;
        }
        /*
         * If we lost out to a signal(), then we can't proceed
         * until it finishes its enq().  Cancelling during an
         * incomplete transfer is both rare and transient, so just
         * spin.
         */
        // 否则说明是调用signal函数后被中断，等待节点被执行signal函数的线程转移至同步等待队列
        while (!isOnSyncQueue(node))
            Thread.yield();
        return false;
    }

调用该函数的前提是等待条件过程中已经被中断

unlinkCancelledWaiters函数将已经取消等待的线程节点从条件等待队列中删除，比较常见的链表指针操作，自己动手画画很容易理解。

### signal()函数

		public final void signal() {
            // 检查当前线程是否占有独占锁的线程
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }

		private void doSignal(Node first) {
            // 每次通知一个线程，将条件等待队列中的第一个节点删除，并加入同步等待队列中
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }
	
	final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
		// 如果节点的状态不为CONDITION，则说明是CANCELLED，已取消
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node); //加入同步等待队列，并返回前一个节点
        int ws = p.waitStatus;
        // 如果前一个节点已经取消，或者设置前一个节点状态为SIGNAL失败，则直接唤醒刚刚加入同步等待队列的这个线程
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }

signal()函数的主要内容为：将条件等待队列中的节点删除，并加入同步等待队列中

signal()为唤醒条件等待队列中第一个等待的线程，signalAll()则为唤醒条件等待队列中的所有线程

# 设计模式
AQS 采用了标准的模版方法设计模式，模板方法模式是一种基于继承的代码复用，通过使用模板方法模式，可以将一些复杂流程的实现步骤封装在一系列基本方法中，在抽象父类中提供一个称之为模板方法的方法来定义这些基本方法的执行次序，而通过其子类来覆盖某些步骤，从而使得相同的算法框架可以有不同的执行结果。

在AQS中，以acquire函数为例，acquire函数封装了线程获取锁的一系列行为

	public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

在acquire中封装了tryAcquire、acquireQueued、addWaiter等基本方法的执行次序，其中tryAcquire在AQS中只提供了一个默认的抛出异常操作，具体实现由子类实现，从而不同的子类实现可以在相同的框架下实现不同的行为。

以ReentrantLock公平模式下和非公平模式下获取锁为例，在内部类FairSync和NonfairSync中分别实现了相应的tryAcquire函数

# 参考链接
[1] [http://blog.csdn.net/javazejian/article/details/75043422](http://blog.csdn.net/javazejian/article/details/75043422)

[2] [https://www.cnblogs.com/dolphin0520/p/3920385.html](https://www.cnblogs.com/dolphin0520/p/3920385.html)

[3] [http://zhanjindong.com/2015/03/15/java-concurrent-package-aqs-AbstractQueuedSynchronizer](http://zhanjindong.com/2015/03/15/java-concurrent-package-aqs-AbstractQueuedSynchronizer)

[4] [http://blog.csdn.net/hustspy1990/article/details/77941102](http://blog.csdn.net/hustspy1990/article/details/77941102)

[5] [http://guochenglai.com/2016/06/06/java-concurrent5-aqs-code-analysis/](http://guochenglai.com/2016/06/06/java-concurrent5-aqs-code-analysis/)

[6] [http://blog.csdn.net/yanbober/article/details/45501715](http://blog.csdn.net/yanbober/article/details/45501715)