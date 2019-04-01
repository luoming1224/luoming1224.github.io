---
title: '[redis学习笔记]主从节点间连接超时判断'
date: 2018-11-10 15:14:20
categories: 'redis学习笔记'
tags: [redis, 主从复制超时]
---
redis主从节点之间会建立一个连接用于复制数据，该连接由从节点收到slave of命令或者cluster replicate（集群模式下）后发起建立到主节点的连接，主从节点均会定时判断该连接是否超时（这个判断过程在replicationCron函数中实现），如果被判断超时，会断开该连接，从节点需要重新建立连接，继续同步数据。

<!--more-->

# 连接建立
在非集群模式下，当节点收到slaveof命令时，依次调用slaveofCommand --> replicationSetMaster 函数

在集群模式下，当节点收到cluster replicate时，会依次调用clusterCommand --> clusterSetMaster --> replicationSetMaster 函数

在replicationSetMaster函数中会设置server.masterhost和server.masterport，且将server.repl\_state设置为REPL\_STATE\_CONNECT;此后并没有直接连接到主节点的连接，而是在replicationCron周期函数中去执行此过程。

replicationCron周期函数每秒钟执行一次，发现server.repl\_state为REPL\_STATE\_CONNECT时，会调用connectWithMaster建立到主节点的连接。

	/* Check if we should connect to a MASTER */
    if (server.repl_state == REPL_STATE_CONNECT) {
        serverLog(LL_NOTICE,"Connecting to MASTER %s:%d",
            server.masterhost, server.masterport);
        if (connectWithMaster() == C_OK) {
            serverLog(LL_NOTICE,"MASTER <-> SLAVE sync started");
        }
    }

	int connectWithMaster(void) {
    	int fd;
	
    	fd = anetTcpNonBlockBestEffortBindConnect(NULL,
     	   server.masterhost,server.masterport,NET_FIRST_BIND_ADDR);
    	if (fd == -1) {
    	    serverLog(LL_WARNING,"Unable to connect to MASTER: %s",
     	       strerror(errno));
     	   return C_ERR;
    	}

   		if (aeCreateFileEvent(server.el,fd,AE_READABLE|	AE_WRITABLE,syncWithMaster,NULL) ==
            AE_ERR)
    	{
        	close(fd);
        	serverLog(LL_WARNING,"Can't create readable event for SYNC");
        	return C_ERR;
    	}

    	server.repl_transfer_lastio = server.unixtime;
    	server.repl_transfer_s = fd;
    	server.repl_state = REPL_STATE_CONNECTING;
    	return C_OK;
	}

从节点建立到主节点的连接后，将socket描述符保存到了server.repl\_transfer\_s，此后从节点都通过该描述符与主节点交互。此外，在redis中，当节点为从节点时，有一个变量server.master用于保存主节点到从节点的连接客户端，我们会发现server.master客户端中保存的socket描述符就是repl\_transfer\_s，因此并不是主节点也建立了一个到从节点的连接，只是从节点为主节点封装了一个客户端。

在全量复制完成后（从节点在readSyncBulkPayload函数中接收并加载完从主节点发送的RDB文件后），会调用replicationCreateMasterClient(server.repl\_transfer\_s,rsi.repl\_stream\_db)函数，会创建一个客户端作为主节点到该从节点的客户端，并保存进server.master。客户端的fd即为server.repl\_transfer\_s。

	void replicationCreateMasterClient(int fd, int dbid) {
	    server.master = createClient(fd);
	    server.master->flags |= CLIENT_MASTER;
	    server.master->authenticated = 1;
	    server.master->reploff = server.master_initial_offset;
	    server.master->read_reploff = server.master->reploff;
	    memcpy(server.master->replid, server.master_replid,
	        sizeof(server.master_replid));
	    /* If master offset is set to -1, this master is old and is not
	     * PSYNC capable, so we flag it accordingly. */
	    if (server.master->reploff == -1)
	        server.master->flags |= CLIENT_PRE_PSYNC;
	    if (dbid != -1) selectDb(server.master,dbid);
	}

