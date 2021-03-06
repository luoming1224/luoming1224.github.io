---
title: '[java1.8源码笔记]ConcurrentHashMap剖析'
date: 2018-02-09 15:23:14
categories: 'java1.8源码笔记'
tags: [ConcurrentHashMap, java1.8, 源码, 扩容]
---

# 概述
ConcurrentHashMap是HashMap的线程安全版本，可以用来替代HashTable，HashTable的线程安全是通过用synchronized关键词来修饰每个方法来实现，，即Hashtable是针对整个table的锁定，同一时刻只能有一个方法被一个线程执行，这样就导致HashTable容器在竞争激烈的并发环境下表现出效率低下。而在java1.8中ConcurrentHashMap通过CAS来实现非阻塞的无锁线程安全、在操作hash值相同的hash桶时用synchronized锁住链表头结点来实现线程安全，锁粒度更小，并发性更高。

<!--more-->

ConcurrentHashMap的底层与Java1.8的HashMap有相通之处，底层依然由“数组”+链表+红黑树来实现的。当发生碰撞时，用链表解决hash碰撞，当链表长度>=8时，将链表转换为红黑树结构（table容量小于64时，不执行转换操作，先扩容）。

ConcurrentHashMap不接受nullkey和nullvalue

ConcurrentHashMap在java1.8中实现与java1.7中的区别：
在java1.7中，ConcurrentHashMap使用的就是分段锁技术，ConcurrentHashMap由多个Segment组成(Segment下包含很多Node，也就是我们的键值对了)，每个Segment都有把锁来实现线程安全，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。

# 部分相关内部类介绍
## Node<K, V>
Node为ConcurrentHashMap中实际保存插入的key和value的节点，实现了Entry接口，在ConcurrentHashMap中实际保存的是Node及其子类的对象

	static class Node<K,V> implements Map.Entry<K,V> {
        //Node节点的属性hash值保存的实际上是key的hash值（确切的说是key的hash值经过spread()函数计算后的结果）
		final int hash;     
        final K key;
        volatile V val;
        volatile Node<K,V> next;
	}

## ForwardingNode<K, V>
ForwardingNode为Node的子类，用于map扩容时标记和通知其他线程，当前正在扩容；当每个线程迁移完table中一个索引位置i处hash桶中所有key时，将table[i]置为一个ForwardingNode对象，其他线程在操作map时发现ForwardingNode对象，则加入一起辅助扩容。

	/**
     * A node inserted at head of bins during transfer operations.
     */
    static final class ForwardingNode<K,V> extends Node<K,V> {
        final Node<K,V>[] nextTable;
        ForwardingNode(Node<K,V>[] tab) {
            super(MOVED, null, null, null); //hash值为MOVED（-1）的节点就是ForwardingNode
            this.nextTable = tab;
        }

        // 通过此方法，访问迁移到nextTable中的数据
        Node<K,V> find(int h, Object k) {
            // loop to avoid arbitrarily deep recursion on forwarding nodes
            outer: for (Node<K,V>[] tab = nextTable;;) {
                Node<K,V> e; int n;
                if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                    return null;
                for (;;) {
                    int eh; K ek;
                    if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                        return e;
                    if (eh < 0) {
						// 如果为ForwardingNode，则重新从outer位置开始
                        if (e instanceof ForwardingNode) {
                            tab = ((ForwardingNode<K,V>)e).nextTable;
                            continue outer;
                        }
                        else
                            return e.find(h, k);//如果不为ForwardingNode，则可能为TreeBin等节点，具体见get()函数分析
                    }
					// 遍历链表中下一个节点
                    if ((e = e.next) == null)
                        return null;
                }
            }
        }
    }

## TreeNode<K,V>
TreeNode为红黑树节点，但TreeNode用于TreeBin中，并未直接插入ConcurrentHashMap的hash桶中

	/**
     * Nodes for use in TreeBins
     */
	static final class TreeNode<K,V> extends Node<K,V> {
        TreeNode<K,V> parent;  // red-black tree links
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;    // needed to unlink next upon deletion
        boolean red;

        TreeNode(int hash, K key, V val, Node<K,V> next,
                 TreeNode<K,V> parent) {
            super(hash, key, val, next);
            this.parent = parent;
        }
		...
	}

## TreeBin<K,V>
ConcurrentHashMap链表转树时，并不会直接转，只是把这些链表节点包装成TreeNode放到TreeBin中，再由TreeBin来转化红黑树。TreeBin用于封装维护TreeNode，当链表转树时，用于封装TreeNode，也就是说，ConcurrentHashMap的红黑树存放的时TreeBin，而不是treeNode。

	static final class TreeBin<K,V> extends Node<K,V> {
        TreeNode<K,V> root;
        volatile TreeNode<K,V> first;
        volatile Thread waiter;
        volatile int lockState;
        // values for lockState
        static final int WRITER = 1; // set while holding write lock
        static final int WAITER = 2; // set when waiting for write lock
        static final int READER = 4; // increment value for setting read lock
	}



