---
title: '[java1.8源码笔记]ThreadPoolExecutor详解'
date: 2018-03-05 17:32:38
categories: 'java1.8源码笔记'
tags:
---
# 概述
在java线程池一文中，我们提到java线程池的创建都是由ThreadPoolExecutor类实现，并且介绍了该类的构造函数参数的含义，本文详细分析java1.8中ThreadPoolExecutor源码，分析线程池的具体实现。

<!--more-->

# 线程池状态
ThreadPoolExecutor线程池有5个状态，分别是：

1. RUNNING：可以接受新的任务，也可以处理阻塞队列里的任务
2. SHUTDOWN：不接受新的任务，但是可以处理阻塞队列里的任务
3. STOP：不接受新的任务，不处理阻塞队列里的任务，中断正在处理的任务
4. TIDYING：过渡状态，也就是说所有的任务都执行完了，当前线程池已经没有有效的线程，这个时候线程池的状态将会TIDYING，并且将要调用terminated方法
5. TERMINATED：终止状态。terminated方法调用完成以后的状态

状态之间可以进行转换，转换关系如下图所示：
![](https://i.imgur.com/ulN7sk1.jpg)

* RUNNING -> SHUTDOWN：手动调用shutdown方法，或者ThreadPoolExecutor要被GC回收的时候调用finalize方法，finalize方法内部也会调用shutdown方法

* (RUNNING or SHUTDOWN) -> STOP：调用shutdownNow方法

* SHUTDOWN -> TIDYING：当队列和线程池都为空的时候

* STOP -> TIDYING：当线程池为空的时候

* TIDYING -> TERMINATED：terminated方法调用完成之后

# 相关变量

* AtomicInteger ctl
	> 这一变量保存了两个内容，int型变量总共32位，低29位保存有效线程数量，高3位保存线程池状态
* int COUNT_BITS = Integer.SIZE - 3;
	> ctl变量中保存有效线程数量的位数（即29）
* int CAPACITY   = (1 << COUNT_BITS) - 1;
	> ctl中能保存的有效线程数量的容量值
* int largestPoolSize;
	> 线程池运行过程中，追踪线程池中最大的线程数量
* long completedTaskCount;
	> 记录线程池完成的任务数量
* int maximumPoolSize;
	> 保存用户设定的线程池最大大小，由构造函数传入，当线程池队列满时，如果线程池中线程数量小于此值，则可以继续创建新的线程
* int corePoolSize;
	> 核心线程池大小，活动线程小于corePoolSize则直接创建，大于等于则先加到workQueue中，当队列满了，如果线程数量小于maximumPoolSize，继续创建新的线程（保存用户设定的线程池基本大小，由构造函数传入）
* long keepAliveTime;
	> 线程从队列中获取任务的超时时间，如果线程数量大于corePoolSize，此时如果线程空闲超过这个时间就会终止。（由构造函数传入）

# 内部类Worker
Worker类封装了线程池工作线程，即一个Worker实例代表一个工作线程。同时Worker类继承于AbstractQueuedSynchronizer，实现了独占锁，一个Worker实例也是一个锁。同时Worker类实现了Runnable接口，重写了run()方法，于是一个Worker实例可以用于线程中执行。

	private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            // 使用ThreadFactory构造Thread，这个构造的Thread内部的Runnable就是本身（传入的参数是this），也就是Worker。
            // 所以得到Worker的thread并start的时候，会执行Worker的run方法，也就是执行ThreadPoolExecutor的runWorker方法
            // 把AQS状态位state设置成-1，这样任何线程都不能得到Worker的锁，除非调用了unlock方法。
            // 这个unlock方法会在runWorker方法中一开始就调用，这是为了确保Worker构造出来之后，没有任何线程能够得到它的锁，除非调用了runWorker之后，其他线程才能获得Worker的锁
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

        // Lock methods 下面是锁方法
        //
        // The value 0 represents the unlocked state.
        // The value 1 represents the locked state.

        protected boolean isHeldExclusively() {
            return getState() != 0;
        }

        protected boolean tryAcquire(int unused) {
            if (compareAndSetState(0, 1)) {
                setExclusiveOwnerThread(Thread.currentThread());
                return true;
            }
            return false;
        }

        protected boolean tryRelease(int unused) {
            setExclusiveOwnerThread(null);
            setState(0);
            return true;
        }

        public void lock()        { acquire(1); }
        public boolean tryLock()  { return tryAcquire(1); }
        public void unlock()      { release(1); }
        public boolean isLocked() { return isHeldExclusively(); }

        void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
    }

