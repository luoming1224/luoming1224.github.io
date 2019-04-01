---
title: '[redis学习笔记]redis字典rehash机制导致数据淘汰分析'
date: 2018-11-14 18:59:13
categories: 'redis学习笔记'
tags: [redis, 扩容, rehash, 淘汰]
---
在redis中，如果在开启了maxmemory，且在内存使用量接近maxmemory时，刚好出现哈希表的rehash过程，会导致redis内存瞬间突增，内存使用量超过maxmemory，同时如果没有开启自动扩容且开启了淘汰策略的情况下，会导致redis中数据被淘汰。

<!--more-->

# 案例
线上某个redis集群所有节点突然陆陆续续出现内存突增，瞬间增长1G左右，业务方开发却说业务没有这么大的写入数据，且监控显示key数量、流入流量均比较正常，没有猛增，部分节点出现大量key淘汰。使用内存和key淘汰监控图表如下所示

![](https://i.imgur.com/cV5W6Sc.png)

![](https://i.imgur.com/T3OWaYn.png)

在无意中重新温习美团的一篇文章时，验证了该现象由redis rehash机制重新分配一个新的更大的哈希表用于重哈希数据，导致内存突增，内存因为rehash增长后使用量超过了节点的maxmemory，且该集群未开启自动扩容，但开启了驱逐策略，所以导致节点淘汰了大量数据。当时节点中key的数量刚好为33554432左右，处于rehash阈值范围，所以原因得到了验证。由于及时发现，所以运维及时调整了集群所有节点的maxmemory，所以只有最先触发rehash的几个节点有数据被淘汰，其余节点幸免了。

# 分析

在[《[redis学习笔记]redis渐进式rehash机制》](https://luoming1224.github.io/2018/11/12/[redis%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0]redis%E6%B8%90%E8%BF%9B%E5%BC%8Frehash%E6%9C%BA%E5%88%B6/)一文中详细介绍了redis渐进式rehash机制，从该文章中我们可以看到，当Redis触发Resize后，就会动态分配一块内存，最终由ht[1].table指向，动态分配的内存大小为：realsize\*sizeof(dictEntry\*)，table指向dictEntry\*的一个指针，大小为8bytes（64位OS），即ht[1].table需分配的内存大小为：8\*2\*2^n （n大于等于2）。

梳理一下哈希表大小和内存申请大小的对应关系：

![](https://i.imgur.com/SwvbsWj.png)

当key数量达到33554432时，会触发哈希表rehash，分配一个512M的新哈希表；为什么线上内存增长了1G左右，有点没有想明白，待继续研究。

# 现象复现
测试环境找一个节点（由于我们默认创建的都是集群模式，所以需要打开配置文件关闭集群模式，重启节点，否则会出现数据写入丢失的错觉，因为集群模式如果key不属于节点所负责的槽，会写入失败），先写入8300000个key，我们通过info stats和debug htstats 0命令看一下此时的节点统计数据。

![](https://i.imgur.com/kQYoRtf.png)

![](https://i.imgur.com/he08sOo.png)

从图中可以看到此时evicted_keys为0，没有驱逐数据，哈希表中key的数量为8300000，哈希表的大小为8388608，而且通过info memory命令查看当前的内存使用量为771M，对照上面的表格，下一次rehash需要分配的新哈希表大小为128M，所以此时通过config set maxmemeory命令设置maxmemory为900M（943718400）。

然后继续写入90000个数据，此时我们再次查看节点的状态。首先需要明确一下，这90000个数据是不足以导致内存超过900M的。

![](https://i.imgur.com/4x8SMU8.png)

![](https://i.imgur.com/JpAwf8w.png)

通过debug htstats 0命令可以看到节点正在扩容，分配了一个新的哈希表，新的哈希表大小为16777216，通过info stats命令可以看到有大量数据被淘汰了。看一下内存增长监控图表：

![](https://i.imgur.com/hxVRO9S.png)

可以看到内存从771M瞬间增长到了900M，一段时间后，又降回了835M，因为rehash完成后，会释放原有的哈希表，所以该现象与线上完全一致。而且内存增长的大小128M符合上一节列出的表格对应关系。

我们可以得出结论：
当Redis 节点中的Key总量到达临界点后，Redis就会触发Dict的扩展，进行Rehash。申请扩展后相应的内存空间大小。如果刚好rehash时扩展后的内存空间超过maxmemory值，会导致数据驱逐淘汰。

# Redis Rehash机制优化

那么针对在Redis满容驱逐状态下，如何避免因Rehash而导致Redis抖动的这种问题。

* 在Redis Rehash源码实现的逻辑上，加上了一个判断条件，如果现有的剩余内存不够触发Rehash操作所需申请的内存大小，即不进行Resize操作；
* 通过提前运营进行规避，比如容量预估时将Rehash占用的内存考虑在内，或者开启自动扩容，或者适当调低内存告警阈值，可以及时增加内存。

在dictExpand函数中，分配内存以前，增加如下判断：

	if (server.maxmemory) {
        size_t mem_used = zmalloc_used_memory();
        size_t overhead = freeMemoryGetNotCountedMemory();
        mem_used = (mem_used > overhead) ? mem_used-overhead : 0;
        mem_used += (realsize* sizeof(dictEntry*));
        if (mem_used >= server.maxmemory) return DICT_ERR;
        if (mem_used >= (server.maxmemory * 0.9)) return DICT_ERR;
    }

增加优化代码后，再次测试，依然是先导入8300000个key，根据info memory返回的内存使用量+128M，设置一个合理的maxmemory，再次导入90000个key。测试结果如下图所示：

![](https://i.imgur.com/UQ9geTz.png)

从图中可以看到，此时的key数量已经为8390000，而哈希表大小依然是8388608，key数量超过哈希表大小，但是没有发生rehash过程。因为内存没有超过maxmemory，所以通过info stats命令查看到evicted_keys为0，没有发生数据淘汰。

# 参考资料
* [美团针对Redis Rehash机制的探索和实践](https://tech.meituan.com/Redis_Rehash_Practice_Optimization.html)

	该文章中还提到了redis rehash带来的另一个问题：Redis使用Scan清理Key由于Rehash导致清理数据不彻底

# 附测试用例代码

	public static void main(String[] args) {
        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxTotal(100);
        config.setMinIdle(100);
        config.setMinIdle(20);

        JedisPool pool = new JedisPool(config, "172.25.61.70", 8212);


        ExecutorService executorService = Executors.newFixedThreadPool(20);
        final AtomicInteger aa = new AtomicInteger();
        for (int j = 0; j < 80; j++) {
            executorService.submit(() -> {
                int counter = aa.addAndGet(1);
                int incr = 0;
                String threadName = Thread.currentThread().getName();

                Jedis jedis = pool.getResource();
                Pipeline pipeline = jedis.pipelined();
                for (int m = 0; m < 100; m++) {
                    for (int i = 0; i < 1000; i++) {
                        incr++;
                        pipeline.set("key" + counter + threadName + RandomUtil.getRandomString(10) + "a" + incr, "value" + i);
                    }
                    pipeline.sync();
                }
                try {
                    pipeline.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            });
        }

        try {
            TimeUnit.HOURS.sleep(24);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

RandomUtil代码：

	import java.util.Random;
	
	public class RandomUtil {
	    private static int DEFAULT_STRING_LENGTH = 100;
	    private static String BASE = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ,./?<>:\"|';][{}+_)(*&^%$#@!~`0123456789";
	
	    protected static Random random = new Random();
	
	    public static String getRandomString(int length) {
	        StringBuilder sb = new StringBuilder();
	        for (int i = 0; i < length; i++) {
	            int number = random.nextInt(BASE.length());
	            sb.append(BASE.charAt(number));
	        }
	        return sb.toString();
	    }
	
	    public static String getRandomString() {
	        return getRandomString(DEFAULT_STRING_LENGTH);
	    }
	
	    @Deprecated
	    public static String getRamdomString() {
	        return getRandomString(DEFAULT_STRING_LENGTH);
	    }
	
	    @Deprecated
	    public static String getRamdomString(int length) {
	        StringBuilder sb = new StringBuilder();
	        for (int i = 0; i < length; i++) {
	            int number = random.nextInt(BASE.length());
	            sb.append(BASE.charAt(number));
	        }
	        return sb.toString();
	    }
	}
