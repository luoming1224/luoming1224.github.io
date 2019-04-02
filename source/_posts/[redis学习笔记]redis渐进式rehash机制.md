---
title: '[redis学习笔记]redis渐进式rehash机制'
date: 2018-11-12 16:17:28
categories: 'redis学习笔记'
tags: [redis, 扩容, rehash]
---
在Redis中，键值对（Key-Value Pair）存储方式是由字典（Dict）保存的，而字典底层是通过哈希表来实现的。通过哈希表中的节点保存字典中的键值对。我们知道当HashMap中由于Hash冲突（负载因子）超过某个阈值时，出于链表性能的考虑，会进行Resize的操作。Redis也一样。

在redis的具体实现中，使用了一种叫做渐进式哈希(rehashing)的机制来提高字典的缩放效率，避免 rehash 对服务器性能造成影响，渐进式 rehash 的好处在于它采取分而治之的方式， 将 rehash 键值对所需的计算工作均摊到对字典的每个添加、删除、查找和更新操作上， 从而避免了集中式 rehash 而带来的庞大计算量。

<!--more-->

# 字典结构
## 哈希表节点
	typedef struct dictEntry {
	    void *key;				//键
	    union {
	        void *val;			//值
	        uint64_t u64;
	        int64_t s64;
	        double d;
	    } v;
	    struct dictEntry *next; //指向下一个节点，形成链表
	} dictEntry;

从哈希表节点结构中，可以看出，在redis中解决hash冲突的方式为采用链地址法。key和v分别用于保存键值对的键和值。
	
## 哈希表

	/* This is our hash table structure. Every dictionary has two of this as we
	 * implement incremental rehashing, for the old to the new table. */
	typedef struct dictht {
	    dictEntry **table;
	    unsigned long size;
	    unsigned long sizemask;
	    unsigned long used;
	} dictht;

* table：哈希表数组，数组的每个项是dictEntry链表的头结点指针
* size：哈希表大小；在redis的实现中，size也是触发扩容的阈值
* sizemask：哈希表大小掩码，用于计算索引值；总是等于 size-1 ；
* used：哈希表中保存的节点的数量

##字典

	typedef struct dict {
	    dictType *type;
	    void *privdata;
	    dictht ht[2];
	    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
	    unsigned long iterators; /* number of iterators currently running */
	} dict;