因为在tryAcquire中获取锁时，通过CAS操作，将state值从0修改成1成功的话，可以获取到独占锁，于是Worker构造函数中将state值设为-1，可以让所有线程都无法获取到Worker的锁；在runWorker函数中调用unlock()时，会调用tryRelease函数，在tryRelease函数中会设置state值为0，于是此时其他线程可以获取到Worker的锁。

注意在Worker中实现的独占锁是**不可重入**的，用于判断工作线程是否空闲。

# 任务执行
在使用ThreadPoolExecutor类时，向线程池提交任务可以使用execute或submit方法，在基类AbstractExecutorService可以看到，实际上submit方法里面最终调用的还是execute()方法，所以execute方法就是执行任务的方法。

execute方法内部分3个步骤进行处理。

1. 如果当前正在执行的Worker数量比corePoolSize(基本大小)要小。直接创建一个新的Worker执行任务，会调用addWorker方法
2. 如果当前正在执行的Worker数量大于等于corePoolSize(基本大小)。将任务放到阻塞队列里，如果阻塞队列没满并且状态是RUNNING的话，直接丢到阻塞队列，否则执行第3步。丢到阻塞队列之后，还需要再做一次验证(丢到阻塞队列之后可能另外一个线程关闭了线程池或者刚刚加入到队列的线程死了)。如果这个时候线程池不在RUNNING状态，把刚刚丢入队列的任务remove掉，调用reject方法，否则查看Worker数量，如果Worker数量为0，起一个新的Worker去阻塞队列里拿任务执行
3. 丢到阻塞队列失败的话，会调用addWorker方法尝试起一个新的Worker去阻塞队列拿任务并执行任务，如果这个新的Worker创建失败，调用reject方法

execute方法源码如下，
	
	public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        
        //addWorker(null, false);这一行，这要结合addWorker一起来看。 主要目的是防止SHUTDOWN状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况。
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) { //offer 非阻塞加入队列,如果队列满则返回false
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);    // 采用线程池指定的策略拒绝任务,默认策略：抛出RejectedExecutionException异常
            else if (workerCountOf(recheck) == 0)
                // 这行代码是为了SHUTDOWN状态下没有活动线程了，但是队列里还有任务没执行这种特殊情况。
                // 添加一个null任务是因为SHUTDOWN状态下，线程池不再接受新任务
                // 非running状态，并且rs != SHUTDOWN,在addWorker中会因为rs>=SHUTDOWN && rs != SHUTDOWN 而返回false
                addWorker(null, false);
        }
        // 两种情况：
        // 1.非RUNNING状态拒绝新的任务,在addWorker中会因为rs>=SHUTDOWN && firstTask != null 而返回false
        // 2.队列满了加入队列失败（workCount > maximumPoolSize）,如果线程数量小于maximumPoolSize，继续创建新的线程来执行任务
        else if (!addWorker(command, false))
            reject(command);
    }

## 新建工作线程
当需要创建新的工作线程的时候，调用addWorker函数，其主要功能为检查线程池的状态，判断是否满足创建新的工作线程的条件，如果满足则创建一个Worker实例作为新的工作线程来执行任务。

	// 两个参数，firstTask表示需要跑的任务。boolean类型的core参数为true的话表示使用线程池的基本大小，为false使用线程池最大大小
    // 返回值是boolean类型，true表示新任务被接收了，并且执行了。否则是false
	private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c); //线程池的状态

            // 这个判断转换成 rs >= SHUTDOWN && (rs != SHUTDOWN || firstTask != null || workQueue.isEmpty)。
            // 以下三种情况下，返回false，函数直接返回，拒绝增加线程执行任务
            // (1)线程池不在RUNNING状态并且状态是STOP、TIDYING或TERMINATED中的任意一种状态(rs > SHUTDOWN)
            // (2)线程池是SHUTDOWN状态，且firstTask != null；
            // (3)线程池是SHUTDOWN状态，firstTask == null，且阻塞队列为空；
            // 总结：线程池为STOP、TIDYING或TERMINATED状态之一时，拒绝执行任务；线程池是SHUTDOWN状态时，拒绝执行新添加任务，但是如果阻塞队列非空，继续执行阻塞队列中的任务
            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);  //线程池中工作线程的数量
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))  //线程池线程数量是否超过最大容量或者是否超过设定的基本大小(core==true)或最大大小(core==false)
                    return false;
                if (compareAndIncrementWorkerCount(c))  //cas操作将线程池中线程数量+1，成功的话跳出循环
                    break retry;
                c = ctl.get();  // Re-read ctl  //重新检查状态
                if (runStateOf(c) != rs)    //如果状态改变了，重新循环操作
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        // 走到这一步说明cas操作成功了，线程池线程数量+1
        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());
                    //如果线程池是running状态 或者线程池在SHUTDOWN状态并且不是新增加任务
                    // （第二种情况一般是线程池状态为SHUTDOWN，但是阻塞队列非空，增加一个线程执行队列中已有的任务；
                    // 同样说明，线程池状态为SHUTDOWN时不接受新任务）
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();	// 启动线程，这里的t是Worker中的thread属性，所以相当于就是调用了Worker的run方法
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

