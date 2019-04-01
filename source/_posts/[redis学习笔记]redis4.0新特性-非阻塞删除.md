---
title: '[redis学习笔记]redis4.0新特性-非阻塞删除'
date: 2018-11-11 19:51:10
categories: 'redis学习笔记'
tags: [redis, 4.0新特性, 惰性删除, unlink]
---
Redis作为一个单线程模型的服务，当执行一些耗时的命令时，比如使用DEL删除一个大key（元素超大的集合类型key），或者使用FLUSHDB 和 FLUSHALL 清空数据库，会造成redis阻塞，影响redis性能，甚至导致集群发生故障转移。另外redis在删除过期数据或因内存超过容量淘汰内存数据时，也有可能因为大key导致redis阻塞。

为了解决以上问题，redis 4.0 引入了惰性删除lazyfree的机制，它可以将删除键或数据库的操作放在后台线程里执行，删除对象时只是进行逻辑删除，从而尽可能地避免服务器阻塞。

<!--more-->

# lazyfree使用
针对以上两种场景，redis分别新增了几个命令和配置选项，同时lazyfree的使用分为2类：第一类是与DEL命令对应的主动删除，第二类是过期key删除、key淘汰删除。

## 主动删除键使用lazyfree
### unlink
UNLINK命令是与DEL一样删除key功能的lazy free实现。

唯一不同是，UNLINK在删除集合类型键时，如果集合键的元素个数大于64个，主线程中只是把待删除键从数据库字典中摘除，会把真正的内存释放操作，给单独的bio来操作。如果元素个数较少（少于64个）或者是String类型，也会在主线程中直接删除。

### flushall/flushdb async
对于清空数据库命令flushall/flushdb，添加了async异步清理选项，使得redis在清空数据库时都是异步操作。

实现逻辑是为数据库新建一个新的空的字典，把原有旧的数据库字典给后台线程来逐一删除其中的数据，释放内存。

## 被动删除键使用lazyfree
lazy free应用于被动删除中，目前有4种场景，每种场景对应一个配置参数； 默认都是关闭。
### lazyfree-lazy-eviction
针对redis内存使用达到maxmeory，并设置有淘汰策略时；在被动淘汰键时，是否采用lazy free机制；

因为此场景开启lazy free, **可能使用淘汰键的内存释放不及时，导致redis内存超用**，超过maxmemory的限制。此场景使用时，请结合业务测试。

### slave-lazy-flush
针对slave进行全量数据同步，slave在加载master的RDB文件前，会运行flushall来清理自己的数据场景；

参数设置决定是否采用异步flush机制。异步flush清空从节点本地数据库，**可减少全量同步耗时，从而减少主库因输出缓冲区爆涨引起的内存使用增长。**

### lazyfree-lazy-expire
针对设置有TTL的键，达到过期时间后，被redis清理删除时是否采用lazy free机制；此场景建议开启，因为TTL本身是自适应调整速度的。

### lazyfree-lazy-server-del
内部删除选项，针对有些指令在处理已存在的键时，会带有一个隐式的DEL键的操作。比如rename oldkey newkey时，如果newkey存在需要删除newkey，如果这些目标键是一个big key,那就会引入阻塞删除的性能问题。 此参数设置就是解决这类问题，建议可开启。

## lazyfree的监控
lazy free能监控的数据指标，只有一个值：lazyfree_pending_objects，表示redis执行lazyfree操作，在等待被实际回收内容的键个数。并不能体现单个大键的元素个数或等待lazyfree回收的内存大小。

下面让我们一起看一下惰性删除lazyfree的具体实现

# unlink命令的实现
unlink命令的入口函数是unlinkCommand()
	
	void unlinkCommand(client *c) {
	    delGenericCommand(c,1);
	}

其中就是和del命令一样，调用函数delGenericCommand()进行删除KEY操作，第二个参数为1表示需要异步删除。

