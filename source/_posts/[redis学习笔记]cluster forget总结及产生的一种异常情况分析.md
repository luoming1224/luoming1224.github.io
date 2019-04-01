---
title: '[redis学习笔记]cluster forget总结及产生的一种异常情况分析'
date: 2018-11-26 17:26:52
categories: 'redis学习笔记'
tags: [redis, cluster forget, handshake]
---
cluster forget用于节点下线时，将待下线节点从集群其他节点保存的节点列表中删除。由于redis cluster采用gossip协议交互节点信息及集群状态，所以只要集群中有一个节点知道待下线节点，随着gossip信息交换，集群中的其他节点最终也都知道该节点，正因为此，向集群添加一个节点时，只需要向集群任意一个节点执行cluster meet命令即可，也因此，为了将一个节点完全的从集群中删除，**必须对集群中其他所有节点都发送cluster forget命令。**

同时由于节点设置了一个黑名单，在收到cluster forget命令时，不仅将节点从集群节点列表中删除，同时将该节点加入黑名单，并且设置了60s的过期时间，在60s内，如果从gossip消息中收到该相同节点，不再允许将其加入集群中，所以其实必须在60s内完成对所有节点执行cluster forget命令。

<!--more-->

# 源码实现
## cluster forget
cluster forget实现代码在clusterCommand中，具体代码如下所示：

	else if (!strcasecmp(c->argv[1]->ptr,"forget") && c->argc == 3) {
        /* CLUSTER FORGET <NODE ID> */
        clusterNode *n = clusterLookupNode(c->argv[2]->ptr);

        if (!n) {
            addReplyErrorFormat(c,"Unknown node %s", (char*)c->argv[2]->ptr);
            return;
        } else if (n == myself) {
            addReplyError(c,"I tried hard but I can't forget myself...");
            return;
        } else if (nodeIsSlave(myself) && myself->slaveof == n) {
            addReplyError(c,"Can't forget my master!");
            return;
        }
        clusterBlacklistAddNode(n);
        clusterDelNode(n);
        clusterDoBeforeSleep(CLUSTER_TODO_UPDATE_STATE|
                             CLUSTER_TODO_SAVE_CONFIG);
        addReply(c,shared.ok);
    }

从代码中可以看到，以下三种情形下，将禁止执行cluster forget命令，该命令返回失败：

* cluster forget命令中指定的节点ID在接收命令的节点中保存的集群节点列表中找不到
* cluster forget命令中指定的节点是接收命令的节点自身
* 接收命令的节点为从节点，cluster forget命令中指定的节点为该从节点的主节点

同时可以看出，在cluster forget中，首先调用clusterBlacklistAddNode将下线节点加入黑名单列表，然后调用clusterDelNode将下线节点从接收命令节点中删除。clusterBlacklistAddNode函数如下：

	/* Cleanup the blacklist and add a new node ID to the black list. */
	void clusterBlacklistAddNode(clusterNode *node) {
	    dictEntry *de;
	    sds id = sdsnewlen(node->name,CLUSTER_NAMELEN);
	
	    clusterBlacklistCleanup();
	    if (dictAdd(server.cluster->nodes_black_list,id,NULL) == DICT_OK) {
	        /* If the key was added, duplicate the sds string representation of
	         * the key for the next lookup. We'll free it at the end. */
	        id = sdsdup(id);
	    }
	    de = dictFind(server.cluster->nodes_black_list,id);
	    dictSetUnsignedIntegerVal(de,time(NULL)+CLUSTER_BLACKLIST_TTL);
	    sdsfree(id);
	}

在该函数中，会将下线节点加入nodes\_black\_list中，同时设置了过期时间为CLUSTER\_BLACKLIST\_TTL（60秒），而且在这之前会调用clusterBlacklistCleanup清除黑名单中超时的节点。

## 将发现的节点加入集群
我们知道redis cluster通过gossip消息相互传播节点信息，所以只要集群中任意一个节点知道某节点，集群其他节点都将会知道该节点，该实现在clusterProcessGossipSection函数中，

			/*
             * 当收到cluster forget xxxx命令时，会将xxxx节点从cluster_nodes中删除，
             * 且加入黑名单列表nodes_black_list中
             *
             * 如果node不处于NOADDR状态，并且集群中没有该节点，那么向node发送一个握手的消息
             * 注意，当前sender节点必须是本集群的众所周知的节点（不在集群的黑名单中），否则有加入另一个集群的风险
             *
             */
            if (sender &&
                !(flags & CLUSTER_NODE_NOADDR) &&
                !clusterBlacklistExists(g->nodename))
            {
                clusterStartHandshake(g->ip,ntohs(g->port),ntohs(g->cport));
            }