# 初始化
ConcurrentHashMap采用延迟初始化，new一个ConcurrentHashMap对象时，并未初始化table结构，直到第一次插入数据时才真正初始化table数组，table是一个Node数组，Node保存了插入的key和value。初始化时如果没有设置ConcurrentHashMap容量的大小（即数组table的大小），则默认大小为16，ConcurrentHashMap容量大小通常为2的幂次方。

ConcurrentHashMap如何保证并发环境下table只被初始化一次？准备初始化table的线程，先利用CAS操作将sizeCtl的值置为-1，如果CAS操作成功再进行初始化；其他线程准备初始化时，先检查sizeCtl的值，如果小于0，则说明有其他线程正在进行初始化，此线程执行Thread.yield()函数放弃时间片。

## 相关成员属性介绍
* transient volatile Node<K,V>[] table;
	> map底层存储数据的数组,采用延迟初始化，直到第一次插入数据时才真正初始化该数组，数组大小通常为2的幂次方

* private transient volatile int sizeCtl;
	> sizeCtl是控制标识符，在不同的阶段，其值代表不同的意义
	> 
    >（1）负数代表正在进行初始化或扩容操作 ,其中-1代表正在初始化 ;表示扩容时，低16位的值N 表示有N-1个线程正在进行扩容操作（具体含义在扩容时介绍）
    >
    >（2）table == null时，表示创建table时的初始化大小，默认初始化值是0
    >
    >（3）table初始化后，表示触发下一次扩容操作的table容量阈值（大于该阈值时需要扩容）
    
## 源码分析
	/**
     * Initializes table, using the size recorded in sizeCtl.
     */
    private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) {
            if ((sc = sizeCtl) < 0)             //如果sizeCtl为负数，则说明已经有其它线程正在进行初始化，
                Thread.yield(); // lost initialization race; just spin
            //初始化table之前，利用CAS将sizeCtl设置为-1，表示当前已有线程正在初始化，保证只有一个线程能够执行table初始化
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        @SuppressWarnings("unchecked")
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                        table = tab = nt;
                        sc = n - (n >>> 2);     //即sc = 0.75n
                    }
                } finally {
                    sizeCtl = sc;	// 更新扩容阈值
                }
                break;
            }
        }
        return tab;
    }

# 插入操作-put
先判断key和value是否为null，如果为null则抛出异常；如果table为null或大小为0，则进行初始化；如果待插入的hash桶位置不存在其他key则直接用CAS操作插入数据；如果发现其他线程正在执行扩容操作，则帮助扩容；如果发生碰撞，则根据hash桶中现有节点为链表还是红黑树结构，执行相应的插入操作；如果是链表结构，插入后检查链表长度是否>=8，如果是则转换成红黑树；最后增加key的数量计数
## 相关成员属性介绍
* static final int MOVED     = -1; // hash for forwarding nodes
	> hash值为MOVED(-1)的节点，表示当前有线程正在扩容；扩容时，每迁移完一个hash桶中的节点时，会在hash桶中插入一个ForwardingNode节点，该节点的hash值为MOVED(-1)
* static final int TREEBIN   = -2; // hash for roots of trees
	> hash值为TREEBIN(-2)的节点，表示该hash桶中为红黑树结构，在将链表转换为红黑树时，在table[i]位置插入一个TreeBin节点，该节点的hash值为TREEBIN(-2)
	> 
	> 因此在插入操作时，如果位置i处的节点hash值>=0，表示为链表；否则hash值为-2，为红黑树
	