在部分重同步中亦是如此，在从节点建立到主节点的连接后，注册了一个该fd读写事件回调函数syncWithMaster，在syncWithMaster函数中调用slaveTryPartialResynchronization处理主节点发送过来的全量同步或部分同步消息，在处理部分同步消息时，调用replicationResurrectCachedMaster函数，在该函数中会将从节点到主节点连接的socket描述符保存进server.master中。

# 从节点判断主节点超时
分为不同的阶段，从节点主要判断repl\_transfer\_lastio和lastinteraction参数，来判断主节点发送数据是否超时。这两个参数均是不同阶段记录上一次主节点发送数据的时间。

## 连接建立阶段
也即在从节点的复制状态（server.repl\_state）为REPL\_STATE\_CONNECTING阶段；

从节点连接到主节点且向主节点发送ip、replconf、psync等命令，从节点判断repl\_transfer\_lastio参数，从节点每次从主节点读取到命令回复数据时都会更新该参数，该更新在sendSynchronousCommand函数、connectWithMaster、syncWithMaster等函数中完成。

	if (server.masterhost &&
        (server.repl_state == REPL_STATE_CONNECTING ||
         slaveIsInHandshakeState()) &&
         (time(NULL)-server.repl_transfer_lastio) > server.repl_timeout)
    {
        serverLog(LL_WARNING,"Timeout connecting to the MASTER...");
        cancelReplicationHandshake();
    }

repl\_transfer\_lastio记录的是主节点上一次发送数据的时间，如果当前距离该时间超过repl\_timeout（默认60秒），则认为主节点发送数据超时，则会调用cancelReplicationHandshake函数取消到主节点的连接，并将复制状态repl\_state重新设置为REP\L_STATE\_CONNECT，等待下一轮调用replicationCron周期函数，重新建立到主节点的连接。

## RDB传输阶段
在从节点的复制状态（server.repl_state）为REPL\_STATE\_TRANSFER阶段；

在等待主节点生成RDB文件和传输RDB文件过程中，从节点依然判断repl\_transfer\_lastio参数。

	/* Bulk transfer I/O timeout? */
    if (server.masterhost && server.repl_state == REPL_STATE_TRANSFER &&
        (time(NULL)-server.repl_transfer_lastio) > server.repl_timeout)
    {
        serverLog(LL_WARNING,"Timeout receiving bulk data from MASTER... If the problem persists try to set the 'repl-timeout' parameter in redis.conf to a larger value.");
        cancelReplicationHandshake();
    }

### 等待主节点生成RDB文件阶段
在生成RDB过程中，主节点会每秒钟给从节点发送一个换行符（'\n'），从节点在readSyncBulkPayload函数中，接收从主节点发送过来的RDB数据，同时在还没有RDB数据过来时，也能接收到这个换行符，然后更新server.repl\_transfer\_lastio，接收RDB数据的过程中也会更新该值，于是可以保持主从之间的心跳连接。

	/*
     * 在从节点等待BGSAVE开始和等待BGSAVE完成时，主节点每秒钟给该从节点发送一个换行符("\n")
     * 从节点会忽略该换行符，但是会更新和主节点最后的交互时间lastinteraction，
     * 保证了不会发生timeout，也不会影响复制偏移量
     */
    listRewind(server.slaves,&li);
    while((ln = listNext(&li))) {
        client *slave = ln->value;

        int is_presync =
            (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_START ||
            (slave->replstate == SLAVE_STATE_WAIT_BGSAVE_END &&
             server.rdb_child_type != RDB_CHILD_TYPE_SOCKET));

        if (is_presync) {
            if (write(slave->fd, "\n", 1) == -1) {
                /* Don't worry about socket errors, it's just a ping. */
            }
        }
    }

