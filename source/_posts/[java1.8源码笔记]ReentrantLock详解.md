---
title: '[java1.8源码笔记]ReentrantLock详解'
date: 2018-02-28 16:31:32
categories: 'java1.8源码笔记'
tags: [ReentrantLock, 公平锁, 非公平锁, 互斥锁]
---
# 概述
ReentrantLock是java并发包中实现的可重入互斥锁，基于AQS实现，支持公平锁与非公平锁两种实现方式。公平锁是指当同步等待队列中有线程在排队获取锁时，新加入的准备获取锁的线程则加入同步等待队列尾；非公平锁是指新加入的准备获取锁的线程，不管同步等待队列中是否有线程正在排队，而直接先尝试获取锁，如果获取失败则加入同步等待队列尾。

<!--more-->

# 内部类
ReentrantLock有三个内部类，其中Sync继承于AbstractQueuedSynchronizer；NonfairSync和FairSync则均继承于Sync分别用于实现非公平锁和公平锁。类依赖图如下所示：
![](https://i.imgur.com/T3M1gFU.png)

## Sync类

	abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = -5179523762034025860L;

        /**
         * Performs {@link Lock#lock}. The main reason for subclassing
         * is to allow fast path for nonfair version.
         */
        abstract void lock();

        /**
         * Performs non-fair tryLock.  tryAcquire is implemented in
         * subclasses, but both need nonfair try for trylock method.
         */
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // state == 0表示当前锁处于空闲状态，CAS操作修改state值，修改成功则表示获取锁成功
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 如果锁非空闲，但当前占有锁的线程即为当前线程本身，增加state的值，同样获得锁
            // 这里说明ReentrantLock为可重入锁
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }

        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            // c == 0 表示锁被当前线程完全释放，处于空闲状态
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }

        protected final boolean isHeldExclusively() {
            // While we must in general read state before owner,
            // we don't need to do so to check if current thread is owner
            return getExclusiveOwnerThread() == Thread.currentThread();
        }

        final ConditionObject newCondition() {
            return new ConditionObject();
        }

        // Methods relayed from outer class

        final Thread getOwner() {
            return getState() == 0 ? null : getExclusiveOwnerThread();
        }

        final int getHoldCount() {
            return isHeldExclusively() ? getState() : 0;
        }

        final boolean isLocked() {
            return getState() != 0;
        }

        /**
         * Reconstitutes the instance from a stream (that is, deserializes it).
         */
        private void readObject(java.io.ObjectInputStream s)
            throws java.io.IOException, ClassNotFoundException {
            s.defaultReadObject();
            setState(0); // reset to unlocked state
        }
    }

Sync类是NonfairSync和FairSync的基类，继承于AQS，其中实现了释放锁的操作以及非公平模式下尝试获取锁的操作。

## NonfairSync类

	static final class NonfairSync extends Sync {
        private static final long serialVersionUID = 7316153563782823691L;

        /**
         * Performs lock.  Try immediate barge, backing up to normal
         * acquire on failure.
         */
        // 非公平获取锁操作，
        final void lock() {
            // 不管是否有线程在排队获取锁，先直接CAS操作尝试修改state的值，修改成功则获得锁
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

实现**非公平锁**，在lock()操作中，不管是否有线程在排队，先尝试获取锁，如果获得成功则返回；否则调用acquire函数，该函数在AQS中实现，其功能为在将线程加入同步等待队列前，再次尝试获取锁，如果获取失败则将线程封装为Node节点加入同步等待队列并挂起，等待其前驱线程释放锁时唤醒。

## FairSync类

	static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            // 如果锁空闲，并且同步等待队列中无其他线程在排队，则尝试获取锁
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            //当前占有锁的线程即为当前线程本身，增加state的值，同样获得锁（可重入）
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

实现**公平锁**，只有在同步等待队列中没有其他线程在排队时才尝试获取锁，否则直接加入同步等待队列尾。

	public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }

hasQueuedPredecessors实现检查同步等待队列中是否有其他线程在排队，两种情况视为队列空：
（1）head == tail，即队列头尾指针指向同一个节点

（2）head.next != null && head.next.thread == Thread.currentThread()，队列中第一个节点为当前线程本身

# 构造函数

* public ReentrantLock() {sync = new NonfairSync();}
	> 默认构造函数返回一个非公平锁
* public ReentrantLock(boolean fair) {sync = fair ? new FairSync() : new NonfairSync();}
	> 带参数的构造函数，参数为true返回公平锁，参数为false返回非公平锁

# 常见方法
* public void lock();
	> 获取锁
* public void unlock();
	> 释放锁
* public Condition newCondition();
	> 新建一个Condition对象