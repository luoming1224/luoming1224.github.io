---
title: '[java1.8源码笔记]CAS介绍'
date: 2018-02-07 15:23:14
categories: 'java1.8源码笔记'
tags: [CSA, 乐观锁, Unsafe类]
---
# 概述
CAS，在java并发应用中通常指CompareAndSwap或CompareAndSet，即比较并交换。它比较一个内存位置的值并且只有等于预期值时才修改这个内存位置的值为新的值，如果有其他线程在这期间修改了内存位置的值则CAS操作失败。CAS返回是否成功或者内存位置原来的值用于判断是否操作成功。即CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

<!--more-->

CAS是一个原子操作，是一条CPU的原子指令，其实现方式是基于硬件平台的汇编指令，也就是说CAS是靠硬件实现的，硬件保证其操作的原子性。CAS是一种系统原语，原语属于操作系统用语范畴，是由若干条指令组成的，用于完成某个功能的一个过程，并且原语的执行必须是连续的，在执行过程中不允许被中断，也就是说CAS是一条CPU的原子指令。

java.util.concurrent包完全建立在CAS之上的，其中大量使用了CAS操作，java.util.concurrent包中借助CAS实现了区别于synchronouse同步锁的一种乐观锁。一般在循环中调用CAS，直到设置成功。

# 乐观锁与悲观锁
悲观锁（Pessimistic Lock）： 
每次获取数据的时候，都会担心数据被修改，所以每次获取数据的时候都会进行加锁，确保在自己使用的过程中数据不会被别人修改，使用完成后进行数据解锁。由于数据进行加锁，期间对该数据进行读写的其他线程都会进行等待。

乐观锁（Optimistic Lock）： 
每次获取数据的时候，都不会担心数据被修改，所以每次获取数据的时候都不会进行加锁，但是在更新数据的时候需要判断该数据是否被别人修改过。如果数据被其他线程修改，则不进行数据更新，如果数据没有被其他线程修改，则进行数据更新。由于数据没有进行加锁，期间该数据可以被其他线程进行读写操作。

# Unsafe类
Unsafe类存在于sun.misc包中，其内部方法操作可以像C的指针一样直接操作内存，单从名称看来就可以知道该类是非安全的，因此总是不应该首先使用Unsafe类。**Java中CAS操作的执行依赖于Unsafe类的方法**，注意Unsafe类中的所有方法都是native修饰的，也就是说Unsafe类中的方法都直接调用操作系统底层资源执行相应任务。

	public class AtomicInteger extends Number implements java.io.Serializable {
	    private static final long serialVersionUID = 6214790243416807050L;
	
	    // setup to use Unsafe.compareAndSwapInt for updates
	    private static final Unsafe unsafe = Unsafe.getUnsafe();
	    private static final long valueOffset;
	
	    static {
	        try {
	            valueOffset = unsafe.objectFieldOffset
	                (AtomicInteger.class.getDeclaredField("value"));
	        } catch (Exception ex) { throw new Error(ex); }
	    }
	
	    private volatile int value;
		public final int getAndIncrement() {
        	return unsafe.getAndAddInt(this, valueOffset, 1);
		}
    }

	public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }

compareAndSwapInt是一个本地方法调用，在intel x86处理器中是通过cmpxchg指令实现。

# CAS的ABA问题
因为CAS需要在操作值的时候检查下值有没有发生变化，如果没有发生变化则更新，但是如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。

在Java中解决ABA问题，我们可以使用以下两个原子类

* AtomicStampedReference

	AtomicStampedReference原子类是一个带有时间戳的对象引用，在每次修改后，AtomicStampedReference不仅会设置新值而且还会记录更改的时间。当AtomicStampedReference设置对象值时，对象值以及时间戳都必须满足期望值才能写入成功，这也就解决了反复读写时，无法预知值是否已被修改的窘境，

* AtomicMarkableReference类

	AtomicMarkableReference与AtomicStampedReference不同的是，AtomicMarkableReference维护的是一个boolean值的标识，也就是说至于true和false两种切换状态

# 参考链接

[1] [http://blog.csdn.net/javazejian/article/details/72772470](http://blog.csdn.net/javazejian/article/details/72772470)

[2] [http://www.jianshu.com/p/fb6e91b013cc](http://www.jianshu.com/p/fb6e91b013cc)

[3] [http://blog.sina.com.cn/s/blog_ee34aa660102wsuv.html](http://blog.sina.com.cn/s/blog_ee34aa660102wsuv.html)

[4] [https://liuzhengyang.github.io/2017/05/11/cas/](https://liuzhengyang.github.io/2017/05/11/cas/)

[5] [http://zl198751.iteye.com/blog/1848575](http://zl198751.iteye.com/blog/1848575)