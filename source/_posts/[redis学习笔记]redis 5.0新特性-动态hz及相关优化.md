---
title: '[redis学习笔记]redis 5.0新特性--动态hz及相关优化'
date: 2018-11-27 15:45:37
categories: 'redis学习笔记'
tags: [redis, dynamic-hz, info, clientsCron]
---
redis5.0增加了一个dynamic-hz参数，用于自适应调整server.hz的值，平衡空闲CPU的使用率和响应能力；另外在redis5.0中对info命令、clientsCron函数、大量的短连接情形进行了优化。

<!--more-->

# 动态hz参数
## 无法解释的延迟
在redis5.0之前，server.hz参数的值默认为10，即serverCron函数的执行周期为100ms，在serverCron函数中会调用clientsCron函数，处理一些连接客户端相关的事情，比如处理客户端超时、客户端查询缓冲区等。clientsCron函数如下所示：

	#define CLIENTS_CRON_MIN_ITERATIONS 5
	void clientsCron(void) {
	    /* Try to process at least numclients/server.hz of clients
	     * per call. Since normally (if there are no big latency events) this
	     * function is called server.hz times per second, in the average case we
	     * process all the clients in 1 second. */
	    int numclients = listLength(server.clients);
	    int iterations = numclients/server.hz;
	    mstime_t now = mstime();
	
	    /* Process at least a few clients while we are at it, even if we need
	     * to process less than CLIENTS_CRON_MIN_ITERATIONS to meet our contract
	     * of processing each client once per second. */
	    if (iterations < CLIENTS_CRON_MIN_ITERATIONS)
	        iterations = (numclients < CLIENTS_CRON_MIN_ITERATIONS) ?
	                     numclients : CLIENTS_CRON_MIN_ITERATIONS;
	
	    while(listLength(server.clients) && iterations--) {
	        client *c;
	        listNode *head;
	
	        /* Rotate the list, take the current head, process.
	         * This way if the client must be removed from the list it's the
	         * first element and we don't incur into O(N) computation. */
	        listRotate(server.clients);
	        head = listFirst(server.clients);
	        c = listNodeValue(head);
	        /* The following functions do different service checks on the client.
	         * The protocol is that they return non-zero if the client was
	         * terminated. */
	        if (clientsCronHandleTimeout(c,now)) continue;
	        if (clientsCronResizeQueryBuffer(c)) continue;
	        if (clientsCronTrackExpansiveClients(c)) continue;
	    }
	}

在上面的代码中，我们可以看到，为了在每秒钟内处理所有客户端连接一次，每次调用必须处理numclients/server.hz个客户端，因为clientsCron函数每秒钟调用server.hz次。当客户端数量非常多的时候，该部分的耗时将会非常多，比如1W个客户端连接，在默认值10的情况下，每次需要处理1000个客户端。

而且由于此处属于redis内部逻辑，并且没有延迟监控，当此处逻辑耗时比较大时，我们无法在慢查询日志或者延迟监控中发现，会使得生产上产生一些无法解释的延时现象。

## 动态hz实现
于是在redis5.0中增加了dynamic-hz参数，默认开启动态hz，使得在客户端连接非常多时，自适应调整hz参数，临时增加hz参数，使得每秒钟执行serverCron更多次，占用更多的CPU，每次可以处理一定数量的客户端连接，不至于产生严重超时现象。

在serverCron函数中，每次检查客户端数量，设置相应的hz值。

	#define CONFIG_MAX_HZ            500
	#define MAX_CLIENTS_PER_CLOCK_TICK 200          /* HZ is adapted based on that. */
	
	server.hz = server.config_hz;   //默认值为10
    /* Adapt the server.hz value to the number of configured clients. If we have
     * many clients, we want to call serverCron() with an higher frequency. */
    if (server.dynamic_hz) {
        while (listLength(server.clients) / server.hz >
               MAX_CLIENTS_PER_CLOCK_TICK)
        {
            server.hz *= 2;
            if (server.hz > CONFIG_MAX_HZ) {
                server.hz = CONFIG_MAX_HZ;
                break;
            }
        }
    }

可以看出，每次循环最多处理200个客户端连接，hz的值最高不超过500（事实上，自己设置的话不建议超过100），默认hz值10将作为一个基线，每次循环都将hz设为配置的hz值，然后，如果客户端数量非常多，就自适应调整hz的值；下一次循环中，如果客户端数量变少了，hz值依然会是默认值了。

# clientsCron优化
## 4.0中实现
上面说了，在clientsCron中一次处理过多客户端连接，可能会引起延时，在redis5.0以前版本中，clientsCron函数中调用的clientsCronResizeQueryBuffer函数存在一个bug，使得该问题得到加剧和放大。

