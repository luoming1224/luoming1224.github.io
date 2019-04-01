---
title: '[java学习笔记]java线程池'
date: 2018-03-02 17:05:43
categories: 'java学习笔记'
tags: [线程池]
---
# 概述
我们都知道线程的创建和销毁过程开销比较大，如果并发的线程数量很多，并且每个线程都是执行一个时间很短的任务就结束了，这样频繁创建线程就会大大降低系统的效率，此时推荐使用线程池来复用线程，使得线程执行完一个任务并不被销毁，而是继续执行其他任务。然而线程池的实现相对比较复杂，于是java并发包中帮我们实现了一个功能强大的线程池。java线程池负责管理工作线程，包含一个等待执行的任务队列。线程池的任务队列是一个Runnable集合，工作线程负责从任务队列中取出并执行Runnable对象。

<!--more-->

在java线程池的使用过程中涉及Executors、Executor、ExecutorService等几个类

# 创建线程池
在java中Executors类中提供了负责生成各种类型的线程池的实例的静态方法，主要有四种类型：固定大小线程池、可变大小线程池、单任务线程池、周期性执行的线程池

## 固定大小线程池
生成固定大小线程池的静态方法

* public static ExecutorService newFixedThreadPool(int nThreads)
* public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory)

创建的线程池包含nThreads个固定数量的线程

## 可变大小线程池
生成可变大小线程池的静态方法

* public static ExecutorService newCachedThreadPool()
* public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory)

创建的线程池包含最少0个，最多Integer.MAX_VALUE = 0x7fffffff 个线程，空闲线程存活时间为60秒，即空闲线程60秒没有获取到任务后就会关闭

## 单任务线程池
生成单任务线程池的静态方法

* public static ExecutorService newSingleThreadExecutor()
* public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory)

创建的线程池包含一个工作线程

* public static ScheduledExecutorService newSingleThreadScheduledExecutor()
* public static ScheduledExecutorService newSingleThreadScheduledExecutor(ThreadFactory threadFactory)

创建包含一个工作线程的线程池，且周期性的执行任务

## 周期性执行的线程池

* public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize)
* public static ScheduledExecutorService newScheduledThreadPool(
            int corePoolSize, ThreadFactory threadFactory)

该方法的返回值ScheduledExecutorService有四个成员函数用于执行任务：

（1）public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit);

（2）public <V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit);

（3）public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit);

> 在上一个任务开始执行之后延迟多少秒之后再执行，是从上一个任务开始时开始计算
> 
> 如果延迟时间比任务执行时间短，还是会等到上一个任务执行完成之后，下一个任务才开始执行

（4）public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit);

> 在上一个任务执行结束之后，延迟多少秒之后再执行，是从上一个任务结束时开始计算

可以用如下简单的示例，观察打印结果，加深对以上两种周期性延迟任务的差别

		ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(3);
        scheduler.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                System.out.println("scheduleAtFixedRate:    " + new Date());
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, 1, 3L , SECONDS);

        scheduler.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                System.out.println("scheduleWithFixedDelay: " + new Date());
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }, 1, 3L , SECONDS);

# 管理线程池
在Executor类及其子类ExecutorService、ScheduledExecutorService中提供了向线程池提交和执行任务的方法，三者之间的继承关系如下图所示：