## 源码分析

	 public V put(K key, V value) {
        return putVal(key, value, false);
     }

    /** Implementation for put and putIfAbsent */
    final V putVal(K key, V value, boolean onlyIfAbsent) {
        if (key == null || value == null) throw new NullPointerException();
        int hash = spread(key.hashCode());
        int binCount = 0;
        for (Node<K,V>[] tab = table;;) {
            Node<K,V> f; int n, i, fh;
            if (tab == null || (n = tab.length) == 0) //如果table为null或大小为0，则初始化
                tab = initTable();
            else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {    // i=(n-1)&hash 等价于i=hash%n(前提是n为2的幂次方)
                if (casTabAt(tab, i, null,
                             new Node<K,V>(hash, key, value, null)))
                    break;                   // no lock when adding to empty bin
            }
            else if ((fh = f.hash) == MOVED) //检查table[i]的节点的hash是否等于MOVED，如果等于，则检测到正在扩容，则帮助其扩容
                tab = helpTransfer(tab, f);
            else {
                V oldVal = null;
                // 当有修改操作时借助了synchronized来对table[i]进行锁定保证了线程安全
                synchronized (f) { //锁定,（hash值相同的链表的头节点）
                    if (tabAt(tab, i) == f) {
                        if (fh >= 0) {  //链表节点，TreeBin节点的hash值在TreeBin构造函数中设置为TREEBIN(-2)
                            binCount = 1;
                            for (Node<K,V> e = f;; ++binCount) {
                                K ek;
                                if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                     (ek != null && key.equals(ek)))) {
                                    oldVal = e.val;
                                    if (!onlyIfAbsent)
                                        e.val = value;
                                    break;
                                }
                                Node<K,V> pred = e;
                                if ((e = e.next) == null) {             //如果链表中不存在相同的key，则插入到链表末尾
                                    pred.next = new Node<K,V>(hash, key,
                                                              value, null);
                                    break;
                                }
                            }
                        }
                        else if (f instanceof TreeBin) {
                            Node<K,V> p;
                            binCount = 2;
                            if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                           value)) != null) {
                                oldVal = p.val;
                                if (!onlyIfAbsent)
                                    p.val = value;
                            }
                        }
                    }
                }
                //插入成功后，如果插入的是链表节点，则要判断下该桶位是否要转化为树
                if (binCount != 0) {
                    if (binCount >= TREEIFY_THRESHOLD)
                        treeifyBin(tab, i);
                    if (oldVal != null)
                        return oldVal;
                    break;
                }
            }
        }
		// 增加计数
        addCount(1L, binCount);
        return null;
    }

在经典的hash表数据结构中，一种常用的散列法为除留余数法，即key的hash值对table大小n取余数（i=hash(key)%n）,然而这里计算key在table中的位置时，由于在ConcurrentHashMap中n的大小为2的幂次方，所以可以用（i = (n - 1) & hash），位运算比取余数运算效率更高。

当插入位置处不存在key时，采用CAS无锁方式插入来保证线程安全；当存在key，需要对链表或红黑树操作时，利用synchronized对table[i]节点进行锁定保证线程安全。

	int hash = spread(key.hashCode());	
	static final int spread(int h) {
        return (h ^ (h >>> 16)) & HASH_BITS;
    }

计算hash值时，让key的hashCode值无符号右移16位得到高16位，与低16进行异或操作，主要是在key的hashCode值比较小时，让所有的位都尽量参与进来，减少hash碰撞

	// 将碰撞链表转换为红黑树，除非table大小非常小（默认小于64），则先直接扩容来解决碰撞
    private final void treeifyBin(Node<K,V>[] tab, int index) {
        Node<K,V> b; int n, sc;
        if (tab != null) {
            if ((n = tab.length) < MIN_TREEIFY_CAPACITY)    //table容量小于64，则先扩容
                tryPresize(n << 1);
            else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {  // b.hash >= 0表示为链表，进一步验证
                synchronized (b) {
                    if (tabAt(tab, index) == b) {   //加锁后进一步验证，有可能被别的线程转换过了，时刻考虑并发
                        TreeNode<K,V> hd = null, tl = null;
                        for (Node<K,V> e = b; e != null; e = e.next) {  //形成了一个TreeNode的双向链表
                            TreeNode<K,V> p =
                                new TreeNode<K,V>(e.hash, e.key, e.val,
                                                  null, null);
                            if ((p.prev = tl) == null)
                                hd = p;
                            else
                                tl.next = p;
                            tl = p;
                        }
                        setTabAt(tab, index, new TreeBin<K,V>(hd));
                    }
                }
            }
        }
    }

转换成红黑树后，table[i]处为一个TreeBin节点，其hash值为TREEBIN(-2)。

helpTransfer（帮助扩容）、tryPresize（扩容）、addCount（计数）这几个函数后面相应的章节分析。

# 扩容与辅助扩容
前面说过，当table初始化后，sizeCtl的值表示扩容阈值，阈值大小通常为当前容量大小的0.75（在ConcurrentHashMap中是用 n - (n >>> 2) 或者 (n << 1) - (n >>> 1) 替代n * 0.75来计算阈值），当table中key的数量大于等于阈值时，开启对table扩容，每次扩容table容量大小变为原来的两倍。

ConcurrentHashMap中支持多线程同时扩容，当线程在插入数据时，如果发现有其他线程正在对table进行扩容，则帮忙一起扩容；前面说过，扩容时每迁移完一个索引为index的hash桶中的key，将table[index]置为ForwardingNode节点，该节点的hash值为MOVED(-1)，当插入数据时，发现节点的hash值为-1，表示有其他线程正在进行扩容。