delGenericCommand函数根据lazy参数来决定是同步删除还是异步删除，异步删除则调用dbAsyncDelete函数。

	#define LAZYFREE_THRESHOLD 64
	int dbAsyncDelete(redisDb *db, robj *key) {
	    /* Deleting an entry from the expires dict will not free the sds of
	     * the key, because it is shared with the main dictionary. */
	    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);
	
	    /* If the value is composed of a few allocations, to free in a lazy way
	     * is actually just slower... So under a certain limit we just free
	     * the object synchronously. */
	    // 如果key存在，则返回一个hash表中的键值对元素，并且将该元素从hash表中摘除，但没有真正释放键值对的内存
		// 摘除后，后续命令就查询不到
	    dictEntry *de = dictUnlink(db->dict,key->ptr);
	    if (de) {
	        robj *val = dictGetVal(de);
	        size_t free_effort = lazyfreeGetFreeEffort(val);    //获取val对象需要释放的内存块数目
	
	        /* If releasing the object is too much work, do it in the background
	         * by adding the object to the lazy free list.
	         * Note that if the object is shared, to reclaim it now it is not
	         * possible. This rarely happens, however sometimes the implementation
	         * of parts of the Redis core may call incrRefCount() to protect
	         * objects, and then call dbDelete(). In this case we'll fall
	         * through and reach the dictFreeUnlinkedEntry() call, that will be
	         * equivalent to just calling decrRefCount(). */
	        /*
	         * 如果val对象比较大，释放val对象的工作量比较大，则将该对象加入惰性释放队列中等待后台线程释放．
	         * 注意如果是一个共享对象，则不能直接释放其内存．
	         * 这很少发生，但是在部分redis内部实现中，会先调用incrRefCount保护该对象，然后调用dbDelete释放．
	         * 这种情况下，我们不操作，然后到达接下来的dictFreeUnlinkedEntry函数调用，在那里处理时会相当于只是调用一个decrRefCount.
	         */
	        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
	            //原子操作给lazyfree_objects加1，以备info命令查看有多少对象待后台线程删除
	            atomicIncr(lazyfree_objects,1);
	            //此时真正把对象val丢到后台线程的任务队列中
	            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL);
	            //把条目里的val指针设置为NULL，防止删除数据库字典条目时重复删除val对象
	            dictSetVal(db->dict,de,NULL);
	        }
	    }
	
	    /* Release the key-val pair, or just the key if we set the val
	     * field to NULL in order to lazy free it later. */
	    //释放键值对内存资源
	    //如果我们把val设置为了NULL（为了惰性释放），则这里仅仅需要释放key
	    if (de) {
	        dictFreeUnlinkedEntry(db->dict,de);
	        if (server.cluster_enabled) slotToKeyDel(key);
	        return 1;
	    } else {
	        return 0;
	    }
	}

以上便是异步删除的逻辑，首先会清除过期时间，然后调用dictUnlink把要删除的对象从数据库字典摘除，再判断下对象的大小（太小就没必要后台删除），如果足够大（元素个数超过64）就丢给后台线程，最后清理下数据库字典的条目信息。

lazyfreeGetFreeEffort主要是计算释放对象内存需要的时间复杂度，未必就是实际元素个数，和对象的实现相关。

# flushall/flushdb async命令
redis 4.0给flush类命令新加了option——async，当flush类命令后面跟上async选项时，就会进入后台删除逻辑，flush类命令的入口函数flushdbCommand最终会调用emptyDbAsync()来进行整个实例或DB的lazy free逻辑处理。

	/*
	 * 异步清空redis数据库．该函数实际上是为hash表创建一个新的空字典．
	 * 这里直接把db->dict和db->expires指向了新创建的两个空字典，然后把原来两个字典丢到后台线程的任务队列就好了然后把
	 */
	void emptyDbAsync(redisDb *db) {
	    dict *oldht1 = db->dict, *oldht2 = db->expires;
	    db->dict = dictCreate(&dbDictType,NULL);
	    db->expires = dictCreate(&keyptrDictType,NULL);
	    atomicIncr(lazyfree_objects,dictSize(oldht1));
	    bioCreateBackgroundJob(BIO_LAZY_FREE,NULL,oldht1,oldht2);
	}

可以看到，这里直接把db->dict和db->expires指向了新创建的两个空字典，然后把原来两个字典丢到后台线程的任务队列就好了。

当这些命令把对象丢到后台线程后，后台线程执行真正的内存释放操作，下面看一下后台线程异步删除怎么实现。