redis5.0以前的clientsCronResizeQueryBuffer函数实现为：

	int clientsCronResizeQueryBuffer(client *c) {
	    size_t querybuf_size = sdsAllocSize(c->querybuf);
	    time_t idletime = server.unixtime - c->lastinteraction;
	
	    /* There are two conditions to resize the query buffer:
	     * 1) Query buffer is > BIG_ARG and too big for latest peak.
	     * 2) Client is inactive and the buffer is bigger than 1k. */
	    if (((querybuf_size > PROTO_MBULK_BIG_ARG) &&
	         (querybuf_size/(c->querybuf_peak+1)) > 2) ||
	         (querybuf_size > 1024 && idletime > 2))
	    {
	        /* Only resize the query buffer if it is actually wasting space. */
	        if (sdsavail(c->querybuf) > 1024) {
	            c->querybuf = sdsRemoveFreeSpace(c->querybuf);
	        }
	    }
	    /* Reset the peak again to capture the peak memory usage in the next
	     * cycle. */
	    c->querybuf_peak = 0;
	    return 0;
	}

满足以下两个条件之一时：

（1） 查询缓冲区大于32K，且远大于查询缓冲区数据峰值

（2） 查询缓冲区大于1K，且客户端当前处于非活跃状态

同时查询缓冲区空闲空间大于1K，就回收空闲空间。

而输入缓冲区在第一次分配时就为32K，所以显然大于1K，所以只要不够活跃很容易满足回收条件

输入缓冲区空间分配在readQueryFromClient函数中，
	
	#define PROTO_IOBUF_LEN         (1024*16)  /* Generic I/O buffer size */
	
	readlen = PROTO_IOBUF_LEN;
	
	c->querybuf = sdsMakeRoomFor(c->querybuf, readlen);

在sdsMakeRoomFor函数中分配时，如果分配的空间小于1M，会乘以2，所以首次就分配了32K空间。

## 5.0中实现
在redis5.0中clientsCronResizeQueryBuffer函数实现如下：

	int clientsCronResizeQueryBuffer(client *c) {
	    size_t querybuf_size = sdsAllocSize(c->querybuf);
	    time_t idletime = server.unixtime - c->lastinteraction;
	
	    /* There are two conditions to resize the query buffer:
	     * 1) Query buffer is > BIG_ARG and too big for latest peak.
	     * 2) Query buffer is > BIG_ARG and client is idle. */
	    if (querybuf_size > PROTO_MBULK_BIG_ARG &&
	         ((querybuf_size/(c->querybuf_peak+1)) > 2 ||
	          idletime > 2))
	    {
	        /* Only resize the query buffer if it is actually wasting
	         * at least a few kbytes. */
	        if (sdsavail(c->querybuf) > 1024*4) {
	            c->querybuf = sdsRemoveFreeSpace(c->querybuf);
	        }
	    }
	    /* Reset the peak again to capture the peak memory usage in the next
	     * cycle. */
	    c->querybuf_peak = 0;
		//...
	}

更改为满足以下两个条件之一：

（1） 查询缓冲区大于32K，且远大于查询缓冲区数据峰值

（2） 查询缓冲区大于32K，且客户端当前处于非活跃状态

同时查询缓冲区空闲空间大于4K，就回收空闲空间。

如果客户端数量比较多，且刚好比较空闲，在5.0以前，很容易因为需要一次处理很多客户端的输入缓冲区，导致节点延迟甚至崩溃。

## 案例分享
下面链接两个相关案例：