扩容时，从table的最右边开始迁移，即从table[n-1]往table[0]逐个迁移hash桶中的节点；每个线程扩容时每次从右到左至少获取MIN_TRANSFER_STRIDE个hash桶并且迁移桶中key；每迁移完一个hash桶中的全部key，将table[i]置为ForwardingNode节点，其hash值为-2，标记正在扩容且该hash桶已经迁移完毕。

另外当调用putAll函数插入数据时，以及将链表转换为红黑树结构时（如果table容量<64），也都会触发扩容，调用了tryPresize()函数

## 相关成员属性介绍

* private static final int MIN_TRANSFER_STRIDE = 16;
	> 扩容线程每次最少要迁移的hash桶数量，默认值为16
* private static int RESIZE_STAMP_BITS = 16;
	> 扩容过程中，sizeCtl数值中用于表示generation stamp的位数
	> 
	> 在扩容过程中，sizeCtl为负数，并且被划分为两个部分
	> 
	> 高RESIZE_STAMP_BITS位表示generation stamp，低(32 - RESIZE_STAMP_BITS)位表示正在辅助扩容的线程数
	> 
	> 为了保证sizeCtl一定为负数，generation stamp的最高位被置为1
* private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
	> 辅助扩容的最大线程数 (32 - RESIZE_STAMP_BITS为sizeCtl中用于表示辅助扩容线程数的位数)
* private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
	> 由于高RESIZE_STAMP_BITS位表示generation stamp，所以操作时需要对generation stamp进行移位,
	> 
	> 保存时需要先左移RESIZE_STAMP_SHIFT位，再与正在扩容的线程数进行按位或操作,
	> 
	> 获取时，需要先右移RESIZE_STAMP_SHIFT位
* private transient volatile int transferIndex;
	> 扩容索引，表示已经分配给扩容线程的table数组的索引位置。主要用来协调多个线程，并发安全的获取迁移任务（hash桶）
	> 
	> 每个准备辅助扩容的线程，都先从transferIndex索引位置开始，向前获取至少MIN_TRANSFER_STRIDE个hash桶，然后迁移获取的hash桶中的key
* private transient volatile Node<K,V>[] nextTable;
	> 扩容时用于保存key的数组结构，扩容完毕后置为null
	


## 源码分析
开启扩容操作在插入数据后的增加计数addCount函数中，
	
	// 判断是否需要扩容
        if (check >= 0) {
            Node<K,V>[] tab, nt; int n, sc;
			// 如果table中key数量大于等于阈值，且table!=null，且当前容量小于最大容量值，则扩容
            while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                   (n = tab.length) < MAXIMUM_CAPACITY) {
                int rs = resizeStamp(n);
                if (sc < 0) { //表示有其他线程正在扩容，则检查是否需要辅助扩容
					/**
                     * 1 (sc >>> RESIZE_STAMP_SHIFT) != rs ： 表示generation stamp不对
                     * 2 sc == rs + 1 ：表示什么？？？ 感觉也有问题
                     * 3 sc == rs + MAX_RESIZERS ： 感觉应该是sc == (rs << RESIZE_STAMP_SHIFT)  + MAX_RESIZERS 表示辅助扩容线程数达到最大
                     * 4 (nt = nextTable) == null ：表示nextTable正在初始化
                     * 5 transferIndex <= 0 ：表示所有hash桶均分配出去
                     */

                    //如果不需要帮其扩容，直接返回
                    if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0)
                        break;
					//CAS设置sizeCtl=sizeCtl+1    增加扩容线程数
                    if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                        transfer(tab, nt);  //帮其扩容
                }
				//第一个执行扩容操作的线程，将sizeCtl设置为：(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)
                else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                             (rs << RESIZE_STAMP_SHIFT) + 2))
                    transfer(tab, null);
                s = sumCount(); //每次扩容后检查占用率是否需要进行再一次扩容，因为扩容滞后于添加元素。
            }
        }

