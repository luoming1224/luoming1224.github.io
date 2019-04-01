---
title: '[java1.8源码笔记]ReentrantReadWriteLock详解'
date: 2018-03-01 10:10:31
categories: 'java1.8源码笔记'
tags: [ReentrantReadWriteLock, 读写锁, 锁降级]
---
# 概述
在并发场景中用于解决线程安全的问题，我们几乎会高频率的使用到独占式锁，通常使用java提供的关键字synchronized或者concurrents包中实现了Lock接口的[ReentrantLock](https://luoming1224.github.io/2018/02/28/[java1.8%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0]ReentrantLock%E8%AF%A6%E8%A7%A3/)。它们都是独占式获取锁，也就是在同一时刻只有一个线程能够获取锁。而在一些业务场景中，大部分只是读数据，写数据很少，如果仅仅是读数据的话并不会影响数据正确性（出现脏读），而如果在这种业务场景下，依然使用独占锁的话，很显然这将是出现性能瓶颈的地方。针对这种读多写少的情况，java还提供了另外一个实现Lock接口的ReentrantReadWriteLock(读写锁)。

ReentrantReadWriteLock，读写锁或者重入读写锁，它维护了一个读锁和一个写锁，它允许同一时刻被多个读线程访问，而此时写线程不能获取到锁，并且当写线程获取到锁时后续的读写线程都将被阻塞不能获取到锁。读写锁保证了写操作对后续的读操作的可见性。同时ReentrantReadWriteLock还支持重入，公平性选择以及锁的降级。

<!--more-->

ReentrantReadWriteLock主要特点如下：

（1）重入性：同一线程获取读锁后能够再次获取读锁；同一线程获取写锁之后能够再次获取写锁，同时也能够获取读锁

（2）公平性选择：支持非公平性（默认）和公平的锁获取方式，吞吐量还是非公平优于公平；

（3）锁降级：获取写锁，获取读锁之后再释放写锁，写锁能降级为读锁（线程同时获取读写锁时, 必须先获取 writeLock, 再获取 readLock，反过来会直接导致死锁）

（4）锁的获取支持线程中断, 且writeLock 中支持 Condition 

（5）需要获取读锁时，

	* 当前不存在锁，或者只有读锁，或者占有写锁的线程为当前线程时才能成功
	* 当存在写锁排队时，写锁后面的读锁均需进入同步队列排队，除非当前线程已经获得读锁且还未释放
（6）线程获取写锁的前提是，没有其他线程的读锁和写锁

# 构造函数
ReentrantReadWriteLock 支持公平与非公平模式, 构造函数中可以通过指定的值传递进去

	public ReentrantReadWriteLock() {
        this(false);
    }

    /**
     * Creates a new {@code ReentrantReadWriteLock} with
     * the given fairness policy.
     *
     * @param fair {@code true} if this lock should use a fair ordering policy
     */
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

默认构造函数创建非公平模式读写锁，带参数的构造函数中传入true创建公平模式读写锁，传入false创建非公平模式读写锁。

# 内部类
ReentrantReadWriteLock中有Sync、NonfairSync、FairSync、ReadLock、WriteLock等内部类，其相互依赖关系如下两图所示：

![](https://i.imgur.com/qOoQa0Y.png) 

![](https://i.imgur.com/cJqr0m4.png)

# Sync类
Sync类继承于AbstractQueuedSynchronizer，是NonfairSync和FairSync的基类，是ReentrantReadWriteLock 的核心类

## 读写锁计数存放

	/**状态的高16位用作读锁，低16位用作写锁，所以无论是读锁还是写锁最多只能被持有65535次*/
        static final int SHARED_SHIFT   = 16;
        static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
        static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
        static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

        /** Returns the number of shared holds represented in count  */
        static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
        /** Returns the number of exclusive holds represented in count  */
        static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

读写锁的获取次数存放在 AQS 里面的state上, state的高 16 位存放 readLock 获取的次数, 低16位存放 writeLock 获取的次数.

每个线程获得读锁的次数用内部类HoldCounter保存，并且存储在ThreadLocal里面

		//用于记录每个线程获取到的读锁的数量
        //使用id和count记录
        static final class HoldCounter {
            int count = 0;
            // Use id, not reference, to avoid garbage retention
            final long tid = getThreadId(Thread.currentThread());
        }

        /**
         * ThreadLocal subclass. Easiest to explicitly define for sake
         * of deserialization mechanics.
         */
        //这里使用了ThreadLocal为每个线程都单独维护了一个HoldCounter来记录获取的读锁的数量
        static final class ThreadLocalHoldCounter
            extends ThreadLocal<HoldCounter> {
            public HoldCounter initialValue() {
                return new HoldCounter();
            }
        }
		
		private transient ThreadLocalHoldCounter readHolds;
		private transient HoldCounter cachedHoldCounter;
		private transient Thread firstReader = null;
        private transient int firstReaderHoldCount;

* readHolds
	> ThreadLocal变量，保存本线程持有的读锁的数量
* cachedHoldCounter
	> 保存最后一个获得读锁的线程的HoldCounter，因为大部分时候，下一个释放锁的线程都是最后一个获取所得线程，减少从ThreadLocalHoldCounter调用get()获取线程局部值的次数，提高程序效率
* firstReader 、 firstReaderHoldCount
	> 记录第一次获取锁的线程, 及重入的次数
	> 
	> 如果业务场景中只有一个线程获取readLock，还用ThreadLocal的话就有点浪费，通过reference 来获取数据效率更高

写锁 writeLock 的获取是由 state 的低16位 及 aqs中的exclusiveOwnerThread 来进行记录

# 写锁的获取
入口为内部类WriteLock的lock()方法，在lock()方法中调用的acquire函数为AQS中实现，在acquire函数中会调用tryAcquire函数尝试获取锁，如果获取成功则返回，失败则加入同步等待队列。tryAcquire函数在内部类Sync中实现。

		protected final boolean tryAcquire(int acquires) {
            /*
             * Walkthrough:
             * 1. If read count nonzero or write count nonzero
             *    and owner is a different thread, fail.
             * 2. If count would saturate, fail. (This can only
             *    happen if count is already nonzero.)
             * 3. Otherwise, this thread is eligible for lock if
             *    it is either a reentrant acquire or
             *    queue policy allows it. If so, update state
             *    and set owner.
             */
            /**
             * 在写锁获取锁之前先判断是否有读锁存在，只有在读锁不存在的情况下才能去获取写锁（可能有多个线程获取了读锁,为了保证写的操作对所有的读都可见）。
             */
            Thread current = Thread.currentThread();
            int c = getState();
			// 获取写锁（排它锁）的数量
            int w = exclusiveCount(c);
            if (c != 0) {
                // (Note: if c != 0 and w == 0 then shared count != 0)
                // (1)c!=0,w==0,锁数量不为0且写锁数量为0，说明读锁存在,获取写锁必须等待
                // (2)w != 0 && current != getExclusiveOwnerThread() 表示存在写锁，并且其他线程获取了写锁。
                if (w == 0 || current != getExclusiveOwnerThread())
                    return false;
                if (w + exclusiveCount(acquires) > MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                // Reentrant acquire
                //执行到这里，说明存在写锁，且由当前线程持有
                // 重入计数
                setState(c + acquires);
                return true;
            }
			// 执行到这里，说明当前不存在任何读锁或写锁（c == 0）
            if (writerShouldBlock() ||
                !compareAndSetState(c, c + acquires))
                return false;
			// 获取写锁成功，则记录拥有写锁的线程
            setExclusiveOwnerThread(current);
            return true;
        }
	
tryAcquire主要逻辑为：在获取写锁时，

如果存在读锁，则该线程必须加入同步等待队列中等待；如果存在写锁且由其他线程持有，也必须加入同步等待队列中，如果写锁由当前线程本身持有，则可以直接获得并支持重入，并且增加写锁的数量state值。

如果当前不存在任何读锁或者写锁，则在公平模式下必须队列中没有其他线程排队才尝试调用CAS操作获取锁，在非公平模式下则直接调用CAS操作尝试获取锁。

所以writerShouldBlock()在公平和非公平模式下有不同的实现，在非公平模式下直接返回false，表示不需要阻塞，可以直接尝试获取锁

	static final class NonfairSync extends Sync {
        final boolean writerShouldBlock() {
            return false; // writers can always barge
        }
    }

在公平模式下，则需要检查同步等待队列中是否有其他线程在排队，如果没有才能尝试获取锁

	static final class FairSync extends Sync {
        final boolean writerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }

hasQueuedPredecessors函数的分析见[ReentrantLock](https://luoming1224.github.io/2018/02/28/[java1.8%E6%BA%90%E7%A0%81%E7%AC%94%E8%AE%B0]ReentrantLock%E8%AF%A6%E8%A7%A3/)中FairSync类的分析。

# 写锁的释放
入口为内部类WriteLock的unlock()函数，unlock()函数中调用的release函数在AQS中实现，在release中会调用tryRelease函数释放锁，然后唤醒同步等待队列中的后继线程。

tryRelease函数在内部类Sync中实现

		protected final boolean tryRelease(int releases) {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            // 写锁的数量由state值的低16位保存，所以直接相减即可
            int nextc = getState() - releases;
            // 计算释放后写锁的数量，如果为0,则可以唤醒后继线程
            boolean free = exclusiveCount(nextc) == 0;
            if (free)
                setExclusiveOwnerThread(null);
            setState(nextc);
            return free;
        }
tryRelease的主要逻辑为：检查写锁是否由当前线程持有，如果不是则返回异常；减少state值中写锁的数量，并且判断释放后写锁数量是否为0，如果是则返回true，表示可以唤醒后继线程。

# 读锁的获取
读锁的获取入口为内部类ReadLock的lock()函数，lock()函数中调用的acquireShared函数在AQS中实现，在acquireShared中会调用tryAcquireShared尝试获取读锁，获取成功则返回，获取失败则将线程加入同步等待队列中。

tryAcquireShared函数在内部类Sync中实现

	protected final int tryAcquireShared(int unused) {
            
            Thread current = Thread.currentThread();
            int c = getState();
            /**持有写锁的线程可以获得读锁*/
            // 当写锁数量不为0,且写锁由其他线程持有时返回-1,表示需要加入同步等待队列中进行等待
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return -1;
            //执行到这里表明：没有写锁，或者写锁由当前线程持有
            int r = sharedCount(c);
            if (!readerShouldBlock() &&
                r < MAX_COUNT &&
                compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    // 记录第一个获取读锁的线程
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    // 更新最后一个获取读锁的线程和本地变量的内容
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current))
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return 1;
            }
            return fullTryAcquireShared(current);
        }

tryAcquireShared的主要逻辑为：

（1）如果存在写锁且由其他线程持有，则获取读锁的线程必须加入同步等待队列中；

（2）如果不存在写锁或者写锁由当前线程持有，则判断是否需要阻塞等待，如果不需要且读锁数量没有达到最大值，则尝试CAS操作修改state值获取锁，如果获取成功则更新本线程持有的读锁数量

readerShouldBlock判断是否需要阻塞，在公平模式和非公平模式下有不同的实现

在**非公平模式**下，如果同步等待队列中有获取写锁的线程在排队，则获取读锁的线程必须加入队列等待，否则可以直接尝试获取读锁。

	static final class NonfairSync extends Sync {
        final boolean readerShouldBlock() {
            return apparentlyFirstQueuedIsExclusive();
        }
    }
	
	final boolean apparentlyFirstQueuedIsExclusive() {
        Node h, s;
        return (h = head) != null &&
            (s = h.next)  != null &&
            !s.isShared()         &&
            s.thread != null;
    }
apparentlyFirstQueuedIsExclusive判断同步等待队列中第一个线程是否为获取写锁的线程。

在**公平模式**下，如果同步等待队列中有其他线程在排队，在获取读锁的线程必须加入队列中等待。

	static final class FairSync extends Sync {
        final boolean readerShouldBlock() {
            return hasQueuedPredecessors();
        }
    }

最后，如果获取读锁失败则调用fullTryAcquireShared函数进一步尝试获取。fullTryAcquireShared主要处理以下三种情况：

（1）readerShouldBlock()返回true，即需要排队等待（公平锁而言，是sync Queue中有节点；非公平锁而言，是head.next是获取writeLock的节点）；此时需要处理可重入读锁情况，即当前线程之前已经获得读锁，并且还没有释放，此时线程可以获得读锁

（2）r==MAX_COUNT，读锁数量达到最大值饱和

（3）CAS设置失败，需要在for循环中设置，确保成功

		final int fullTryAcquireShared(Thread current) {
            /*
             * This code is in part redundant with that in
             * tryAcquireShared but is simpler overall by not
             * complicating tryAcquireShared with interactions between
             * retries and lazily reading hold counts.
             */
            HoldCounter rh = null;
            for (;;) {
                int c = getState();
                if (exclusiveCount(c) != 0) {
                    // 写锁存在且由其他线程持有，获取读锁的线程需等待
                    if (getExclusiveOwnerThread() != current)
                        return -1;
                    // else we hold the exclusive lock; blocking here
                    // would cause deadlock.
                } else if (readerShouldBlock()) {   //在需要排队等待时，如果当前线程之前获得了读锁，并且还未释放，当前线程可以继续获取读锁
                    /**
                     * 以下代码为判断当前线程是否获得了读锁并且暂未释放，
                     *（1）firstReader == current，说明当前线程至少持有一个读锁，并且是第一个获得读锁的线程，
                     *   为什么这里不用判断firstReaderHoldCount，因为在tryReleaseShared中，当前firstReaderHoldCount<1时，会设置firstReader = null
                     *（2）获取缓存的线程读锁数量，如果未缓存或者缓存的不是当前线程，则必须去ThreadLocal中取得线程局部值，
                     *   如果rh.count==0，说明当前线程之前未获得读锁，故返回-1，当前线程进入同步队列Sync Queue，
                     *   如果rh.count！=0，说明当前线程之前获得过读锁，且还未释放，所以继续向下执行，通过CAS操作获取读锁
                     *以上的操作唯一的目的就是查看当前线程之前是否获得读锁并且还未释放；
                     *至于先判断firstReader ，再判断cachedHoldCounter，都只是先查看本地缓存，提高效率，因为操作ThreadLocal的get()操作需要一定的时间
                     */
                    // Make sure we're not acquiring read lock reentrantly
                    if (firstReader == current) {
                        // assert firstReaderHoldCount > 0;
                    } else {
                        if (rh == null) {
                            rh = cachedHoldCounter;
                            if (rh == null || rh.tid != getThreadId(current)) {
                                rh = readHolds.get();
                                if (rh.count == 0)
                                    readHolds.remove();
                            }
                        }
                        if (rh.count == 0)
                            return -1;
                    }
                }
                if (sharedCount(c) == MAX_COUNT)
                    throw new Error("Maximum lock count exceeded");
                if (compareAndSetState(c, c + SHARED_UNIT)) {
                    if (sharedCount(c) == 0) {
                        firstReader = current;
                        firstReaderHoldCount = 1;
                    } else if (firstReader == current) {
                        firstReaderHoldCount++;
                    } else {
                        if (rh == null)
                            rh = cachedHoldCounter;
                        if (rh == null || rh.tid != getThreadId(current))
                            rh = readHolds.get();
                        else if (rh.count == 0)
                            readHolds.set(rh);
                        rh.count++;
                        cachedHoldCounter = rh; // cache for release
                    }
                    return 1;
                }
            }
        }

注：类比ReentrantLock，在ReentrantLock中检查是否重入锁，只需要检查当前获得独占锁的线程是否为当前线程；在ReentrantReadWriteLock中，判断**写锁**是否可重入，同样如此，只需要检查当前获得独占锁的线程是否为当前线程；

但是判断**读锁**是否可重入，则需要判断两种情况，一是写锁如果被占用，是否由当前线程占用，二是在需要进入同步队列时，判断当前线程之前是否已获得读锁并且还没有释放，如果是，则不需要进入同步队列，可以直接获得读锁

# 读锁的释放
读锁的释放入口为内部类ReadLock的unlock()函数，unlock函数中调用的releaseShared函数在AQS中实现，releaseShared函数中调用tryReleaseShared释放读锁，然后唤醒后继线程或者标记节点状态为PROPAGATE。tryReleaseShared在内部类Sync中实现。

		protected final boolean tryReleaseShared(int unused) {
            Thread current = Thread.currentThread();
            // 更新firstReader
            if (firstReader == current) {
                // assert firstReaderHoldCount > 0;
                if (firstReaderHoldCount == 1)
                    firstReader = null;
                else
                    firstReaderHoldCount--;
            } else {
                // 更新缓存最后一次获取读锁的变量cachedHoldCounter或本地线程变量
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                int count = rh.count;
                if (count <= 1) {
                    readHolds.remove();
                    if (count <= 0) //如果没有持有读锁，释放是非法的
                        throw unmatchedUnlockException();
                }
                --rh.count;
            }
            for (;;) {
                int c = getState();
                // 将读锁计数减1
                int nextc = c - SHARED_UNIT;
                if (compareAndSetState(c, nextc))
                    // Releasing the read lock has no effect on readers,
                    // but it may allow waiting writers to proceed if
                    // both read and write locks are now free.
                    return nextc == 0;
            }
        }

tryReleaseShared的主要逻辑为：更新线程本地变量中保存的该线程获取读锁的数量以及一些为了提高效率的变量值，将读锁数量减1。

# 锁降级

锁降级：从写锁变成读锁；锁升级：从读锁变成写锁。读锁是可以被多线程共享的，写锁是单线程独占的。也就是说写锁的并发限制比读锁高，这可能就是升级/降级名称的来源。

如下代码会产生死锁，因为同一个线程中，在没有释放读锁的情况下，就去申请写锁，这属于锁升级，ReentrantReadWriteLock是不支持的。

	ReadWriteLock rtLock = new ReentrantReadWriteLock();  
	rtLock.readLock().lock();  
	System.out.println("get readLock.");  
	rtLock.writeLock().lock();  
	System.out.println("blocking");  

ReentrantReadWriteLock支持锁降级，如果线程先获取写锁，再获得读锁，再释放写锁，这样写锁就降级为读锁了，如下代码是不会死锁的

		ReadWriteLock rtLock = new ReentrantReadWriteLock();
        rtLock.writeLock().lock();
        System.out.println("writeLock");

        rtLock.readLock().lock();
        System.out.println("get read lock");

        rtLock.writeLock().unlock();

        new Thread(() -> {
            rtLock.readLock().lock();
            System.out.println("other thread get read lock");
            System.out.flush();
            rtLock.readLock().unlock();
        }).start();
		
		rtLock.readLock().unlock();

# 演示代码
我们知道获取读锁时，当存在写锁排队时，写锁后面的读锁均需进入同步队列排队，除非当前线程已经获得读锁且还未释放，下面的代码演示一下子线程获得读锁并未释放时，即使有写锁排队，依然可以获得读锁

	final ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
        System.out.println("readWriteLock.readLock().lock() begin");
        readWriteLock.readLock().lock();
        System.out.println("readWriteLock.readLock().lock() over");


        new Thread(){
            @Override
            public void run() {
                for(int i = 0; i< 2; i++){
                    System.out.println(" ");
                    System.out.println("Thread readWriteLock.readLock().lock() begin i:"+i);
                    readWriteLock.readLock().lock(); // 获取过一次就能再次获取
                    System.out.println("Thread readWriteLock.readLock().lock() over i:" + i);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                    }
                }
            }
        }.start();

        new Thread(() -> {
            System.out.println("readWriteLock.writeLock().lock() begin");
            readWriteLock.writeLock().lock();
            System.out.println("readWriteLock.writeLock().lock() over");
        }).start();
		// 但是其他没有获取过读锁的线程，因为 syn queue里面 head.next 是获取write的线程, 则到 syn queue 里面进行等待
        // 下面的字符串不会打印出来
        new Thread(() -> {
            readWriteLock.readLock().lock();
            System.out.println("other thread get read lock");
            readWriteLock.readLock().unlock();
        }).start();


# 参考链接

[1] [https://www.jianshu.com/p/4a624281235e](https://www.jianshu.com/p/4a624281235e)

[2] [https://www.jianshu.com/p/6923c126e762](https://www.jianshu.com/p/6923c126e762)

[3] [http://blog.csdn.net/aitangyong/article/details/38315885](http://blog.csdn.net/aitangyong/article/details/38315885)