* [记一个真实的排障案例：携程Redis偶发连接失败案例分析](http://www.cnblogs.com/CtripDBA/p/9776466.html?utm_source=tuicool&utm_medium=referral)
* [seconds hang up report ](https://github.com/antirez/redis/issues/4983)

# info命令优化
## info命令慢日志
按说info命令只是返回一些基本统计数据，不应该有慢日志，但是在redis5.0以前，当客户端连接非常大的时候，可能会出现info慢日志。

原因是，在redis5.0以前的info命令实现中，每次会获取所有客户端中使用的最大输入/输出缓冲区大小，因此需要循环遍历每个客户端连接，当客户端数据非常大，比如上万连接时，将操作将会非常耗时。

info命令实现依次调用genRedisInfoString --> getClientsMaxBuffers，代码就不列出了，可以自行查看源码，并且是在genRedisInfoString函数中一开始就调用，所以对所有info子命令都有影响。

## info命令优化
在redis5.0中，对该问题进行了优化，不再每次获取所有客户端连接的最大输入/输出缓冲区大小，而是在clientsCron函数中用一个数组记录客户端连接最近几秒时间的输入/输出缓冲区最大值，在info clients子命令中调用getExpansiveClientsInfo函数，获取这个数组中保存的最近几秒时间内的最大值，这样不管客户端连接数量多达，获取时间都是恒定的，且耗时非常小。

在clientsCron函数中调用clientsCronTrackExpansiveClients追踪所有客户端连接中最近几秒的输入/输出缓冲区最大值
	
	int clientsCronTrackExpansiveClients(client *c) {
	    size_t in_usage = sdsAllocSize(c->querybuf);
	    size_t out_usage = getClientOutputBufferMemoryUsage(c);
	    int i = server.unixtime % CLIENTS_PEAK_MEM_USAGE_SLOTS;
	    int zeroidx = (i+1) % CLIENTS_PEAK_MEM_USAGE_SLOTS;
	
	    /* Always zero the next sample, so that when we switch to that second, we'll
	     * only register samples that are greater in that second without considering
	     * the history of such slot.
	     *
	     * Note: our index may jump to any random position if serverCron() is not
	     * called for some reason with the normal frequency, for instance because
	     * some slow command is called taking multiple seconds to execute. In that
	     * case our array may end containing data which is potentially older
	     * than CLIENTS_PEAK_MEM_USAGE_SLOTS seconds: however this is not a problem
	     * since here we want just to track if "recently" there were very expansive
	     * clients from the POV of memory usage. */
	    ClientsPeakMemInput[zeroidx] = 0;
	    ClientsPeakMemOutput[zeroidx] = 0;
	
	    /* Track the biggest values observed so far in this slot. */
	    if (in_usage > ClientsPeakMemInput[i]) ClientsPeakMemInput[i] = in_usage;
	    if (out_usage > ClientsPeakMemOutput[i]) ClientsPeakMemOutput[i] = out_usage;
	
	    return 0; /* This function never terminates the client. */
	}

在info命令的实现中依次调用genRedisInfoString --> getExpansiveClientsInfo

	/* Return the max samples in the memory usage of clients tracked by
	 * the function clientsCronTrackExpansiveClients(). */
	void getExpansiveClientsInfo(size_t *in_usage, size_t *out_usage) {
	    size_t i = 0, o = 0;
	    for (int j = 0; j < CLIENTS_PEAK_MEM_USAGE_SLOTS; j++) {
	        if (ClientsPeakMemInput[j] > i) i = ClientsPeakMemInput[j];
	        if (ClientsPeakMemOutput[j] > o) o = ClientsPeakMemOutput[j];
	    }
	    *in_usage = i;
	    *out_usage = o;
	}

## 参考资料

* [move getClientsMaxBuffers func into info clients command](https://github.com/antirez/redis/pull/4727)
* [Make INFO faster again!11one](https://github.com/antirez/redis/issues/5145)

# 短连接优化
## 5.0以前实现
Redis在释放客户端连接的时候，会依次调用freeClient --> unlinkClient --> listSearchKey,可以看到在listSearchKey中，redis遍历双端列表server.clients查找到对应的redisClient对象然后调用listDelNode把该redisClient对象从server.clients删除,

	if (c->fd != -1) {
        /* Remove from the list of active clients. */
        ln = listSearchKey(server.clients,c);
        serverAssert(ln != NULL);
        listDelNode(server.clients,ln);

        /* Unregister async I/O handlers and close the socket. */
        aeDeleteFileEvent(server.el,c->fd,AE_READABLE);
        aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);
        close(c->fd);
        c->fd = -1;
    }

listSearchKey遍历列表为O(n)时间复杂度，当大量短连接操作redis时，频繁的释放客户端会引起redis的CPU使用率显著上升。

## 5.0优化
在redis5.0中为了解决此问题，对此操作进行了优化。在createClient的时候将redisClient的指针地址保留，在freeClient的时候直接删除对应的listNode即可，无需再次遍历server.clients。

**在createClient函数中**，最后会调用linkClient函数
	
	if (fd != -1) linkClient(c);
	
	//linkClient实现如下：
	/* This function links the client to the global linked list of clients.
	 * unlinkClient() does the opposite, among other things. */
	void linkClient(client *c) {
	    listAddNodeTail(server.clients,c);
	    /* Note that we remember the linked list node where the client is stored,
	     * this way removing the client in unlinkClient() will not require
	     * a linear scan, but just a constant time operation. */
	    c->client_list_node = listLast(server.clients);
	    uint64_t id = htonu64(c->id);
	    raxInsert(server.clients_index,(unsigned char*)&id,sizeof(id),c,NULL);
	}

可以看到，在将新建的client插入server.clients列表时，将封装该client的节点指针保存进了c->client_list_node；同时以client的唯一索引作为索引，将client插入基数树中。

**在释放客户端时**，仍然是依次调用freeClient --> unlinkClient，在unlinkClient中此时只需要根据保存的节点指针直接去列表中把节点删除，无需遍历列表，时间复杂度直接降为O(1)。

	if (c->fd != -1) {
        /* Remove from the list of active clients. */
        if (c->client_list_node) {
            uint64_t id = htonu64(c->id);
            raxRemove(server.clients_index,(unsigned char*)&id,sizeof(id),NULL);
            listDelNode(server.clients,c->client_list_node);
            c->client_list_node = NULL;
        }

        /* Unregister async I/O handlers and close the socket. */
        aeDeleteFileEvent(server.el,c->fd,AE_READABLE);
        aeDeleteFileEvent(server.el,c->fd,AE_WRITABLE);
        close(c->fd);
        c->fd = -1;
    }

该函数中根据client唯一id，将客户端从基数树中删除，根据节点指针将客户端直接从server.clients列表中删除。

基数树用于client unblock命令操作。

## 参考资料
* [Redis高级实践之————Redis短连接性能优化](https://www.cnblogs.com/tinywan/p/6080293.html)