因为第一个扩容线程将sizeCtl置为(resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，所以sizeCtl的低32 - RESIZE_STAMP_BITS表示的实际上是正在扩容的线程数+1

在扩容前，sizeCtl为阈值，大于0，在并发时，使用CAS交换只有一个线程有机会设置成功并且执行nextTable的初始化，保证了nextTable也只会被初始化一次

	static final int resizeStamp(int n) {
        return Integer.numberOfLeadingZeros(n) | (1 << (RESIZE_STAMP_BITS - 1));
    }

此函数为计算扩容时的generation stamp，numberOfLeadingZeros返回二进制值中第一个1左边0的个数，由于在ConcurrentHashMap中容量大小为2的幂次方，所以每次扩容时，容量大小扩大一倍（乘2），numberOfLeadingZeros返回值就会减小1，generation stamp也就会减小1。
numberOfLeadingZeros的实现详解可以参考[http://blog.csdn.net/abcdef314159/article/details/70176707](http://blog.csdn.net/abcdef314159/article/details/70176707)

0 <= Integer.numberOfLeadingZeros(n) <= 32;(只有n为负数时返回值为0，这里n>=0，返回值不会等于0)

按位或运算是为了将generation stamp的最高位置为1，这样再将generation stamp左移RESIZE_STAMP_SHIFT后，第32位就为1，保证在扩容过程中sizeCtl为负数（RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS）

扩容函数如下：

	private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
        int n = tab.length, stride;
        // 计算每次迁移的hash桶个数
        if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
            stride = MIN_TRANSFER_STRIDE; // subdivide range（细分范围）
        // 这里初始化nextTable时，貌似没有像初始化table那样考虑并发初始化问题
        // 其实是在调用transfer(tab,null)处，通过CAS设置sizeClt值控制了并发问题，只有设置成功才可以调用transfer(tab,null)
        if (nextTab == null) {            // initiating
            try {
                @SuppressWarnings("unchecked")
                Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
                nextTab = nt;
            } catch (Throwable ex) {      // try to cope with OOME
                sizeCtl = Integer.MAX_VALUE;
                return;
            }
            nextTable = nextTab;
            transferIndex = n;  // 从右边向左开始迁移桶中node
        }
        int nextn = nextTab.length;
        ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
        boolean advance = true;
        boolean finishing = false; // to ensure sweep before committing nextTab
        for (int i = 0, bound = 0;;) {
            Node<K,V> f; int fh;
            while (advance) {
                int nextIndex, nextBound;
                //bound表示由当前线程负责迁移的hash桶的最后一个索引值（从右向左），--i>==bound，表示还在当前线程负责迁移的范围内，继续迁移
                if (--i >= bound || finishing)
                    advance = false;
                else if ((nextIndex = transferIndex) <= 0) {    //表示table中的hash桶已经全部分配完毕，所有桶均有线程在进行迁移
                    i = -1;
                    advance = false;
                }
                //cas无锁算法设置 transferIndex = transferIndex - stride
                //设置成功，表示table[transferIndex-1]--table[transferIndex-stride]这个闭区间内（stride）个hash桶由当前线程负责迁移
                //当迁移完bound(负责的最后一个桶索引值)这个桶后，尝试更新transferIndex，获取下一批待迁移的hash桶
                else if (U.compareAndSwapInt
                         (this, TRANSFERINDEX, nextIndex,
                          nextBound = (nextIndex > stride ?
                                       nextIndex - stride : 0))) {
                    bound = nextBound;
                    i = nextIndex - 1;
                    advance = false;
                }
            }
            if (i < 0 || i >= n || i + n >= nextn) {
                int sc;
                // 迁移完成，将table指向新的数组,sizeClt设置为扩容阈值
                // 这里貌似也没有考虑并发，没有通过CAS操作来设置这些值，其实此时只剩下最后一个扩容线程在完成此项工作，其他的辅助扩容线程均已退出
                if (finishing) {
                    nextTable = null;
                    table = nextTab;
                    sizeCtl = (n << 1) - (n >>> 1);
                    return;
                }
                /**
                 * 第一个扩容的线程，执行transfer方法之前，会设置 sizeCtl = (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)
                 * 后续帮其扩容的线程，执行transfer方法之前，会设置 sizeCtl = sizeCtl+1
                 * 每一个退出transfer的方法的线程，退出之前，会设置 sizeCtl = sizeCtl-1
                 * 那么最后一个线程退出时：
                 * 必然有sc == (resizeStamp(n) << RESIZE_STAMP_SHIFT) + 2)，即 (sc - 2) == resizeStamp(n) << RESIZE_STAMP_SHIFT
                 * https://www.jianshu.com/p/487d00afe6ca
                 */
                //不相等，说明不到最后一个线程，直接退出transfer方法
                if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                    if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                        return;
                    finishing = advance = true;
                    i = n; // recheck before commit
                }
            }
            else if ((f = tabAt(tab, i)) == null)
                advance = casTabAt(tab, i, null, fwd);
            else if ((fh = f.hash) == MOVED)
                advance = true; // already processed
            else {
                synchronized (f) {
                    if (tabAt(tab, i) == f) {
                        Node<K,V> ln, hn;
                        // 链表迁移
                        if (fh >= 0) {
                            int runBit = fh & n;
                            Node<K,V> lastRun = f;
                            for (Node<K,V> p = f.next; p != null; p = p.next) {
                                int b = p.hash & n;
                                if (b != runBit) {
                                    runBit = b;
                                    lastRun = p;
                                }
                            }
                            if (runBit == 0) {
                                ln = lastRun;
                                hn = null;
                            }
                            else {
                                hn = lastRun;
                                ln = null;
                            }
                            for (Node<K,V> p = f; p != lastRun; p = p.next) {
                                int ph = p.hash; K pk = p.key; V pv = p.val;
                                if ((ph & n) == 0)
                                    ln = new Node<K,V>(ph, pk, pv, ln);
                                else
                                    hn = new Node<K,V>(ph, pk, pv, hn);
                            }
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn); //为什么直接设置为i+n,见文章画图说明
                            setTabAt(tab, i, fwd);      //迁移完毕设置为ForwardingNode节点，节点hash值设置为MOVED（-1）
                            advance = true;
                        }
                        //红黑树迁移
                        else if (f instanceof TreeBin) {
                            TreeBin<K,V> t = (TreeBin<K,V>)f;
                            TreeNode<K,V> lo = null, loTail = null;
                            TreeNode<K,V> hi = null, hiTail = null;
                            int lc = 0, hc = 0;
                            for (Node<K,V> e = t.first; e != null; e = e.next) {
                                int h = e.hash;
                                TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                                if ((h & n) == 0) {
                                    if ((p.prev = loTail) == null)
                                        lo = p;
                                    else
                                        loTail.next = p;
                                    loTail = p;
                                    ++lc;
                                }
                                else {
                                    if ((p.prev = hiTail) == null)
                                        hi = p;
                                    else
                                        hiTail.next = p;
                                    hiTail = p;
                                    ++hc;
                                }
                            }
                            ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                            hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                            setTabAt(nextTab, i, ln);
                            setTabAt(nextTab, i + n, hn);
                            setTabAt(tab, i, fwd);
                            advance = true;
                        }
                    }
                }
            }
        }
    }

