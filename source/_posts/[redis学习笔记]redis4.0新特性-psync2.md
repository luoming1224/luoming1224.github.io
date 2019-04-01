---
title: '[redis学习笔记]redis新特性--psync2'
date: 2018-11-20 20:35:41
categories: 'redis学习笔记'
tags: [redis, psync, 部分同步]
---
在 2.8 版本之前 redis 没有增量同步的功能，主从只要重连就必须全量同步数据。如果实例数据量比较大的情况下，网络轻轻一抖就会把主从的网卡跑满从而影响正常服务。2.8 为了解决这个问题引入了 psync (partial sync)功能，顾名思义就是增量同步。

在redis4.0版本中，作者对psync进行了优化，更改为psync2，主要解决Redis运维管理过程中，从实例重启和主实例故障切换等场景带来的全量重新同步(full resync)问题。

<!--more-->

# 2.8版本psync1机制
## psync1解决了什么问题
在psync1功能出现前，redis复制秒级中断，就会触发从实例进行fullsync。

每一次的fullsync，集群的性能和资源使用都可能带来抖动；如果redis所处的网络环境不稳定，那么fullsync的出步频率可能较高。为解决此问题，redis2.8引入psync1, 有效地解决这种复制闪断，带来的影响。redis的fullsync对业务而言，算是比较“重”的影响；对性能和可用性都有一定危险。

这里列举几个fullsync常见的影响：

* master需运行bgsave,出现fork()，可能造成master达到毫秒或秒级的卡顿(latest_fork_usec状态监控)；    
* redis进程fork导致Copy-On-Write内存使用消耗(后文简称COW)，最大能导致master进程内存使用量的消耗。(eg 日志中输出 RDB: 5213 MB of memory used by copy-on-write)   
* redis slave load RDB过程，会导致复制线程的client output buffer增长很大；增大Master进程内存消耗；    
* redis保存RDB(不考虑disless replication),导致服务器磁盘IO和CPU(压缩)资源消耗    
* 发送数GB的RDB文件,会导致服务器网络出口爆增,如果千兆网卡服务器，期间会影响业务正常请求响应时间(以及其他连锁影响)

由于在网络闪断等情况下，从节点依然有闪断前同步的完整数据，所以psync1允许在满足条件的时候，只同步闪断期间主节点新增的数据，而不需要重新全量同步所有数据。