由于Worker中的线程创建时传入的Runnable是Worker本身（this），Worker中的线程start的时候，调用Worker本身run方法，而run方法调用外部类ThreadPoolExecutor的runWorker方法。

runWorker中刚进入时，如果传入了任务，则先执行传入的任务，执行完成后，从阻塞队列中获取任务并执行，获取失败则进入回收线程逻辑。
	
	public void run() {
            runWorker(this);
        }

	final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) { //一直死循环，如果从队列中获取任务返回null，则退出循环，收回线程
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                // 执行任务前，做一些检查
                // 1. 如果线程池已经处于STOP状态并且当前线程没有被中断，中断线程
                // 2. 如果线程池还处于RUNNING或SHUTDOWN状态，并且当前线程已经被中断了，重新检查一下线程池状态，如果处于STOP状态并且没有被中断，那么中断线程
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run(); // 真正的开始执行任务
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);    //回收工作线程Worker
        }
    }

## 从阻塞队列中获取任务
看一下getTask方法如何从阻塞队列中获取任务的

	// 如果发生了以下四件事中的任意一件，那么Worker需要被回收：
    // 1. Worker个数比线程池最大大小要大
    // 2. 线程池处于STOP状态
    // 3. 线程池处于SHUTDOWN状态并且阻塞队列为空
    // 4. 使用超时时间从阻塞队列里拿数据，并且超时之后没有拿到数据(allowCoreThreadTimeOut || workerCount > corePoolSize)
	private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            // 线程池处于STOP状态 或者 线程池处于SHUTDOWN状态并且阻塞队列为空，worker数量减一
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            // allowCoreThreadTimeOut默认为false，表示数量小于基本大小的线程会一直保存在线程池中;如果是true表示核心线程使用keepAliveTime这个参数来作为超时时间
            // 如果worker数量比基本大小要大的话，timed就为true，需要进行回收worker
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();   //take()方法一直阻塞
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

通过前面的说明，都知道当线程池中线程的数量大于基本大小时，空闲线程的存活时间为keepAliveTime，当超过keepAliveTime时间没有从队列中获取到任务时，线程会被收回，这样**保证线程池中工作线程的数量保持在基本大小**。

实现方法就是这里，当线程数量大于基本大小时，线程采用队列的带超时的poll方法从队列中获取任务，poll方法如果超时没有获取到任务会返回null，于是检查poll的结果，如果为null则说明超时，回收线程。

如果线程数量小于等于基本大小时，线程采用队列的take()方法从队列中获取任务，该方法当队列为空时会一直阻塞，直到获取任务成功。