# 异步删除的实现
## 后台线程的初始化
redis启动的时候，在主函数main中会依次调用initServer --> bioInit函数。在bioInit函数中会初始化三个后台线程，以及相应的锁、条件变量、任务队列等变量，还会设置线程的栈空间大小。三个后台线程分别用于关闭文件描述符、aof持久化冲刷磁盘、惰性删除。

主线程需要将删除任务传递给异步线程，它是通过一个普通的双向链表来传递的。因为链表需要支持多线程并发操作，所以它需要有锁来保护。

## 异步删除任务结构体
执行懒惰删除时，redis将删除操作的相关参数封装成一个bio\_job结构，然后追加到链表尾部。异步线程通过遍历链表摘取job元素来挨个执行异步任务。

	/* This structure represents a background Job. It is only used locally to this
	 * file as the API does not expose the internals at all. */
	struct bio_job {
	    time_t time; /* Time at which the job was created. */
	    /* Job specific arguments pointers. If we need to pass more than three
	     * arguments we can just pass a pointer to a structure or alike. */
	    void *arg1, *arg2, *arg3;
	};

我们注意到这个job结构有三个参数，为什么删除对象需要三个参数呢？在线程函数体bioProcessBackgroundJobs中可以看到：

		if (type == BIO_LAZY_FREE) {
            /* What we free changes depending on what arguments are set:
             * arg1 -> free the object at pointer.
             * arg2 & arg3 -> free two dictionaries (a Redis DB).
             * only arg3 -> free the skiplist. */
            if (job->arg1)
                lazyfreeFreeObjectFromBioThread(job->arg1);     // 释放一个普通对象，string/set/zset/hash等等，用于普通对象的异步删除
            else if (job->arg2 && job->arg3)
                lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);      // 释放全局redisDb对象的dict字典和expires字典，用于flushdb
            else if (job->arg3)
                lazyfreeFreeSlotsMapFromBioThread(job->arg3);       // 释放Cluster的slots_to_keys对象，
        } 

可以看到通过组合这三个参数可以实现不同结构的释放逻辑。

## 异步删除普通对象
普通对象异步删除实现如下：

	/* Release objects from the lazyfree thread. It's just decrRefCount()
	 * updating the count of objects to release. */
	void lazyfreeFreeObjectFromBioThread(robj *o) {
	    decrRefCount(o);
	    atomicDecr(lazyfree_objects,1);
	}
	
	void decrRefCount(robj *o) {
	    if (o->refcount == 1) {
	        switch(o->type) {
	        case OBJ_STRING: freeStringObject(o); break;
	        case OBJ_LIST: freeListObject(o); break;
	        case OBJ_SET: freeSetObject(o); break;
	        case OBJ_ZSET: freeZsetObject(o); break;
	        case OBJ_HASH: freeHashObject(o); break;
	        case OBJ_MODULE: freeModuleObject(o); break;
	        case OBJ_STREAM: freeStreamObject(o); break;
	        default: serverPanic("Unknown object type"); break;
	        }
	        zfree(o);
	    } else {
	        if (o->refcount <= 0) serverPanic("decrRefCount against refcount <= 0");
	        if (o->refcount != OBJ_SHARED_REFCOUNT) o->refcount--;
	    }
	}

可以看到，后台删除对象，调用decrRefCount来减少对象的引用计数，引用计数为0时会真正的释放资源。而在unlink的实现中，我们可以看到，只有当对象的引用计数为1，才会加入后台删除队列。换句话说交给lazyfree线程处理的对象必然是1。

## 数据库字典异步删除
当调用flushdb/flushall async命令，将数据库字典加入异步删除队列后，会调用lazyfreeFreeDatabaseFromBioThread函数进行异步删除。该函数会依次调用dictRelease --> _dictClear 遍历hash table，逐一删除hash表中的元素，释放内存。

## 释放slots_to_keys对象
集群模式下，slots_to_keys用于保存key与插槽的对应关系，方便迁移时导出属于某个插槽中的key。在4.0以前是通过跳跃表skiplist实现，在4.0以后通过基数树实现。该对象的异步删除实现如下：

	void lazyfreeFreeSlotsMapFromBioThread(rax *rt) {
	    size_t len = rt->numele;
	    raxFree(rt);
	    atomicDecr(lazyfree_objects,len);
	}