* type 属性是一个指向 dictType 结构的指针， 每个 dictType 结构保存了一簇用于操作特定类型键值对的函数， Redis 会为用途不同的字典设置不同的类型特定函数。
* 而 privdata 属性则保存了需要传给那些类型特定函数的可选参数。
* dictht ht[2]：在字典内部，维护了两张哈希表。 一般情况下， 字典只使用 ht[0] 哈希表， ht[1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用。
* rehashidx：和 rehash 有关的属性，它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 -1 。

type 属性和 privdata 属性是针对不同类型的键值对， 为创建多态字典而设置的。

# rehash检查
随着操作的不断执行， 哈希表保存的键值对会逐渐地增多或者减少， 为了让哈希表的负载因子（load factor）维持在一个合理的范围之内， 当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。

## 扩容
redis中，每次插入键值对时，都会检查是否需要扩容。如果满足扩容条件，则进行扩容。

在向redis中添加键时都会依次调用dictAddRaw --> _dictKeyIndex --> _dictExpandIfNeeded函数，在_dictExpandIfNeeded函数中会判断是否需要扩容。

	/* Expand the hash table if needed */
	static int _dictExpandIfNeeded(dict *d)
	{
	    /* Incremental rehashing already in progress. Return. */
	    // 如果正在进行渐进式扩容，则返回OK
	    if (dictIsRehashing(d)) return DICT_OK;
	
	    /* If the hash table is empty expand it to the initial size. */
	    // 如果哈希表ht[0]的大小为0，则初始化字典
	    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);
	
	    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
	     * table (global setting) or we should avoid it but the ratio between
	     * elements/buckets is over the "safe" threshold, we resize doubling
	     * the number of buckets. */
	    /*
	     * 如果哈希表ht[0]中保存的key个数与哈希表大小的比例已经达到1:1，即保存的节点数已经大于哈希表大小
	     * 且redis服务当前允许执行rehash，或者保存的节点数与哈希表大小的比例超过了安全阈值（默认值为5）
	     * 则将哈希表大小扩容为原来的两倍
	     */
	    if (d->ht[0].used >= d->ht[0].size &&
	        (dict_can_resize ||
	         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
	    {
	        return dictExpand(d, d->ht[0].used*2);
	    }
	    return DICT_OK;
	}

从上面代码和注释可以看到，如果没有进行初始化或者满足扩容条件则对字典进行扩容。

先来看看字典初始化，**在redis中字典中的hash表也是采用延迟初始化**策略，在创建字典的时候并没有为哈希表分配内存，只有当第一次插入数据时，才真正分配内存。看看字典创建函数dictCreate。

	/* Create a new hash table */
	dict *dictCreate(dictType *type,
	        void *privDataPtr)
	{
	    dict *d = zmalloc(sizeof(*d));
	
	    _dictInit(d,type,privDataPtr);
	    return d;
	}
	
	/* Initialize the hash table */
	int _dictInit(dict *d, dictType *type,
	        void *privDataPtr)
	{
	    _dictReset(&d->ht[0]);
	    _dictReset(&d->ht[1]);
	    d->type = type;
	    d->privdata = privDataPtr;
	    d->rehashidx = -1;
	    d->iterators = 0;
	    return DICT_OK;
	}
	
	static void _dictReset(dictht *ht)
	{
	    ht->table = NULL;
	    ht->size = 0;
	    ht->sizemask = 0;
	    ht->used = 0;
	}

从上面的创建过程可以看出，ht[0].table为NULL，且ht[0].size为0，直到第一次插入数据时，才调用dictExpand函数初始化。

我们再看看dict_can_resize字段，该字段在dictEnableResize和dictDisableResize函数中分别赋值1和0，在updateDictResizePolicy函数中会调用者两个函数。
	
	void updateDictResizePolicy(void) {
	    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
	        dictEnableResize();
	    else
	        dictDisableResize();
	}

	void dictEnableResize(void) {
	    dict_can_resize = 1;
	}
	
	void dictDisableResize(void) {
	    dict_can_resize = 0;
	}

而在redis中每次开始执行aof文件重写或者开始生成新的RDB文件或者执行aof重写/生成RDB的子进程结束时，都会调用updateDictResizePolicy函数，所以从该函数中，也可以看出来，如果当前没有子进程在执行aof文件重写或者生成RDB文件，则运行进行字典扩容；否则禁止字典扩容。

综上，字典扩容需要同时满足如下两个条件：

（1）哈希表中保存的key数量超过了哈希表的大小（可以看出size既是哈希表大小，同时也是扩容阈值）

（2）当前没有子进程在执行aof文件重写或者生成RDB文件；或者保存的节点数与哈希表大小的比例超过了安全阈值（默认值为5）

也可以如下理解：

当以下条件中的任意一个被满足时， 程序会自动开始对哈希表执行扩展操作：

* 服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 1 ；

* 服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 5 ；


## 缩容
 当哈希表的负载因子小于 0.1 时， 程序自动开始对哈希表执行收缩操作。

在周期函数serverCron中，调用databasesCron函数，该函数中会调用tryResizeHashTables函数检查用于保存键值对的redis数据库字典是否需要缩容。如果需要则调用dictResize进行缩容，dictResize函数中也是调用dictExpand函数。

看看databasesCron中相关部分

	if (server.rdb_child_pid == -1 && server.aof_child_pid == -1) {
        /* We use global counters so if we stop the computation at a given
         * DB we'll be able to start from the successive in the next
         * cron loop iteration. */
        static unsigned int resize_db = 0;
        static unsigned int rehash_db = 0;
        int dbs_per_call = CRON_DBS_PER_CALL;
        int j;

        /* Don't test more DBs than we have. */
        if (dbs_per_call > server.dbnum) dbs_per_call = server.dbnum;

        /* Resize */
        for (j = 0; j < dbs_per_call; j++) {
            tryResizeHashTables(resize_db % server.dbnum);
            resize_db++;
        }
可以看到要检查是否需要缩容的前提也是当前没有子进程执行aof重写或者生成RDB文件。

	/* If the percentage of used slots in the HT reaches HASHTABLE_MIN_FILL
	 * we resize the hash table to save memory */
	void tryResizeHashTables(int dbid) {
	    if (htNeedsResize(server.db[dbid].dict))
	        dictResize(server.db[dbid].dict);
	    if (htNeedsResize(server.db[dbid].expires))
	        dictResize(server.db[dbid].expires);
	}
	
	/* Hash table parameters */
	#define HASHTABLE_MIN_FILL        10      /* Minimal hash table fill 10% */
	int htNeedsResize(dict *dict) {
	    long long size, used;
	
	    size = dictSlots(dict);
	    used = dictSize(dict);
	    return (size > DICT_HT_INITIAL_SIZE &&
	            (used*100/size < HASHTABLE_MIN_FILL));
	}
	
	/* Resize the table to the minimal size that contains all the elements,
	 * but with the invariant of a USED/BUCKETS ratio near to <= 1 */
	int dictResize(dict *d)
	{
	    int minimal;
	
	    if (!dict_can_resize || dictIsRehashing(d)) return DICT_ERR;
	    minimal = d->ht[0].used;
	    if (minimal < DICT_HT_INITIAL_SIZE)
	        minimal = DICT_HT_INITIAL_SIZE;
	    return dictExpand(d, minimal);
	}

从htNeedsResize函数中可以看到，当哈希表保存的key数量与哈希表的大小的比例小于10%时需要缩容。最小容量为4。

从dictResize函数中可以看到缩容时，缩容后的哈希表大小为当前哈希表中key的数量，当然经过dictExpand函数中_dictNextPower函数计算后，缩容后的大小为第一个大于等于当前key数量的2的n次方。最小容量为4。同样从dictResize函数中可以看到，如果当前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令，则不进行缩容（有篇文章中提到缩容时没有考虑bgsave，该说法是错误的）。

# 渐进式rehash
## 渐进式rehash初始化
从上面可以看到，不管是扩容还是缩容，最终都是调用dictExpand函数来完成。看看dictExpand函数实现。

	/* Expand or create the hash table */
	int dictExpand(dict *d, unsigned long size)
	{
	    /* the size is invalid if it is smaller than the number of
	     * elements already inside the hash table */
	    if (dictIsRehashing(d) || d->ht[0].used > size)
	        return DICT_ERR;
	
	    //计算新的哈希表大小，使得新的哈希表大小为一个2的n次方;大于等于size的第一个2的n次方
	    dictht n; /* the new hash table */
	    unsigned long realsize = _dictNextPower(size);
	
	    /* Rehashing to the same table size is not useful. */
	    if (realsize == d->ht[0].size) return DICT_ERR;
	
	    /* Allocate the new hash table and initialize all pointers to NULL */
	    n.size = realsize;
	    n.sizemask = realsize-1;
	    n.table = zcalloc(realsize*sizeof(dictEntry*));
	    n.used = 0;
	
	    /* Is this the first initialization? If so it's not really a rehashing
	     * we just set the first hash table so that it can accept keys. */
	    // 并非扩容，而是第一次初始化；前面说了，第一次初始化也是通过该函数完成
	    if (d->ht[0].table == NULL) {
	        d->ht[0] = n;
	        return DICT_OK;
	    }
	
	    /* Prepare a second hash table for incremental rehashing */
	    // 为渐进式扩容作准备，下面两个赋值非常重要
	    d->ht[1] = n;
	    d->rehashidx = 0;
	    return DICT_OK;
	}

可以看到该函数计算一个新的哈希表大小，满足2的n次方，为什么要满足2的n次方？因为哈希表掩码sizemask为size-1，当size满足2的n次方时，计算每个key的索引值时只需要用key的hash值与掩码sizemask进行位与操作，替代求余操作，计算更快。

然后分配了一个新的哈希表，为该哈希表分配了新的大小的内存。最后将该哈希表赋值给字典的ht[1]，然后将rehashidx赋值为0，打开渐进式rehash标志。同时该值也标志渐进式rehash当前已经进行到了哪个hash槽。

从该函数中，我们并没有看到真正执行哈希表rehash的相关操作，只是分配了一个新的哈希表就结束了。我们知道哈希表rehash需要遍历原有的整个哈希表，对原有的所有key进行重新hash，存放到新的哈希槽。

在redis的实现中，没有集中的将原有的key重新rehash到新的槽中，而是分解到各个命令的执行中，以及周期函数中。

## 操作辅助rehash
在redis中每一个增删改查命令中都会判断数据库字典中的哈希表是否正在进行渐进式rehash，如果是则帮助执行一次。
	
	dictEntry *dictAddRaw(dict *d, void *key, dictEntry **existing)
	{
	    long index;
	    dictEntry *entry;
	    dictht *ht;
	
	    if (dictIsRehashing(d)) _dictRehashStep(d);
		......
	}

类似的在dictFind、dictGenericDelete、dictGetRandomKey、dictGetSomeKeys等函数中都有以下语句判断是否正在进行渐进式rehash。

	if (dictIsRehashing(d)) _dictRehashStep(d);
	//dictIsRehashing(d)定义如下，rehashidx不等于-1即表示正在进行渐进式rehash
	#define dictIsRehashing(d) ((d)->rehashidx != -1)

_dictRehashStep函数的定义如下：

	/*
	 * 此函数仅执行一步hash表的重散列，并且仅当没有安全迭代器绑定到哈希表时。
	 * 当我们在重新散列中有迭代器时，我们不能混淆打乱两个散列表的数据，否则某些元素可能被遗漏或重复遍历。
	 *
	 * 该函数被在字典中查找或更新等普通操作调用，以致字典中的数据能自动的从哈系表１迁移到哈系表２
	 */
	static void _dictRehashStep(dict *d) {
	    if (d->iterators == 0) dictRehash(d,1);
	}


## 定时辅助rehash
虽然redis实现了在读写操作时，辅助服务器进行渐进式rehash操作，但是如果服务器比较空闲，redis数据库将很长时间内都一直使用两个哈希表。所以在redis周期函数中，如果发现有字典正在进行渐进式rehash操作，则会花费**1毫秒**的时间，帮助一起进行渐进式rehash操作。

在databasesCron函数中，实现如下：

	/* Rehash */
        if (server.activerehashing) {
            for (j = 0; j < dbs_per_call; j++) {
                int work_done = incrementallyRehash(rehash_db);
                if (work_done) {
                    /* If the function did some work, stop here, we'll do
                     * more at the next cron loop. */
                    break;
                } else {
                    /* If this db didn't need rehash, we'll try the next one. */
                    rehash_db++;
                    rehash_db %= server.dbnum;
                }
            }
        }

前提是配置了activerehashing，允许服务器在周期函数中辅助进行渐进式rehash，该参数默认值是1。

	/* Our hash table implementation performs rehashing incrementally while
	 * we write/read from the hash table. Still if the server is idle, the hash
	 * table will use two tables for a long time. So we try to use 1 millisecond
	 * of CPU time at every call of this function to perform some rehahsing.
	 *
	 * The function returns 1 if some rehashing was performed, otherwise 0
	 * is returned. */
	int incrementallyRehash(int dbid) {
	    /* Keys dictionary */
	    if (dictIsRehashing(server.db[dbid].dict)) {
	        dictRehashMilliseconds(server.db[dbid].dict,1);
	        return 1; /* already used our millisecond for this loop... */
	    }
	    /* Expires */
	    if (dictIsRehashing(server.db[dbid].expires)) {
	        dictRehashMilliseconds(server.db[dbid].expires,1);
	        return 1; /* already used our millisecond for this loop... */
	    }
	    return 0;
	}

	/* Rehash for an amount of time between ms milliseconds and ms+1 milliseconds */
	int dictRehashMilliseconds(dict *d, int ms) {
	    long long start = timeInMilliseconds();
	    int rehashes = 0;
	
	    while(dictRehash(d,100)) {
	        rehashes += 100;
	        if (timeInMilliseconds()-start > ms) break;
	    }
	    return rehashes;
	}

## 渐进式rehash实现
从上面可以看到，不管是在操作中辅助rehash执行，还是在周期函数中辅助执行，最终都是调用dictRehash函数。
	
	/* Performs N steps of incremental rehashing. Returns 1 if there are still
	 * keys to move from the old to the new hash table, otherwise 0 is returned.
	 *
	 * Note that a rehashing step consists in moving a bucket (that may have more
	 * than one key as we use chaining) from the old to the new hash table, however
	 * since part of the hash table may be composed of empty spaces, it is not
	 * guaranteed that this function will rehash even a single bucket, since it
	 * will visit at max N*10 empty buckets in total, otherwise the amount of
	 * work it does would be unbound and the function may block for a long time. */
	int dictRehash(dict *d, int n) {
	    int empty_visits = n*10; /* Max number of empty buckets to visit. */
	    if (!dictIsRehashing(d)) return 0;
	
	    while(n-- && d->ht[0].used != 0) {
	        dictEntry *de, *nextde;
	
	        /* Note that rehashidx can't overflow as we are sure there are more
	         * elements because ht[0].used != 0 */
	        assert(d->ht[0].size > (unsigned long)d->rehashidx);
	        while(d->ht[0].table[d->rehashidx] == NULL) {
	            d->rehashidx++;
	            if (--empty_visits == 0) return 1;
	        }
	        de = d->ht[0].table[d->rehashidx];
	        /* Move all the keys in this bucket from the old to the new hash HT */
	        while(de) {
	            uint64_t h;
	
	            nextde = de->next;
	            /* Get the index in the new hash table */
	            h = dictHashKey(d, de->key) & d->ht[1].sizemask;
	            de->next = d->ht[1].table[h];
	            d->ht[1].table[h] = de;
	            d->ht[0].used--;
	            d->ht[1].used++;
	            de = nextde;
	        }
	        d->ht[0].table[d->rehashidx] = NULL;
	        d->rehashidx++;
	    }
	
	    /* Check if we already rehashed the whole table... */
	    if (d->ht[0].used == 0) {
	        zfree(d->ht[0].table);
	        d->ht[0] = d->ht[1];
	        _dictReset(&d->ht[1]);
	        d->rehashidx = -1;
	        return 0;
	    }
	
	    /* More to rehash... */
	    return 1;
	}

## 渐进式rehash小结
在redis中，扩展或收缩哈希表需要将 ht[0] 里面的所有键值对 rehash 到 ht[1] 里面， 但是， 这个 rehash 动作并不是一次性、集中式地完成的， 而是分多次、渐进式地完成的。为了避免 rehash 对服务器性能造成影响， 服务器不是一次性将 ht[0] 里面的所有键值对全部 rehash 到 ht[1] ， 而是分多次、渐进式地将 ht[0] 里面的键值对慢慢地 rehash 到 ht[1] 。

以下是哈希表渐进式 rehash 的详细步骤：

（1）为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。

（2）在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 0 ， 表示 rehash 工作正式开始。

（3）在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值增一。

（4）随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成。

渐进式 rehash 的好处在于它采取分而治之的方式， 将 rehash 键值对所需的计算工作均滩到对字典的每个添加、删除、查找和更新操作上， 从而避免了集中式 rehash 而带来的庞大计算量。

# 渐进式 rehash 执行期间的哈希表操作

因为在进行渐进式 rehash 的过程中， 字典会同时使用 ht[0] 和 ht[1] 两个哈希表， 所以在渐进式 rehash 进行期间， 字典的删除（delete）、查找（find）、更新（update）等操作会在两个哈希表上进行： 比如说， 要在字典里面查找一个键的话， 程序会先在 ht[0] 里面进行查找， 如果没找到的话， 就会继续到 ht[1] 里面进行查找， 诸如此类。

另外， 在渐进式 rehash 执行期间， 新添加到字典的键值对一律会被保存到 ht[1] 里面， 而 ht[0] 则不再进行任何添加操作： 这一措施保证了 ht[0] 包含的键值对数量会只减不增， 并随着 rehash 操作的执行而最终变成空表。

#渐进式rehash带来的问题
渐进式rehash避免了redis阻塞，可以说非常完美，但是由于在rehash时，需要分配一个新的hash表，在rehash期间，同时有两个hash表在使用，会使得redis内存使用量瞬间突增，在Redis 满容状态下由于Rehash会导致大量Key驱逐。

我们在生产上也遇到过这种情景。下面的博文《美团针对Redis Rehash机制的探索和实践》也有详细分析。

# 参考资料
* [美团针对Redis Rehash机制的探索和实践](https://tech.meituan.com/Redis_Rehash_Practice_Optimization.html)
* [rehash](http://redisbook.com/preview/dict/rehashing.html)
* [渐进式 rehash](http://redisbook.com/preview/dict/incremental_rehashing.html)
* [浅谈Redis中的Rehash机制](https://blog.csdn.net/cqk0100/article/details/80400811)
* [redis 字典的实现](https://cloud.tencent.com/developer/article/1005121)