## 回收工作线程
如果getTask返回的是null，那说明阻塞队列已经没有任务并且当前调用getTask的Worker需要被回收，那么会调用processWorkerExit方法进行回收。worker回收时需要检查线程池中的数量必须满足最小条件，当核心线程没有设置超时时，线程数量必须至少保持为基本大小corePoolSize；设置了超时的话，线程池中线程数量可以位0，但是如果阻塞队列不为空，则至少有一个工作线程。

	private void processWorkerExit(Worker w, boolean completedAbruptly) {
        // 如果Worker没有正常结束流程调用processWorkerExit方法，worker数量减一。如果是正常结束的话，在getTask方法里worker数量已经减一了
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks; //记录线程池完成的任务数量
            workers.remove(w);  //从线程池的worker集合中删除需要回收的worker
        } finally {
            mainLock.unlock();
        }

        tryTerminate(); //尝试结束线程池

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {    //如果线程池还处于RUNNING或者SHUTDOWN状态
            if (!completedAbruptly) {       // worker是正常结束流程
                //如果设置了allowCoreThreadTimeOut==true，即核心线程也设置了超时，所以线程池中线程数量可以为0
                //否则，线程池中线程的数量至少保持在设定的基本大小corePoolSize
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())  //当阻塞队列不为空时，至少要有一个线程
                    min = 1;
                if (workerCountOf(c) >= min)    //如果目前线程池中的线程数量满足最小条件，则可以直接返回
                    return; // replacement not needed
            }
            // 新开一个Worker代替原先的Worker
            // 新开一个Worker需要满足以下3个条件中的任意一个：
            // 1. 用户执行的任务发生了异常
            // 2. Worker数量比线程池基本大小要小
            // 3. 阻塞队列不空但是没有任何Worker在工作
            addWorker(null, false);
        }
    }

## 尝试结束线程池
在processWorkerExit函数中回收工作线程的时候，以及在addWorkerFailed函数中处理添加工作线程失败的时候，都会调用tryTerminate尝试结束线程池。

	final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            // 满足3个条件中的任意一个，不终止线程池
            // 1. 线程池还在运行，不能终止
            // 2. 线程池处于TIDYING或TERMINATED状态，说明已经在关闭了，不允许继续处理
            // 3. 线程池处于SHUTDOWN状态并且阻塞队列不为空，这时候还需要处理阻塞队列的任务，不能终止线程池
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            // 走到这一步说明线程池已经不在运行，阻塞队列已经没有任务，但是还要回收正在工作的Worker
            if (workerCountOf(c) != 0) { // Eligible to terminate
                // 中断闲置Worker，直到回收全部的Worker。
                // 这里没有那么暴力，只中断一个，中断之后退出方法，中断了Worker之后，Worker会回收，调用processWorkerExit，
                // 然后还是会调用tryTerminate方法，如果还有闲置线程，那么继续中断
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            // 走到这里说明worker已经全部回收了，并且线程池已经不在运行，阻塞队列已经没有任务。可以准备结束线程池了
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0)); // terminated方法调用完毕之后，状态变为TERMINATED
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }

# 线程池的关闭
## shutdown方法
shutdown方法，关闭线程池，调用shutdown关闭之后线程池状态变为SHUTDOWN，线程池会继续处理阻塞队列里的任务，但是新的任务不会被接受。

	public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN); // 把线程池状态更新到SHUTDOWN
            interruptIdleWorkers(); // 中断闲置的Worker
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate(); //尝试结束线程池
    }

	private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }

	private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }

在interruptIdleWorkers函数中，依次从worker线程集合中取出一个worker处理，调用

	if (!t.isInterrupted() && w.tryLock())
判断该线程是否被中断，如果未被中断，则尝试获取其独占锁，如果获取锁成功，则说明该线程为一个空闲线程（why?见下面说明），调用interrupt()中断该线程。

工作线程在runWorker函数中执行任务时，如果获得任务并且开始执行前，会调用worker的锁lock()操作，**由于该锁为不可重入的独占锁**，于是只有线程阻塞在阻塞队列中获取任务时，在interruptIdleWorkers中调用tryLock才会获取锁成功，所以此处如果获得锁成功，表示该线程为空闲线程。

## shutdownNow方法
调用shutdownNow方法关闭线程池后，线程池状态变为STOP，此时线程池不会接受新的任务，也不会继续处理阻塞队列中的任务，同时会立即终止所有有效工作线程的运行。

	public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);  // 把线程池状态更新到STOP
            interruptWorkers();     // 终止正在执行的任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
	
	private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }

		void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }

# 任务拒绝策略
当线程池的任务缓存队列已满并且线程池中的线程数目达到maximumPoolSize，如果还有任务到来就会采取任务拒绝策略，通常有以下四种策略：

	ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。默认操作
	ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。
	ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）
	ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务

# 参考链接
[1] [https://fangjian0423.github.io/2016/03/22/java-threadpool-analysis/](https://fangjian0423.github.io/2016/03/22/java-threadpool-analysis/)

[2] [http://www.importnew.com/19011.html](http://www.importnew.com/19011.html)