## 队列安全
前面提到任务队列是一个不安全的双向链表，需要使用锁来保护它。当主线程将任务追加到队列之前它需要加锁，追加完毕后，再释放锁，还需要唤醒异步线程，如果它在休眠的话。

	void bioCreateBackgroundJob(int type, void *arg1, void *arg2, void *arg3) {
	    struct bio_job *job = zmalloc(sizeof(*job));
	
	    job->time = time(NULL);
	    job->arg1 = arg1;
	    job->arg2 = arg2;
	    job->arg3 = arg3;
	    pthread_mutex_lock(&bio_mutex[type]); // 加锁
	    listAddNodeTail(bio_jobs[type],job); // 追加任务
	    bio_pending[type]++; // 计数
	    pthread_cond_signal(&bio_newjob_cond[type]); // 唤醒异步线程
	    pthread_mutex_unlock(&bio_mutex[type]); // 释放锁
	}

异步线程需要对任务队列进行轮训处理，依次从链表表头摘取元素逐个处理。摘取元素的时候也需要加锁，摘出来之后再解锁。如果一个元素的没有，它需要等待，直到主线程来唤醒它继续工作。详见后台线程执行函数体bioProcessBackgroundJobs函数。

# 被动删除键使用lazy free的实现
被动删除4个场景，redis在每个场景调用时，都会判断对应的参数是否开启，如果参数开启，则调用以上对应的lazy free函数处理逻辑实现。

## lazyfree-lazy-eviction

	int freeMemoryIfNeeded(long long timelimit) {
	     ...
	             /* Finally remove the selected key. */
	             if (bestkey) {
	                 ...
	                 propagateExpire(db,keyobj,server.lazyfree_lazy_eviction);
	                 if (server.lazyfree_lazy_eviction)
	                     dbAsyncDelete(db,keyobj);
	                 else
	                     dbSyncDelete(db,keyobj);
	                 ...
	             }
	     ...
	 }         

如果开启了异步驱逐淘汰key选项，则会调用dbAsyncDelete函数根据key大小选择异步删除对象。

## slave-lazy-flush

	void readSyncBulkPayload(aeEventLoop *el, int fd, void *privdata, int mask) {
	     ...
	     if (eof_reached) {
	         ...
	         emptyDb(
	             -1,
	             server.repl_slave_lazy_flush ? EMPTYDB_ASYNC : EMPTYDB_NO_FLAGS,
	             replicationEmptyDbCallback);
	         ...
	     }
	     ...
	 }

从节点在接收完RDB文件后，调用emptyDb清空本地数据库中数据，如果开启了slave-lazy-flush选项，则可以异步删除数据库中的数据，减少全量同步过程的时间，减少主节点输出缓存区的压力。同时在这里传入了一个回调函数replicationEmptyDbCallback，该函数会给主节点发送一个换行符，用于保持到主节点的心跳，详细分析见《 [redis学习笔记]主从节点间连接超时判断 》。

## lazyfree-lazy-expire

	int activeExpireCycleTryExpire(redisDb *db, struct dictEntry *de, long long now) {
	     ...
	     if (now > t) {
	         ...
	         propagateExpire(db,keyobj,server.lazyfree_lazy_expire);
	         if (server.lazyfree_lazy_expire)
	             dbAsyncDelete(db,keyobj);
	         else
	             dbSyncDelete(db,keyobj);
	         ...
	     }
	     ...
	 }

在删除过期键的过程中，会判断是否开启lazyfree-lazy-expire，如果开启则调用dbAsyncDelete函数异步删除key。

## lazyfree-lazy-server-del
	
	int dbDelete(redisDb *db, robj *key) {
	     return server.lazyfree_lazy_server_del ? dbAsyncDelete(db,key) :
	                                              dbSyncDelete(db,key);
	 }

# 参考资料
* [Redis4.0新特性(三)-Lazy Free](https://www.jianshu.com/p/e927e99e650d)
* [如履薄冰 —— Redis懒惰删除的巨大牺牲](https://zhuanlan.zhihu.com/p/41754417)
* [Redis · lazyfree · 大key删除的福音](https://yq.aliyun.com/articles/655899)