SLAVE\_STATE\_WAIT\_BGSAVE\_START状态表示该从节点在等待BGSAVE完成，即主节点处于生产RDB文件过程。

	/* If repl_transfer_size == -1 we still have to read the bulk length
     * from the master reply. */
    if (server.repl_transfer_size == -1) {
        if (syncReadLine(fd,buf,1024,server.repl_syncio_timeout*1000) == -1) {
            serverLog(LL_WARNING,
                "I/O error reading bulk count from MASTER: %s",
                strerror(errno));
            goto error;
        }

        if (buf[0] == '-') {
            serverLog(LL_WARNING,
                "MASTER aborted replication with an error: %s",
                buf+1);
            goto error;
        } else if (buf[0] == '\0') {
            /* At this stage just a newline works as a PING in order to take
             * the connection live. So we refresh our last interaction
             * timestamp. */
            server.repl_transfer_lastio = server.unixtime;
            return;
        } else if (buf[0] != '$') {
            serverLog(LL_WARNING,"Bad protocol from MASTER, the first byte is not '$' (we received '%s'), are you sure the host and port are right?", buf);
            goto error;
        }
		......
	}
	
repl\_transfer\_size为-1表示从节点还在等待主节点发送rdb文件的大小，即RDB文件传输还没有开始，此时会收到主节点发送的换行符（'\n'），更新主节点发送数据时间repl\_transfer\_lastio。

### 从节点清空本地数据阶段
从节点在readSyncBulkPayload函数中接收完RDB文件后，调用emptyDb函数清空本地数据，该过程是一个阻塞过程，所以是不会判断主节点是否超时的。但是从节点在清空数据的过程中，会定期给主节点发送一个换行符（'\n'），用于保持到主节点的心跳。

### 从节点加载RDB文件阶段
在加载RDB文件前，会在rdbLoadRio函数中设置一个回调函数rdbLoadProgressCallback，在加载过程中，会调用该回调函数，在该回调函数中会给主节点发送一个换行符（使得在加载RDB过程中主节点不会认为从节点超时），同时会调用processEventsWhileBlocked函数，加载RDB过程本来是一个阻塞过程，从节点无法响应客户端命令，但是在processEventsWhileBlocked函数中会调用aeProcessEvents(server.el, AE\_FILE\_EVENTS|AE\_DONT\_WAIT)；调用事件循环，处理客户端事件，注意到该函数中没有传入AE\_TIME\_EVENTS，也就是在加载RDB过程中会处理客户端事件，但是不会处理定时任务事件，所以不会进入到replicationCron，也就是说**从节点加载RDB过程中不会判断主节点是否超时**。

	void rdbLoadProgressCallback(rio *r, const void *buf, size_t len) {
	    if (server.rdb_checksum)
	        rioGenericUpdateChecksum(r, buf, len);
	    if (server.loading_process_events_interval_bytes &&
	        (r->processed_bytes + len)/server.loading_process_events_interval_bytes > r->processed_bytes/server.loading_process_events_interval_bytes)
	    {
	        /* The DB can take some non trivial amount of time to load. Update
	         * our cached time since it is used to create and update the last
	         * interaction time with clients and for other important things. */
	        updateCachedTime();
	        if (server.masterhost && server.repl_state == REPL_STATE_TRANSFER)
	            replicationSendNewlineToMaster();
	        loadingProgress(r->processed_bytes);
	        processEventsWhileBlocked();
	    }
	}
	
	int processEventsWhileBlocked(void) {
	    int iterations = 4; /* See the function top-comment. */
	    int count = 0;
	    while (iterations--) {
	        int events = 0;
	        events += aeProcessEvents(server.el, AE_FILE_EVENTS|AE_DONT_WAIT);
	        events += handleClientsWithPendingWrites();
	        if (!events) break;
	        count += events;
	    }
	    return count;
	}

注意在aeProcessEvents中没有传入AE\_TIME\_EVENTS选项，因此不会处理定时事件，只会处理客户端事件。因此也可以看出，**在节点加载RDB文件的过程中，并不是完全阻塞的**。