![](https://i.imgur.com/BAHp4RC.png)

其中ScheduledExecutorService提供了向周期性任务线程池提交任务的方法，具体介绍和示例见上面；在Executor和ExecutorService中提供的主要方法有：

* void execute(Runnable command);
* <T> Future<T> submit(Callable<T> task);
* <T> Future<T> submit(Runnable task, T result);
* Future<?> submit(Runnable task);

> execute(Runnable x) 没有返回值。可以执行任务，但无法判断任务是否成功完成。
> submit(Runnable x) 返回一个future。可以用这个future来判断任务是否成功完成。

而且函数的参数可以是Runnable或者Callable<T>，两者都可以用于将耗时操作写在其中，然后使用某个线程去执行，即可实现多线程；两者的区别为：Runnable有一个run()函数，该函数没有返回值，不能将结果返回给客户程序，Callable中有一个call()函数，但是call()函数有返回值，可以通过Future类的get()函数获得线程执行的返回值。

# 自定义线程池及参数说明
通过查看java1.8源码可以发现，在Executors类中创建各类线程池的方法都是由ThreadPoolExecutor实现，所以我们自己也可以直接调用ThreadPoolExecutor构造函数自定义创建线程池。这种方式并不推荐使用，因为对于开发者来说比较困难，也不好管理和维护，但这种方式可以做到对线程池更细致更自由化的控制。

ThreadPoolExecutor类的构造函数接受以下这几个重要的参数：

* corePoolSize：线程池基本数量
* workQueue：用于保存等待执行的任务的阻塞队列。阻塞队列的类型可自己选择，阻塞有这几种类型，ArrayBlockingQueue(基于数组的有界阻塞队列，按FIFO进出任务)，LinkedBlockingQueue(基于链表的阻塞队列，FIFO方式)，SynchronousQueue(不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量要高于LinkedBlockingQueue，是一个无限阻塞队列)，PriorityBlockingQueue(具有优先级的无限阻塞队列)
* maximunPoolSize：线程池最大数量，也就是线程池允许创建的最大线程数。这个参数跟阻塞队列的关系是这样的：如果阻塞队列满了，这个时候又来了一个任务，那么这个时候如果当前线程数小于最大数量，那么就会直接创建新的线程执行任务。当然，如果工作队列是无界的，那么这个参数就没有意义了，因为队列无界，任务都会在队列中存储着
* ThreadFactory：创建线程的工厂
* RejectedExecutionHandler：饱和策略，也就是当队列和线程数目都满了以后，采取的策略。有AbortPolicy(直接抛出异常)，CallerRunsPolicy(只用调用者所在线程来运行任务)，DiscardOldestPolicy(丢弃队列里最近的一个任务，并执行当前任务)，DiscardPolicy(不处理，直接丢弃)。当然，还可以自定义策略
* keeyAliveTime：存活时间。如果当前线程池中的线程数量比基本数量要多，并且是闲置状态的话，这些闲置的线程能存活的最大时间
* TimeUnit，跟第6个参数一起使用，表示存活时间的时间单位

这些参数中，有3个参数跟线程池大小有关，分别是基本数量，最大数量和阻塞队列。

# 线程池的处理流程

1. 首先判断线程池的基本大小，如果基本大小还没满，那么直接创建新的线程执行任务，否则进行下一步
2. 判断线程池中的阻塞队列是是否已满，没满的话存到阻塞队列里等待执行，否则执行下一步(所以如果是个无界的阻塞队列，那么这一步永远都成立)
3. 判断线程池最大大小是否已满，没满的话直接创建线程执行任务，否则交给饱和策略处理

通过如下示例来理解线程池的处理流程：

	public static void main(String[] args) {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(1, 20, 10, TimeUnit.SECONDS,new LinkedBlockingQueue(10));

        for(int i = 0; i < 15; i ++) {
            threadPoolExecutor.submit(new MyThread(i + 1));
        }
    }

    static class MyThread implements Runnable {
        private int index;
        public MyThread(int index) {
            this.index = index;
        }
        @Override
        public void run() {
            System.out.println(this.index);
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
                Thread.currentThread().interrupt();
            }
        }
    }

执行该示例，我们可以看到，线程池先创建一个线程打印1；接着由于线程池基本大小为1，队列大小为10，于是将2-11缓存进队列中；队列满后，由于线程池最大大小为20，于是接着创建4个线程，打印12-15；此时线程池中有5个线程，5个线程完成当前任务后，从队列中获取任务来执行；当队列中的任务都执行完毕后，超过10秒钟没有可执行的任务的话，有4个线程会被线程池关闭，最终保留1个基本大小的线程。

# 参考链接

[1] [http://fangjian0423.github.io/2015/07/24/java-poolthread/](http://fangjian0423.github.io/2015/07/24/java-poolthread/)

[2] [http://blog.csdn.net/bboyfeiyu/article/details/24851847](http://blog.csdn.net/bboyfeiyu/article/details/24851847)


