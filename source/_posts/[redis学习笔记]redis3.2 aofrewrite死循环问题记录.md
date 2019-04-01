---
title: '[redis学习笔记]redis3.2 aofrewrite死循环问题记录'
date: 2018-11-26 14:22:54
categories: 'redis学习笔记'
tags: [redis, aofrewrite, bug, 死循环]
---
redis3.2中，当节点正在执行aof文件重写时，如果此时调用config set aofrewrite no命令关闭节点aof持久化，会触发bug导致aof文件重写进入死循环，表现为节点不停的执行aof文件重写，严重影响节点性能和磁盘性能。

<!--more-->

# 背景
线上某机器发生磁盘繁忙告警，运维发现该机器上某节点在aof持久化关闭的状态下，持续不断的执行aofrewrite，严重影响磁盘性能。问题日志如下图所示：

![](https://i.imgur.com/JArBs0J.png)

# aof重写原理
## 触发aof重写
在serverCron函数中，redis会每次在该循环中检查是否满足aofrewrite条件，如果满足条件就会执行aofrewrite。在redis3.2中该部分代码如下所示：

	/* Trigger an AOF rewrite if needed */
         if (server.rdb_child_pid == -1 &&
             server.aof_child_pid == -1 &&
             server.aof_rewrite_perc &&
             server.aof_current_size > server.aof_rewrite_min_size)
         {
            long long base = server.aof_rewrite_base_size ?
                            server.aof_rewrite_base_size : 1;
            long long growth = (server.aof_current_size*100/base) - 100;
            if (growth >= server.aof_rewrite_perc) {
                serverLog(LL_NOTICE,"Starting automatic rewriting of AOF on %lld%% growth",growth);
                rewriteAppendOnlyFileBackground();
            }
         }

可以看出需要出发aof文件重写需要同时满足如下几个条件：

* 当前没有子进程在执行aof文件重写或者生成RDB文件
* 节点设置了aof_rewrite_perc配置，该配置默认值为100，即aof文件大小比上次重写时大一倍则再次进行aof文件重写
* aof文件大小大于64M，主要用于控制触发第一次aof重写
* aof文件比上一次重写时增长了一倍大小或者是满足第一次重写的条件

此处我们发现，该逻辑有个明显的问题，没有判断是否开启了aof持久化，我们翻看了一下redis4.0代码，发现该逻辑增加了一个判断条件，判断aof是否开启（server.aof_state == AOF_ON），显然虽然该逻辑有点问题，但显然不是这个导致的。

## aof重写完成
在serverCron函数中，当有子进程在执行aof文件重写时，会检查是否收到子进程完成的通知，如果收到完成信号通知，会调用backgroundRewriteDoneHandler函数进行剩下的一些工作，比如将重写期间的缓冲区内容写入新的aof文件，用新的aof文件替换旧的aof文件等。其主要代码实现如下所示（省略了部分代码）：

		if (server.aof_fd == -1) {
            /* AOF disabled, we don't need to set the AOF file descriptor
             * to this new file, so we can close it. */
            close(newfd);
        } else {
            /* AOF enabled, replace the old fd with the new one. */
            oldfd = server.aof_fd;
            server.aof_fd = newfd;
            if (server.aof_fsync == AOF_FSYNC_ALWAYS)
                aof_fsync(newfd);
            else if (server.aof_fsync == AOF_FSYNC_EVERYSEC)
                aof_background_fsync(newfd);
            server.aof_selected_db = -1; /* Make sure SELECT is re-issued */
            aofUpdateCurrentSize();
            server.aof_rewrite_base_size = server.aof_current_size;

            /* Clear regular AOF buffer since its contents was just written to
             * the new AOF from the background rewrite buffer. */
            sdsfree(server.aof_buf);
            server.aof_buf = sdsempty();
        }

# 问题分析
该现象非常明显应该是上面的流程中某处有bug导致了不停的执行aof文件重写，所以我们临时改了一下server.aof_rewrite_min_size变量，让节点不要再次进入aofrewrite流程。修改完后，可以看到磁盘繁忙度明显下降，且节点日志也不再进入aofrewrite了。磁盘繁忙监控图表如下所示：

![](https://i.imgur.com/gHDKlWm.png)

从上面可以看到，如果aof文件重写正常完成的话，在backgroundRewriteDoneHandler函数中会更新server.aof_rewrite_base_size为当前文件的大小。当时看到问题时，有推测应该是某种原因导致该参数更新失败了，在继续追踪前，感觉该问题是一个bug引起的，所以想搜一下是否有其他人遇到该问题。搜索了一下发现确实有人遇到，见参考资料1.

在搜到的资料中，有人提到当节点正在执行aof文件重写时，如果此时刚好调用config set aofrewrite no命令关闭aof持久化，会导致aofrewrite死循环执行。

因为在关闭aof持久化时，会调用stopAppendOnly设置server.aof_fd为-1，如果关闭的时候正在进行aofrewrite操作，当子进程结束后，从而导致在backgroundRewriteDoneHandler函数中上面展示的代码中逻辑进入if，永远无法更新server.aof_rewrite_base_size，从而周而复始不停的执行aof文件重写。

然后我们查看了一下该集群的任务记录，确实该集群之前是开启aof持久化的，并且在不久前关闭了。记录如下所示

![](https://i.imgur.com/xTAvriE.png)

并且在该服务器上找到该节点一个最后的临时aof文件，且最后一次更新时间与记录中关闭aof持久化操作的时间一致，也就验证了确实正好在节点执行aof文件重写时，关闭了aof持久化策略，触发了该bug。

![](https://i.imgur.com/Bd0nPTh.png)

# 参考资料

[1] [https://github.com/antirez/redis/issues/4545](https://github.com/antirez/redis/issues/4545)