## 增量同步阶段
也即在从节点的复制状态（server.repl\_state）为REPL\_STATE\_CONNECTED阶段；

从节点在接收并加载完RDB文件后，设置server.repl\_state == REPL\_STATE\_CONNECTED；以后在replicationCron中将判断server.master->lastinteraction决定与主节点是否超时。**注意
lastinteraction是一个client属性。**

	if (server.masterhost && server.repl_state == REPL_STATE_CONNECTED &&
        (time(NULL)-server.master->lastinteraction) > server.repl_timeout)
    {
        serverLog(LL_WARNING,"MASTER timeout: no data nor PING received...");
        freeClient(server.master);
    }

如果超时则会释放客户端，在freeClient中，如果被释放的客户端带有CLIENT\_MASTER标志位，则会调用replicationHandleMasterDisconnection函数将从节点的复制状态设置为REPL\_STATE\_CONNECT，于是等待下一轮执行replicationCron周期函数时，重新建立到主节点的连接。

在整个过程中，主节点每10秒钟会给所有的从节点发送一个ping命令，从节点在接收主节点的增量数据过程中，同时也能收到这个ping命令，于是在读取来自客户端的数据函数中更新该交互时间lastinteraction，在该readQueryFromClient函数中更新。为了保证只有在读取主节点数据时才更新这个时间，在从节点给主节点发送心跳数据时是不更新该数据的，区别于其他的客户端交互时间更新逻辑，见writeToClient函数。

在replicationCron中主节点每10秒钟给所有从节点发送一个ping命令

	/* First, send PING according to ping_slave_period. */
    // 主节点每10秒钟给所有的从节点发一个PING命令 (repl_ping_slave_period默认值是10)
    if ((replication_cron_loops % server.repl_ping_slave_period) == 0 &&
        listLength(server.slaves))
    {
        ping_argv[0] = createStringObject("PING",4);
        replicationFeedSlaves(server.slaves, server.slaveseldb,
            ping_argv, 1);
        decrRefCount(ping_argv[0]);
    }

上面的换行符数据不会影响复制偏移量，仅仅保持心跳作用，而在整个过程中发送的ping命令是会影响复制积压缓冲区的，ping命令是通过replicationFeedSlaves函数发送的，会加入复制积压缓冲区。

# 主节点判断从节点超时
主节点通过在replicationCron函数中判断slave->repl\_ack\_time参数判断从节点是否超时，repl\_ack\_time记录每个从节点上一次给主节点发送数据的时间。

	/* Disconnect timedout slaves. */
    if (listLength(server.slaves)) {
        listIter li;
        listNode *ln;

        listRewind(server.slaves,&li);
        while((ln = listNext(&li))) {
            client *slave = ln->value;

            if (slave->replstate != SLAVE_STATE_ONLINE) continue; //RDB文件传输完毕前不检查从节点的连接状态
            if (slave->flags & CLIENT_PRE_PSYNC) continue;
            if ((server.unixtime - slave->repl_ack_time) > server.repl_timeout)
            {
                serverLog(LL_WARNING, "Disconnecting timedout slave: %s",
                    replicationGetSlaveName(slave));
                freeClient(slave);
            }
        }
    }

## 连接建立阶段
主节点在接收完psync命令以后，才调用listAddNodeTail函数将从节点加入从节点队列中，所以在这个之前不会判断从节点是否超时

## 主节点生成和传输RDB阶段
主节点在传输RDB文件完成后，才会在sendBulkToSlave函数中调用putSlaveOnline函数，设置slave->replstate = SLAVE\_STATE\_ONLINE;将状态改为SLAVE\_STATE\_ONLINE；主节点在replicationCron函数中检查从节点的连接状态时，如果从节点的状态不为SLAVE\_STATE\_ONLINE则不检查，即在RDB文件生成并传输完毕前，是**不会检查**该从节点的状态。