## psync1基本实现
redis2.8为支持psync1，引入了replication backlog buffer(后文称：复制积压缓冲区）；复制积压缓冲区是redis维护的固定长度环形缓冲队列(由参数repl-backlog-size设置，默认1MB)，master的写入命令在同步给slaves的同时，会在缓冲区中写入一份**(master只有1个积压缓冲区，所有slaves共享）**。

当redis复制中断后，slave会尝试采用psync, 上报原master runid + 当前已同步master的offset(复制偏移量，类似mysql的binlog file和position)；

如果runid与master的一致，且复制偏移量在master的复制积压缓冲区中还有(即offset >= min(backlog值)，master就认为部分重同步成功，不再进行全量同步。

* 从库尝试发送 psync 命令到主库，而不是直接使用 sync 命令进行全量同步
* 主库判断是否满足 psync 条件, 满足就返回 +CONTINUE 进行增量同步, 否则返回 +FULLRESYNC runid offfset

## psync1源码实现
### 从节点发起同步
从节点在闪断的情况下不再直接发起全量同步，而是上报原master runid和复制偏移量offset，尝试进行部分同步。从节点发起同步的过程位于slaveTryPartialResynchronization函数中

		if (server.cached_master) {
            psync_runid = server.cached_master->replrunid;
            snprintf(psync_offset,sizeof(psync_offset),"%lld", server.cached_master->reploff+1);
            serverLog(LL_NOTICE,"Trying a partial resynchronization (request %s:%s).", psync_runid, psync_offset);
        } else {
            serverLog(LL_NOTICE,"Partial resynchronization not possible (no cached master)");
            psync_runid = "?";
            memcpy(psync_offset,"-1",3);
        }

        /* Issue the PSYNC command */
        reply = sendSynchronousCommand(SYNC_CMD_WRITE,fd,"PSYNC",psync_runid,psync_offset,NULL);

从节点发起部分复制时，在slaveTryPartialResynchronization函数中，如果server.cached_master不为空，则将数据发给主节点尝试部分同步复制，否则直接发送psync ? -1发起全量复制。

### 主节点判断全量同步or部分同步
当收到从节点发起的同步请求后，主节点判断需要全量同步还是部分同步，实现过程在masterTryPartialResynchronization函数中实现。

redis 判断是否允许部分同步有两个条件:

* 条件一: psync 命令携带的 runid 需要和主库的 runid 一致才可以进行增量同步，否则需要全量同步。
> NOTE: 主库的 runid 是在主库进程启动之后生成的唯一标识(由进程id加上随机数组成), 在第一次全量同步的时候发送给从库，上面有看到 FULLSYNC 返回带有 runid 和 offset, 从库会在内存缓存这个 runid 和 offset 信息

	if (strcasecmp(master_runid, server.runid)) {
        /* Run id "?" is used by slaves that want to force a full resync. */
        if (master_runid[0] != '?') {
            serverLog(LL_NOTICE,"Partial resynchronization not accepted: "
                "Runid mismatch (Client asked for runid '%s', my runid is '%s')",
                master_runid, server.runid);
        } else {
            serverLog(LL_NOTICE,"Full resync requested by slave %s",
                replicationGetSlaveName(c));
        }
        goto need_full_resync;
    }

如果runid不相等，则跳转到返回全量同步

* 条件二: psync 命令携带的 offset 是否超过缓冲区。如果超过则需要全量同步，否则就进行增量同步。
> NOTE: backlog 是一个固定大小(默认1M)的环形缓冲区，用来缓存主从同步的数据。如果 offset 超过这个范围说明中间有一段数据已经丢失，需要全量同步。

	if (!server.repl_backlog ||
        psync_offset < server.repl_backlog_off ||
        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
    {
        serverLog(LL_NOTICE,
            "Unable to partial resync with slave %s for lack of backlog (Slave request was: %lld).", replicationGetSlaveName(c), psync_offset);
        if (psync_offset > server.master_repl_offset) {
            serverLog(LL_WARNING,
                "Warning: slave %s tried to PSYNC with an offset that is greater than the master replication offset.", replicationGetSlaveName(c));
        }
        goto need_full_resync;
    }

有了 psync 之后主从短时间断掉重连就可以不用全量同步数据。前提也是这段时间的写入不能超过缓冲区。如果写入量比较大的，也建议稍微调大这个缓冲区。

### 从节点断线保存复制信息
断线前，从节点在freeClient中调用replicationCacheMaster，将当前的数据保存至server.cached_master，以便可以执行psync

freeClient相关代码如下：

	if (server.master && c->flags & CLIENT_MASTER) {
        serverLog(LL_WARNING,"Connection with master lost.");
        if (!(c->flags & (CLIENT_CLOSE_AFTER_REPLY|
                          CLIENT_CLOSE_ASAP|
                          CLIENT_BLOCKED|
                          CLIENT_UNBLOCKED)))
        {
            replicationCacheMaster(c);
            return;
        }
    }

## psync1的不足
虽然 2.8 引入的 psync 可以解决短时间主从同步断掉重连问题，但以下几个场景仍然是需要全量同步:

1. 主库/从库有重启过。因为 runnid 重启后就会丢失，所以当前机制无法做增量同步。
2. 从库提升为主库。其他从库切到新主库全部要全量不同数据，因为新主库的 runnid 跟老的主库是不一样的。

这两个应该是我们比较常见的场景。主库切换或者重启都需要全量同步数据在从库实例比较大或者多的场景下，那内网网络带宽和服务都会有很大的影响。所以 redis 4.0 对 psync 优化之后可以一定程度上规避这些问题。

# 4.0版本psync2机制
为了解决主从角色切换导致的重新全量同步，redis 4.0 引入多另外一个变量 replid2 来存放同步过的主库的 replid，同时 replid 在不同角色意义也有写变化。replid 在主库的意义和之前 replid 仍然是一样的，但对于从库来说，replid 表示当前正在同步的主库的 replid 而不再是本身的 replid。replid2 则表示前一个主库的 replid，这个在主从角色切换的时候会用到，默认初始化为全0。

## 从节点发起部分同步
从节点发起部分同步的过程依然是在slaveTryPartialResynchronization函数中实现，代码和前面是一样的。

所以在4.0中主要是保存和恢复server.cached\_master时机不同，在断线、重启、主从切换时都会对应的保存server.cached\_master，以实现部分同步。

## 主库判断psync
主节点在masterTryPartialResynchronization函数中判断，在主库判断是否允许 psync 的判断条件也有了一些变化：

	/*
     * (1)master_replid和server.replid不相等并且和server.replid2也不相等，需要执行全量复制
     *    即既不是重连或者重启，也不是failover情形下，也需要全量复制
     *    master_replid和server.replid相等应用于从节点断线重连或者重启
     *    master_replid和server.replid2相等，应用于从节点提升为主节点情形
     * (2)master_replid和server.replid不相等，虽然和server.replid2相等，但是psync_offset > server.second_replid_offset，也需要执行全量复制
     * 　　这种情形即为从节点提升为主节点时，虽然和server.replid2相等，但是从节点的复制偏移量比新主节点大，也无法执行部分复制
     * 参考 https://zhuanlan.zhihu.com/p/44105707
     */
    if (strcasecmp(master_replid, server.replid) &&
        (strcasecmp(master_replid, server.replid2) ||
         psync_offset > server.second_replid_offset))
    {
        /* Run id "?" is used by slaves that want to force a full resync. */
        if (master_replid[0] != '?') {
            if (strcasecmp(master_replid, server.replid) &&
                strcasecmp(master_replid, server.replid2))
            {
                serverLog(LL_NOTICE,"Partial resynchronization not accepted: "
                    "Replication ID mismatch (Slave asked for '%s', my "
                    "replication IDs are '%s' and '%s')",
                    master_replid, server.replid, server.replid2);
            } else {
                //从节点同步进度比failover后新的主节点还快
                serverLog(LL_NOTICE,"Partial resynchronization not accepted: "
                    "Requested offset for second ID was %lld, but I can reply "
                    "up to %lld", psync_offset, server.second_replid_offset);
            }
        } else {
            serverLog(LL_NOTICE,"Full resync requested by slave %s",
                replicationGetSlaveName(c));
        }
        goto need_full_resync;
    }

    /* We still have the data our slave is asking for? */
    //判断主节点是否没有复制积压缓冲区或者同步进度是否超过复制积压缓冲区范围
    if (!server.repl_backlog ||
        psync_offset < server.repl_backlog_off ||
        psync_offset > (server.repl_backlog_off + server.repl_backlog_histlen))
    {
        serverLog(LL_NOTICE,
            "Unable to partial resync with slave %s for lack of backlog (Slave request was: %lld).", replicationGetSlaveName(c), psync_offset);
        if (psync_offset > server.master_repl_offset) {
            serverLog(LL_WARNING,
                "Warning: slave %s tried to PSYNC with an offset that is greater than the master replication offset.", replicationGetSlaveName(c));
        }
        goto need_full_resync;
    }

从代码可以看到，主库判断条件相比之前版本多了一个 replid2 的判断，用于发生主从切换的时候。如果之前这两个曾经属于同一个主库(多级也允许)， 那么新主库的 replid2 就是之前主库的 replid。只要之前是同一主库且新主库的同步进度比这个从库还快就允许增量同步。当然前提也是新主从的写入落后不能超过 backlog 大小。

下面看看前面提到的server.cached\_master在重启、主从切换时保存的过程。也就是重启、主从切换时怎样实现来保证接下来可以尝试部分同步。

## redis重启的的部分同步
redis4.0为实现重启后，仍可进行部分重新同步，主要做以下3点：

* redis关闭时，把复制信息作为辅助字段(AUX Fields)存储在RDB文件中；以实现同步信息持久化；
* redis启动加载RDB文件时，会把复制信息赋给相关字段；
* redis重新同步时，会上报repl-id和repl-offset同步信息，如果和主实例匹配，且offset还在主实例的复制积压缓冲区内，则只进行部分重新同步。

### redis关闭时，持久化复制信息到RDB
关闭前在rdbSaveInfoAuxFields函数中会将replid和repl\_offset保存进rdb文件。

redis在关闭前，依次调用prepareForShutdown-->rdbSave-->rdbSaveRio-->rdbSaveInfoAuxFields.

		if (rdbSaveAuxFieldStrStr(rdb,"repl-id",server.replid)
            == -1) return -1;
        if (rdbSaveAuxFieldStrInt(rdb,"repl-offset",server.master_repl_offset)
            == -1) return -1;

### redis启动读取RDB中复制信息
重启时，也是调用replicationCacheMasterUsingMyself函数，会把加载进来的数据保存至server.cached\_master以便可以执行psync

从节点重启时会依次调用loadDataFromDisk-->replicationCacheMasterUsingMyself

在loadDataFromDisk函数中会把RDB文件中的replid和repl\_offset加载进redis中相应的变量。

		memcpy(server.replid,rsi.repl_id,sizeof(server.replid));
        server.master_repl_offset = rsi.repl_offset;

然后在replicationCacheMasterUsingMyself函数中，将这些变量的值复制到server.cached\_master中。

	void replicationCacheMasterUsingMyself(void) {
	    /* The master client we create can be set to any DBID, because
	     * the new master will start its replication stream with SELECT. */
	    server.master_initial_offset = server.master_repl_offset;
	    replicationCreateMasterClient(-1,-1);
	
	    /* Use our own ID / offset. */
	    memcpy(server.master->replid, server.replid, sizeof(server.replid));
	
	    /* Set as cached master. */
	    unlinkClient(server.master);
	    server.cached_master = server.master;
	    server.master = NULL;
	    serverLog(LL_NOTICE,"Before turning into a slave, using my master parameters to synthesize a cached master: I may be able to synchronize with the new master with just a partial transfer.");
	}

## Failover部分同步
为解决主实例故障切换后，重新同步新主实例数据时使用psync，而非fullsync；

1. redis4.0使用两组replid、offset替换原来的master runid和offset.

2. redis **slave默认开启复制积压缓冲区功能**；以便slave故障切换变化master后，其他落后从可以从缓冲区中获取写入指令。

### 新主节点
在failover最后阶段，获得多数投票的从节点将提升为新的主节点，从节点将自己提升为新的主节点时，将会依次调用clusterFailoverReplaceYourMaster --> replicationUnsetMaster --> shiftReplicationId

	/* Use the current replication ID / offset as secondary replication
	 * ID, and change the current one in order to start a new history.
	 * This should be used when an instance is switched from slave to master
	 * so that it can serve PSYNC requests performed using the master
	 * replication ID. */
	void shiftReplicationId(void) {
	    memcpy(server.replid2,server.replid,sizeof(server.replid));
	    /* We set the second replid offset to the master offset + 1, since
	     * the slave will ask for the first byte it has not yet received, so
	     * we need to add one to the offset: for example if, as a slave, we are
	     * sure we have the same history as the master for 50 bytes, after we
	     * are turned into a master, we can accept a PSYNC request with offset
	     * 51, since the slave asking has the same history up to the 50th
	     * byte, and is asking for the new bytes starting at offset 51. */
	    server.second_replid_offset = server.master_repl_offset+1;
	    changeReplicationId();
	    serverLog(LL_WARNING,"Setting secondary replication ID to %s, valid up to offset: %lld. New replication ID is %s", server.replid2, server.second_replid_offset, server.replid);
	}

可以看到，在该函数中，从节点会将自身的server.replid保存进server.replid2，server.master\_repl\_offset+1保存进server.second\_replid\_offset，然后调用changeReplicationId改变其server.replid，这样从节点在提升为主节点之前，将自身之前复制的主节点ID保存进了server.replid2。

因此，在该从节点成为新主节点后，如果之前的原主节点变成从节点发起部分同步，或者之前的兄弟节点发起部分同步，因为同步过同一主节点，于是可以完成部分同步。

### 原主节点及兄弟节点
对于**主节点**，如果之前自己负责的槽位有新的节点声明由其负责，且此时主节点不再负责任何槽位，说明发生了failover，自己由主节点变成了从节点；

对于**从节点**，会发现自己的主节点所负责的槽位有新的节点声明由其负责，说明主节点发生了改变。

此时**均需要重新设置自己的主节点**，让自己复制新的主节点，成为新主节点的从节点。

这个判断在clusterUpdateSlotsConfigWith函数中实现，该函数中会执行clusterSetMaster函数，设置新的主节点。然后执行以下调用关系：clusterUpdateSlotsConfigWith --> clusterSetMaster --> replicationSetMaster --> replicationCacheMasterUsingMyself

replicationSetMaster代码如下：

	/* Set replication to the specified master address and port. */
	void replicationSetMaster(char *ip, int port) {
	    int was_master = server.masterhost == NULL;
	
	    sdsfree(server.masterhost);
	    server.masterhost = sdsnew(ip);
	    server.masterport = port;
	    //释放原来主节点建立到本从节点的连接
	    if (server.master) {
	        freeClient(server.master);
	    }
	    //解除所有客户端的阻塞状态
	    disconnectAllBlockedClients(); /* Clients blocked in master, now slave. */
	
	    /* Force our slaves to resync with us as well. They may hopefully be able
	     * to partially resync with us, but we can notify the replid change. */
	    // 关闭所有从节点服务器的连接，强制从节点服务器进行重新同步操作
	    disconnectSlaves();
	    cancelReplicationHandshake();
	    /* Before destroying our master state, create a cached master using
	     * our own parameters, to later PSYNC with the new master. */
	    if (was_master) replicationCacheMasterUsingMyself();
	    server.repl_state = REPL_STATE_CONNECT;
	}

在replicationSetMaster函数中会断开原来主节点的连接，设置server.repl\_state = REPL\_STATE\_CONNECT，于是在replicationCron函数中会重新建立到新的主节点的连接。

**主节点：**在replicationSetMaster函数中，判断如果当前节点之前为主节点的话，则调用
replicationCacheMasterUsingMyself函数，在该函数中会将当前作为主节点时的server.master\_repl\_offset和server.replid数据保存进server.cached\_master，于是在连接上新的主节点后，可以尝试发起部分复制。

**其他兄弟从节点：**因为满足server.master不为NULL条件，所以会调用freeClient函数释放原来主节点到该从节点的连接，如同节点断线时，调用freeCient函数一样，在该函数中会调用replicationCacheMaster函数将当前复制的主节点replid和repl\_offset保存进server.cached\_master，以便可以发起部分同步。

被failover的主节点和兄弟从节点因此可以在连上新的主节点后，发送自己之前同步的主节点replid和repl\_offset，新的主节点会判断发现之前同步过同一主节点，于是可以正常部分同步。

在这些节点收到新的主节点发送的+CONTINUE后，发现新的主节点replid和自己缓存的主节点replid不一致，则更新正在同步的主节点replid

			if (strcmp(new,server.cached_master->replid)) {
                /* Master ID changed. */
                serverLog(LL_WARNING,"Master replication ID changed to %s",new);
	
                /* Set the old ID as our ID2, up to the current offset+1. */
                memcpy(server.replid2,server.cached_master->replid,
                    sizeof(server.replid2));
                server.second_replid_offset = server.master_repl_offset+1;
	
                /* Update the cached master ID and our own primary ID to the
                 * new one. */
                memcpy(server.replid,new,sizeof(server.replid));
                memcpy(server.cached_master->replid,new,sizeof(server.replid));
	
                /* Disconnect all the sub-slaves: they need to be notified. */
                disconnectSlaves();
            }

## 两个关键参数说明
### server.replid
* 主节点：保存的是自己的复制ID
> 节点启动时，会在main --> initServerConfig --> changeReplicationId 生成一个随机的复制ID
> 
> failover后提升的新主节点，会调用shiftReplicationId生成。

* 从节点：保存的是主节点的复制ID
> **全量复制时**，在收到+FULLRESYNC时，将主节点的复制ID保存到master\_replid，memcpy(server.master\_replid, replid, offset-replid-1);然后在从节点会在接收rdb文件时保存主节点的复制ID到replid，在readSyncBulkPayload函数中调用replicationCreateMasterClient将server.master\_replid 复制到server.master->replid ，然后调用memcpy(server.replid,server.master->replid,sizeof(server.replid))；在replicationCreateMasterClient函数中能够看出来虽然server.master注释为主节点到从节点的连接，其实创建的时候传入的是从节点建立的到主节点连接的fd，所以是同一个连接
> 
> **部分复制时**，收到+CONTINUE时，包含主节点的复制ID，如果属于failover主节点发生改变，从节点的server.cached_master->replid与主节点复制ID不一致，则需要更新主节点复制ID，在slaveTryPartialResynchronization函数中实现
> 
> **重启时**，关闭前在rdbSaveInfoAuxFields函数中会将replid和repl\_offset保存进rdb文件，重启时会将这两个数据加载进server.replid和cached\_master

### server.replid2
* 主从节点均保存的是当前从节点上一次复制的主节点的复制ID

> **主节点更新：**初始化时, 是40个字符长度为0，只有当主实例发生故障切换时，redis调用shiftReplicationId把自己replid1和master_repl_offset+1分别赋值给master_replid2和second_repl_offset。
> 
> **从节点更新：**在发生failover时，从节点变成主节点时，会将之前复制的主节点的复制ID保存进server.replid2，以便自己作为新主节点时，其他从节点能够成功发起psync，并且返回continue
在从节点failover结束阶段调用clusterFailoverReplaceYourMaster—> replicationUnsetMaster  shiftReplicationId

# 参考资料