### rehash过程
rehash过程中以下这两行代码的理解，为什么可以直接一部分保留在当前索引位置的hash桶，另一部分的位置索引为当前索引+扩容前的大小？这里可以看到每次扩容后的大小保持为2的幂次方展现的优势。

	setTabAt(nextTab, i, ln);
    setTabAt(nextTab, i + n, hn);

例如我们从16扩展为32时，具体的变化如下所示：
![](https://i.imgur.com/OFa6YCa.png)

如图假设hashcode为10101，当n为16时索引为（(n-1)&hashcode=0101），当n为32时，则索引为（(n-1)&hashcode=10101==0101（当前index值）+10000（扩容前的容量值）），索引位置index为当前index直接加扩充前的容量值

计算索引时，需要用hashcode与（n-1）进行位与操作，（n-1）的二进制表示中为1的个数为低m=log(n)位，所以hashCode二进制表示中低m位才有效，参与索引计算。当扩容时n<<1，因为n变为2倍，那么n-1的mask范围在高位多1bit(红色)，hashcode中参与索引计算的有效位变为m+1，且最高有效位（即第m+1位）决定了索引值，当hashcode中第m+1位为0时，新的索引值与扩容前相同，当hashcode中第m+1位为1时，新的索引值为扩容前索引值+2的m次方（2的m次方即为扩容前的容量大小n）

因此rehash时，不需要重新计算散列索引值，根据扩容后hashcode的最高有效位（新增的那个bit位）是1还是0，将原来hash桶中的链表或者红黑树分为两部分，最高有效位为0的部分索引值与扩容前相同，保留在当前hash桶中，最高有效位为1的部分在新的table中的索引位置为i+n

可以看看下图为16扩充为32的resize示意图：
![](https://i.imgur.com/Nd9BRL2.png)
图中蓝色节点表示hashcode中新增的那个bit位为0，灰色节点表示新增的那个bit位为1

将链表根据最高有效位划分为两部分的代码如下所示：

		int runBit = fh & n;
		Node<K,V> lastRun = f;
		for (Node<K,V> p = f.next; p != null; p = p.next) {
		    int b = p.hash & n;
		    if (b != runBit) {
		        runBit = b;
		        lastRun = p;
		    }
		}
		if (runBit == 0) {
		    ln = lastRun;
		    hn = null;
		}
		else {
		    hn = lastRun;
		    ln = null;
		}
		for (Node<K,V> p = f; p != lastRun; p = p.next) {
		    int ph = p.hash; K pk = p.key; V pv = p.val;
		    if ((ph & n) == 0)
		        ln = new Node<K,V>(ph, pk, pv, ln);
		    else
		        hn = new Node<K,V>(ph, pk, pv, hn);
		}

示意图如下图所示：

![](https://i.imgur.com/bJ2Qw6H.png)

假设图中蓝色节点的最高有效位为0，红色节点的最高有效位为1，

* 第一遍遍历后，设置lastRun和对应的runBit，lastRun指向G，runBit为1，所以hn为节点G，ln为null；
* 重新遍历链表，以lastRun节点为终止条件，将最高有效位为0的节点插入ln头部，为1的节点插入hn头部；完成后ln链表中的节点都是倒序的，hn链表中的节点lastRun以前的也都是倒序；
* 最后将ln指针指向的链表插入当前槽位，hn指针指向的链表插入当前槽位index+n的槽位，完成rehash操作。

### 辅助扩容
当线程对ConcurrentHashMap操作时，发现有其他线程正在进行扩容，则会帮助一起扩容

	final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) {
				// 检查是否需要帮忙扩容，如若不满足条件则返回
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                    break;
				// 将正在扩容的线程数加1
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                    transfer(tab, nextTab);
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
helpTransfer函数和前面介绍的addCount函数中代码基本相同

还有一个函数TryPresize，可能会触发table初始化或者扩容操作，其函数实现和前面介绍的内容基本相同，就不重复贴代码了

# 获取操作--get()函数实现
从ConcurrentHashMap中查找key，存在返回对应的value，否则返回null，无需加锁；当table正在扩容时，如果key所在的hash桶已经被迁移到nextTable中，则到nextTable中进行查找

	public V get(Object key) {
        Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
        int h = spread(key.hashCode());
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
            if ((eh = e.hash) == h) {
                if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                    return e.val;
            }
            /**
             * 此处的find操作会表现出多态性，出现动态调用，根据Node的实际类型调用相应的Node子类中的find函数
             * eh < 0 有以下几种情况：
             * （1）eh == MOVED(-1)，此节点为ForwardingNode，表示正在进行扩容操作，该hash桶已经被迁移至nextTable，会调用ForwardingNode子类的find函数
             * （2）eh == TREEBIN(-2)，此节点为TreeBin，表示此hash桶中节点保存形式为红黑树结构，调用TreeBin子类的find函数
             * （3）eh == RESERVED(-3)，此节点为ReservationNode，表示为一保留节点，调用ReservationNode子类的find函数（返回 null）
             */
            else if (eh < 0)
                return (p = e.find(h, key)) != null ? p.val : null;
            while ((e = e.next) != null) {
                if (e.hash == h &&
                    ((ek = e.key) == key || (ek != null && key.equals(ek))))
                    return e.val;
            }
        }
        return null;
    }

在ForwardingNode、TreeBin等Node子类中均有重新实现find函数

# 计数--size()函数实现
当向ConcurrentHashMap中插入和删除数据时，会调用addCount函数增加或减少map中key的计数，ConcurrentHashMap中使用一个volatile类型的变量baseCount记录元素的个数，在并发量很高时，如果存在多个线程同时执行CAS修改baseCount值，CAS操作失败的线程会使用CounterCell数组记录元素的个数；

因此计算size时，使用baseCount大小加上CounterCell数组中各个元素中保存的key大小；由于可能存在并发操作，其实在高并发下，size大小可能会有误差。

## 相关成员属性介绍
* private transient volatile long baseCount;
	> map中key的计数,CAS操作
* private transient volatile CounterCell[] counterCells;
	> 当CAS操作baseCount失败时，使用CounterCell数组来记录元素个数
* private transient volatile int cellsBusy;
	> 用于扩容或者创建counterCells时的CAS自旋锁变量
	> 
	> 线程需要扩容或者创建counterCells时，先使用CAS操作将cellsBusy从0设置为1，设置成功的线程可以扩容或者创建counterCells，保证线程安全

## 源码分析
### 增加或减少计数
增加或减少计数的代码在addCount函数中

		CounterCell[] as; long b, s;
		// 如果counterCells != null，则直接在counterCells中记录元素个数
        if ((as = counterCells) != null ||
            !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
            // 初始化时counterCells为空，在并发量很高时，如果存在两个线程同时执行CAS修改baseCount值，
            // 则失败的线程会继续执行方法体中的逻辑，使用CounterCell记录元素个数的变化；
            CounterCell a; long v; int m;
            boolean uncontended = true;
            if (as == null || (m = as.length - 1) < 0 ||
                (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
                !(uncontended =
                  U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
                // 如果CounterCell数组为null或者大小为空数组；或者对应的数组元素为null；或者CAS设置元素值失败，则调用fullAddCount()
                fullAddCount(x, uncontended);
                return;
            }
            if (check <= 1)
                return;
            s = sumCount();

fullAddCount函数如下：
	
	private final void fullAddCount(long x, boolean wasUncontended) {
        int h;
        // 初始时ThreadLocalRandom.getProbe()为0
        if ((h = ThreadLocalRandom.getProbe()) == 0) {
            ThreadLocalRandom.localInit();      // force initialization
            h = ThreadLocalRandom.getProbe();
            wasUncontended = true;
        }
        boolean collide = false;                // True if last slot nonempty
        for (;;) {
            CounterCell[] as; CounterCell a; int n; long v;
            // 如果counterCells不为null，则找一个数组位置，在其元素CounterCell对象中记录元素计数
            if ((as = counterCells) != null && (n = as.length) > 0) {
                if ((a = as[(n - 1) & h]) == null) {    // 由于CounterCell数组大小n始终为2的幂次方，所以用(n-1)&h替代h%n
                    // 如果插入位置的CounterCell对象为null，则CAS修改cellsBusy，修改成功则插入一个对象保存计数，保证线程安全
                    if (cellsBusy == 0) {            // Try to attach new Cell
                        CounterCell r = new CounterCell(x); // Optimistic create
                        if (cellsBusy == 0 &&
                            U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                            boolean created = false;
                            try {               // Recheck under lock
                                CounterCell[] rs; int m, j;
                                if ((rs = counterCells) != null &&
                                    (m = rs.length) > 0 &&
                                    rs[j = (m - 1) & h] == null) {
                                    rs[j] = r;
                                    created = true;
                                }
                            } finally {
                                cellsBusy = 0;
                            }
                            if (created)
                                break;
                            continue;           // Slot is now non-empty
                        }
                    }
                    collide = false;
                }
                else if (!wasUncontended)       // CAS already known to fail
                    wasUncontended = true;      // Continue after rehash
                // 如果对应的位置存在CounterCell对象，则CAS修改计数，修改成功则返回
                else if (U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))
                    break;
                else if (counterCells != as || n >= NCPU)
                    collide = false;            // At max size or stale
                else if (!collide)
                    collide = true;
                else if (cellsBusy == 0 &&
                         U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                    // 如果经常CAS设置值失败，则扩容CounterCell数组
                    try {
                        if (counterCells == as) {// Expand table unless stale
                            CounterCell[] rs = new CounterCell[n << 1];
                            for (int i = 0; i < n; ++i)
                                rs[i] = as[i];
                            counterCells = rs;
                        }
                    } finally {
                        cellsBusy = 0;
                    }
                    collide = false;
                    continue;                   // Retry with expanded table
                }
				// 修改h，换一个数组位置记录计数
                h = ThreadLocalRandom.advanceProbe(h);
            }
            // 如果CounterCell数组counterCells为空，进行初始化，并插入对应的记录数
            // 通过CAS设置cellsBusy字段，只有设置成功的线程才能初始化CounterCell数组，保证并发安全
            else if (cellsBusy == 0 && counterCells == as &&
                     U.compareAndSwapInt(this, CELLSBUSY, 0, 1)) {
                boolean init = false;
                try {                           // Initialize table
                    if (counterCells == as) {
                        CounterCell[] rs = new CounterCell[2];
                        rs[h & 1] = new CounterCell(x);
                        counterCells = rs;
                        init = true;
                    }
                } finally {
                    cellsBusy = 0;
                }
                if (init)
                    break;
            }
            // 如果通过CAS设置cellsBusy字段失败的话，则继续尝试通过CAS修改baseCount字段，
            // 如果修改baseCount字段成功的话，就退出循环，否则继续循环插入CounterCell对象；
            else if (U.compareAndSwapLong(this, BASECOUNT, v = baseCount, v + x))
                break;                          // Fall back on using base
        }
    }

### 统计元素个数
baseCount加上数组counterCells中各个CounterCell对象中保存的记录之和

	public int size() {
        long n = sumCount();
        return ((n < 0L) ? 0 :
                (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
                (int)n);
    }
	final long sumCount() {
        CounterCell[] as = counterCells; CounterCell a;
        long sum = baseCount;
        if (as != null) {
            for (int i = 0; i < as.length; ++i) {
                if ((a = as[i]) != null)
                    sum += a.value;
            }
        }
        return sum;
    }

# 参考链接
[1] [http://wuzhaoyang.me/2016/09/05/java-collection-map-2.html](http://wuzhaoyang.me/2016/09/05/java-collection-map-2.html)

[2] [http://blog.csdn.net/u010412719/article/details/52145145](http://blog.csdn.net/u010412719/article/details/52145145)

[3] [https://www.cnblogs.com/yangming1996/p/8031199.html](https://www.cnblogs.com/yangming1996/p/8031199.html)

[4] [http://blog.csdn.net/u010723709/article/details/48007881](http://blog.csdn.net/u010723709/article/details/48007881)