## 从节点加载RDB阶段
从节点在清空本地数据和加载RDB过程中，会定期给主节点发送一个换行符，保持到主节点的心跳，主节点在processInlineBuffer函数中会解析该换行符并忽略，但是会更新从节点最后发送数据的时间repl\_ack\_time，保证了从节点在清空本地数据和加载大的RDB文件期间，主节点不会认为从节点断线。

在processInlineBuffer函数中可以看到如下处理：

	/* Newline from slaves can be used to refresh the last ACK time.
     * This is useful for a slave to ping back while loading a big
     * RDB file. */
    if (querylen == 0 && c->flags & CLIENT_SLAVE)
        c->repl_ack_time = server.unixtime;

### 从节点清空本地数据期间
在readSyncBulkPayload函数中，会调用emptyDb，并传入一个回调函数replicationEmptyDbCallback，该回调函数即向主节点写入一个换行符，在遍历数据库清空数据的过程中会定期调用该回调函数，详见\_dictClear函数；

	/* Callback used by emptyDb() while flushing away old data to load
	 * the new dataset received by the master. */
	void replicationEmptyDbCallback(void *privdata) {
	    UNUSED(privdata);
	    replicationSendNewlineToMaster();
	}
	
	/*
	 * 在全量同步加载RDB文件的过程中，避免master认为slave超时，slave会给master发送一个换行符("\n")
	 * 该函数会在两种情景下调用：
	 * （１）在slave加载RDB文件前，需要先调用emptyDb()清空本地数据，在emptyDb()函数中会调用一个回调函数（即为该函数），
	 * （２）在从节点加载RDB文件的过程中，从节点每次读取RDB文件内容时，会调用校验函数，在校验函数中满足一定条件时会调用该函数给主节点发送一个换行符
	 */
	void replicationSendNewlineToMaster(void) {
	    static time_t newline_sent;
	    if (time(NULL) != newline_sent) {
	        newline_sent = time(NULL);
	        if (write(server.repl_transfer_s,"\n",1) == -1) {
	            /* Pinging back in this stage is best-effort. */
	        }
	    }
	}


### 从节点加载RDB文件期间
在readSyncBulkPayload函数中，清空完本地数据后，接着在调用rdbLoad --> rdbLoadRio加载RDB数据的过程中，会传入一个CRC校验函数，该校验函数中如果节点为从节点，且该从节点的复制状态（server.repl\_state）为REPL\_STATE\_TRANSFER，即还在接收或者加载RDB文件的过程中，则会调用replicationSendNewlineToMaster函数给主节点发送一个换行符。

## 增量数据传输阶段
在正常增量数据传输过程中，在replicationCron函数中，从节点会每秒钟给主节点发送一个replconf ack，带上自己的复制偏移量，主节点在replconfCommand函数中解析ack命令，并更新repl\_ack\_time时间

	/* Send ACK to master from time to time.
     * Note that we do not send periodic acks to masters that don't
     * support PSYNC and replication offsets. */
    // 从节点每秒钟给主节点发送一个ack，并且带上复制的偏移量
    if (server.masterhost && server.master &&
        !(server.master->flags & CLIENT_PRE_PSYNC))
        replicationSendAck();

# 总结
* 在数据复制阶段，主节点每10秒钟给所有从节点发送一个ping消息，从节点每秒钟给主节点发送一个replconf ack消息，且附带上已从主节点复制的偏移量。
* 主节点在生成RDB过程中会每秒钟给从节点发送一个换行符（'\n'）；从节点在清空本地数据和加载RDB过程中，会定期给主节点发送一个换行符（'\n'），保持到主节点的心跳。
* 从节点在清空和加载RDB过程中，不检查主节点是否超时；在RDB文件传输完毕前，主节点不检查从节点是否超时。
* 从节点在加载RDB文件的过程中，会定期处理客户端事件，但是不会处理定时时间事件。