集群中节点在解析其他节点发送的gossip消息时，如果其他节点gossip消息中携带的节点，在本节点中没有找到，并且该节点不在黑名单中，就会调用clusterStartHandshake函数，和其握手，且将其加入集群节点列表中。在clusterBlacklistExists函数中也会先调用clusterBlacklistCleanup函数，清除黑名单中过期的节点。

所以，如果不能在60秒内，对集群所有其他节点都发送cluster forget命令，将无法将节点下线。

这也是为什么需要设置一个黑名单的原因，否则要下线一个节点，必须在一瞬间同时对所有其他节点都发送cluster forget，要不然极有可能又被重新加入了集群。

# cluster forget产生的一种异常
我们上面说了，要下线一个节点，必须对集群中所有其他节点都发送cluster forget命令，如果没有全部发送，将无法将节点从集群中删除，随着gossip消息的传播，最终该节点又会被加入集群。但是如果同时刚好待下线的节点处于宕机状态，通过cluster nodes命令查看集群状态时，可能会看到一种非常奇怪的现象。

该现象表现为：该宕机节点在没有发送cluster forget的节点中显示为failed状态（即任然属于集群一部分），在发送了cluster forget命令的节点一直显示为handshake状态（即还不属于集群的一部分），且显示handshake状态时的nodeID一直发生变化。在[redis cluster节点handshake状态问题](https://githubmota.github.io/2018/06/15/TODO/)这篇文章中也分析了这样一直情况。

## 现象复现
下面两种情形都可以复现：

（1） 假如待下线的宕机节点为B，首先将节点B宕机，然后在节点A发送 cluster forget nodeID(B)，一段时间后观察集群各个节点cluster nodes命令返回结果

（2） 假如集群中存在两个宕机节点，分别为B和C，现要将B节点下线，即使同时在60秒内对集群中所有其他节点都执行了cluster forget命令（节点C除外，因为C此时宕机，不能接收命令），然后重启节点C，一段时间后观察各个节点cluster nodes命令返回结果。

![](https://i.imgur.com/GK0hz7c.png)

从图中可以看到，一部分节点认为节点B状态为failed，一部分节点认为节点B的状态为handshake，并且你可以不停地查看cluster nodes，会发现显示为handshake状态时的节点B nodeID会不停的变化。

## 分析
执行过cluster forget命令的节点（假如为A），在过了60秒后，由于其他节点发送的gossip消息中包含节点B的信息，节点A会调用clusterStartHandshake和节点B进行握手，试图将节点B加入集群，在clusterStartHandshake函数中会设置节点B的状态为handshake，同时由于此时节点A还不知道节点B的nodeID，所以会在clusterStartHandshake函数中调用createClusterNode时，传入NULL，由createClusterNode调用getRandomHexChars为节点B随机生成一个nodeID，待握手成功，收到节点B发送的消息时，再把随机ID替换成节点B自身真正的ID。

redis在clusterCron中会检查处于handshake状态的节点是否握手超时，如果超时，则会将其从节点列表中删除，由于节点B处于宕机状态，所以肯定会握手超时。

		/* A Node in HANDSHAKE state has a limited lifespan equal to the
         * configured node timeout. */
        if (nodeInHandshake(node) && now - node->ctime > handshake_timeout) {
            clusterDelNode(node);
            continue;
        }

超时时间handshake_timeout默认为15秒，

删除后，由于其他节点还会继续在gossip消息中发送节点B，于是节点A又会调用clusterStartHandshake重复上面的过程，所以我们就能看到在节点A中节点B一直处于handshake状态，且其nodeID每隔一段时间（15秒）不断变化。

因此，在处理节点下线时，处理cluster forget命令时，需要特别注意，尤其有集群中有多个节点处于宕机时，特别注意cluster forget命令执行失败的情况。

# 参考资料
* [redis cluster节点handshake状态问题](https://githubmota.github.io/2018/06/15/TODO/)
* [https://redis.io/commands/cluster-forget](https://redis.io/commands/cluster-forget)