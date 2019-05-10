---
title: '[redis学习笔记]redis中Geo命令介绍'
date: 2019-04-08 20:51:27
categories: 'redis学习笔记'
tags: [redis, GEO, GeoHash, 地理信息]
---
redis3.2开始新增了GEO地理位置相关的命令，本文对Geo相关命令使用及实现原理进行介绍。Redis将地理位置的52位GeoHash值作为有序集合的score，将地理位置存放在有序集合中进行保存。阅读本文前可以先阅读[《GeoHash算法详解及其实现》](https://luoming1224.github.io/2019/04/04/[redis%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0]GeoHash%E7%AE%97%E6%B3%95%E8%AF%A6%E8%A7%A3%E5%8F%8A%E5%85%B6%E5%AE%9E%E7%8E%B0/)，了解GeoHash算法。

<!--more-->

# GEOADD
增加地理位置坐标，命令格式如下：
	
	GEOADD key longitude latitude member [longitude latitude member ...]
	
Redis中接受的有效的经度范围为-180到180度，有效纬度范围为-85.05112878到 85.05112878度（靠近南北极的一小块地方是无法生成索引的）。

## 实现方式
Redis内部使用有序集合来保存key，每一个member的score大小为一个52位的Geohash值（double类型精度为52位）。

实际上Redis内部实现的时候就是将GEOADD命令转换成ZADD命令来实现的。（这也解释了为什么没有专门的georem命令，地理位置信息是通过使用ZREM命令来删除成员。）

源码实现如下所示：

	/* GEOADD key long lat name [long2 lat2 name2 ... longN latN nameN] */
	void geoaddCommand(client *c) {
	    /* Check arguments number for sanity. */
	    if ((c->argc - 2) % 3 != 0) {
	        /* Need an odd number of arguments if we got this far... */
	        addReplyError(c, "syntax error. Try GEOADD key [x1] [y1] [name1] "
	                         "[x2] [y2] [name2] ... ");
	        return;
	    }
	
	    int elements = (c->argc - 2) / 3;
	    int argc = 2+elements*2; /* ZADD key score ele ... */
	    robj **argv = zcalloc(argc*sizeof(robj*));
	    argv[0] = createRawStringObject("zadd",4);
	    argv[1] = c->argv[1]; /* key */
	    incrRefCount(argv[1]);
	
	    /* Create the argument vector to call ZADD in order to add all
	     * the score,value pairs to the requested zset, where score is actually
	     * an encoded version of lat,long. */
	    int i;
	    for (i = 0; i < elements; i++) {
	        double xy[2];
	
	        if (extractLongLatOrReply(c, (c->argv+2)+(i*3),xy) == C_ERR) {
	            for (i = 0; i < argc; i++)
	                if (argv[i]) decrRefCount(argv[i]);
	            zfree(argv);
	            return;
	        }
	
	        /* Turn the coordinates into the score of the element. */
			// 根据坐标计算出 geohash 值
	        GeoHashBits hash;
	        geohashEncodeWGS84(xy[0], xy[1], GEO_STEP_MAX, &hash);
	        GeoHashFix52Bits bits = geohashAlign52Bits(hash);
	        robj *score = createObject(OBJ_STRING, sdsfromlonglong(bits));
	        robj *val = c->argv[2 + i * 3 + 2];
	        argv[2+i*2] = score;
	        argv[3+i*2] = val;
	        incrRefCount(val);
	    }
	
	    /* Finally call ZADD that will do the work for us. */
	    replaceClientCommandVector(c,argc,argv);
		// 将刚才创建的元素全部添加到有序集合里面
	    zaddCommand(c);
	}

## 使用示例

	127.0.0.1:6379> geoadd china 120.19 30.26 hangzhou
	(integer) 1
	127.0.0.1:6379> ZRANGE china 0 -1 withscores
	1) "hangzhou"
	2) "4054134100314554"

# GEODIST
返回两点间距离，命令格式如下：
	
	GEODIST key member1 member2 [unit]

单位可选项为m（米，默认值）, km（千米）,mi（英里）,ft（英尺）。

返回double值，若有任意一个member不存在，则返回NULL.

## 实现方式
使用WGS84坐标系统，计算距离时使用Haversine公式。由于地球并不是严格标准的，计算出来的距离有最大约0.5%的误差。

Haversine formula 公式介绍：[https://en.wikipedia.org/wiki/Haversine_formula](https://en.wikipedia.org/wiki/Haversine_formula)

## 使用示例

	127.0.0.1:6379> geoadd china 121.480306 31.236329 shanghai
	(integer) 1
	127.0.0.1:6379> geodist china hangzhou shanghai km
	"164.3309"
	127.0.0.1:6379> geodist china hangzhou suzhou
	(nil)


# GEOHASH
返回key中对应成员的geohash值。命令格式如下：

	GEOHASH key member [member ...]

如果member不存在，则返回null。

## 特点
该命令返回一个11个字符的GeoHash字符串，返回的GeoHash有如下特性：


- 可以从字符串的右侧移除部分字符，使得字符串变短，这样做会损失精度，但是依然会定位到同样的区域。
- 拥有相同前缀的字符串表示的区域位置接近，但是反过来不一定成立。也有可能地理位置接近，但是其字符串前缀不相同。

## 实现方式
Redis在内部生成有序集合成员score时的geohash值与标准的算法略有差异（Redis内部使用-85,85作为维度范围，标准使用-90,90）。

这个命令返回的是**标准值**，与[https://en.wikipedia.org/wiki/Geohash](https://en.wikipedia.org/wiki/Geohash)中标准算法和geohash.org网站的结果一致。代码如下：

			/* The internal format we use for geocoding is a bit different
             * than the standard, since we use as initial latitude range
             * -85,85, while the normal geohashing algorithm uses -90,90.
             * So we have to decode our position and re-encode using the
             * standard ranges in order to output a valid geohash string. */

            /* Decode... */
			//先解码出地理位置的坐标
            double xy[2];
            if (!decodeGeohash(score,xy)) {
                addReply(c,shared.nullbulk);
                continue;
            }

            /* Re-encode */
			//再使用纬度范围[-90,90]重新编码标准的geohash值
            GeoHashRange r[2];
            GeoHashBits hash;
            r[0].min = -180;
            r[0].max = 180;
            r[1].min = -90;
            r[1].max = 90;
            geohashEncode(&r[0],&r[1],xy[0],xy[1],26,&hash);

            char buf[12];
            int i;
            for (i = 0; i < 11; i++) {
                int idx = (hash.bits >> (52-((i+1)*5))) & 0x1f;
                buf[i] = geoalphabet[idx];
            }
            buf[11] = '\0';
            addReplyBulkCBuffer(c,buf,11);

## 使用示例

	127.0.0.1:6379> geohash china hangzhou shanghai suzhou
	1) "wtmknuxtmb0"
	2) "wtw3sq7kb30"
	3) (nil)

可以在[http://geohash.org](http://geohash.org)中使用geohash，如下的url:[http://geohash.org/wtmknuxtmb0](http://geohash.org/wtmknuxtmb0)可以在地图中定位到杭州

# GEOPOS
返回指定成员的经纬度坐标，成员不存在则返回null。命令格式如下：
	
	GEOPOS key member [member ...]

经纬度坐标是被转成52位的GeoHash保存起来的，返回的时候重新解码成经纬度坐标。由于精度问题，返回值可能与设置的值略有差异。

## 使用示例
	
	127.0.0.1:6379> geoadd china 120.19 30.26 hangzhou
	(integer) 0
	127.0.0.1:6379> geopos china hangzhou suzhou
	1) 1) "120.18999785184860229"
	   2) "30.25999927289620928"
	2) (nil)

# GEORADIUS，GEORADIUSBYMEMBER，GEORADIUS\_RO，GEORADIUSBYMEMBER\_RO
获取指定范围内的地理位置集合，命令格式如下：
	
	GEORADIUS key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]
	
	GEORADIUSBYMEMBER key member radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC] [STORE key] [STOREDIST key]

	GEORADIUS_RO key longitude latitude radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC]
	
	GEORADIUSBYMEMBER_RO key member radius m|km|ft|mi [WITHCOORD] [WITHDIST] [WITHHASH] [COUNT count] [ASC|DESC]

GEORADIUS是获取任意经纬度周围的地理集合，GEORADIUSBYMEMBER是获取某个地理位置周围的地理位置集合。它们的内部实现和可选参数是一致的。
可选项：

- WITHDIST: 同时返回地理位置与给定位置的距离
- WITHCOORD: 同时返回地理位置的经纬度坐标
- WITHHASH: 同时返回Redis内部的GeoHash值（非标准算法值），一般用于debug
- ASC|DESC：结果按距离升降序排序
- STORE|STOREDIST: 结果存到新的有序集合中，前者以GeoHash值做score，后者以与指定位置的距离作score，该选项与WITH[DIST|COORD|HASH]选项冲突，这两种类型选项只能二选一
- COUNT： 指定返回的元素个数，必须大于0，且最好同时指定排序规则，如果未指定排序规则，默认为ASC（升序）

GEORADIUS\_RO和GEORADIUSBYMEMBER\_RO是redis 3.2.10后新引进的两个命令。由于GEORADIUS和GEORADIUSBYMEMBER命令存在store和storedist选项，在redis中该两个命令被标记为写命令类型。即使在从节点的连接设置了readonly模式下，标记为写命令类型的命令，依然会收到MOVED消息，被转向到相应主节点。所以redis新增了该两个命令的只读版本，这两个命令除了不支持store和storedist选项外，其他可选参数与GEORADIUS和GEORADIUSBYMEMBER一致。新增的两个只读命令在从节点连接设置readonly模式下可以在从节点执行。

## 使用示例

	127.0.0.1:6379> GEORADIUSBYMEMBER china hangzhou 300 km 
	1) "hangzhou"
	2) "shanghai"
	127.0.0.1:6379> GEORADIUSBYMEMBER china hangzhou 300 km store new_key
	(integer) 2
	127.0.0.1:6379> zrange new_key 0 -1
	1) "hangzhou"
	2) "shanghai"
	127.0.0.1:6379> GEORADIUSBYMEMBER_RO china hangzhou 300 km withcoord count 1 desc
	1) 1) "shanghai"
   	2) 1) "121.48030668497085571"
       2) "31.23632823849270324"
	127.0.0.1:6379> GEORADIUSBYMEMBER china hangzhou 300 km withdist store new_key_test
	(error) ERR STORE option in GEORADIUS is not compatible with WITHDIST, WITHHASH and WITHCOORDS options

## 实现方式
geohash编码越长精度越高，相反编码越短，表示的范围越大，前缀相同的字符串表示的范围接近，根据中心点位置和搜索范围距离计算geohash编码的长度和geohash编码，然后搜索以该geohash编码为前缀的点以及周边８个范围内的点，返回满足要求的点。

函数geohashGetAreasByRadiusWGS84根据中心点位置和搜索范围距离计算geohash编码的长度和geohash编码，以及在geohash长度编码的基础上，计算周边8个区块的geohash。

函数membersOfAllNeighbors对中心点以及它周边八个方向进行查找，找出所有范围内的元素，返回满足搜索距离范围的点。该函数中依次对中心点及周边8个区块调用membersOfGeoHashBox函数。

	/* Obtain all members between the min/max of this geohash bounding box.
	 * Populate a geoArray of GeoPoints by calling geoGetPointsInRange().
	 * Return the number of points added to the array. */
	int membersOfGeoHashBox(robj *zobj, GeoHashBits hash, geoArray *ga, double lon, double lat, double radius) {
	    GeoHashFix52Bits min, max;
	
	    scoresOfGeoHashBox(hash,&min,&max);
	    return geoGetPointsInRange(zobj, min, max, lon, lat, radius, ga);
	}

	// 计算出从有序集合里面查找指定范围内的所有位置所需的 min 值和 max 值。
	void scoresOfGeoHashBox(GeoHashBits hash, GeoHashFix52Bits *min, GeoHashFix52Bits *max) {
	    /* We want to compute the sorted set scores that will include all the
	     * elements inside the specified Geohash 'hash', which has as many
	     * bits as specified by hash.step * 2.
	     *
	     * So if step is, for example, 3, and the hash value in binary
	     * is 101010, since our score is 52 bits we want every element which
	     * is in binary: 101010?????????????????????????????????????????????
	     * Where ? can be 0 or 1.
	     *
	     * To get the min score we just use the initial hash value left
	     * shifted enough to get the 52 bit value. Later we increment the
	     * 6 bit prefis (see the hash.bits++ statement), and get the new
	     * prefix: 101011, which we align again to 52 bits to get the maximum
	     * value (which is excluded from the search). So we get everything
	     * between the two following scores (represented in binary):
	     *
	     * 1010100000000000000000000000000000000000000000000000 (included)
	     * and
	     * 1010110000000000000000000000000000000000000000000000 (excluded).
	     */
	    *min = geohashAlign52Bits(hash);
	    hash.bits++;
	    *max = geohashAlign52Bits(hash);
	}

由于geoadd命令将地理位置以geohash为值保存进zset数据类型中，所以根据中心点及周边8个区块的geohash值，计算出该区块中所有点的geohash最小值与最大值，再根据最小值与最大值使用zset命令查找出所有的点。scoresOfGeoHashBox函数即为计算该最小值与最大值。

函数geoGetPointsInRange即为根据最小值与最大值在zset中查找所有点，然后计算这些点是否在搜索范围内。并返回结果。

# 参考文章
[阿里云Redis GEO地理位置功能上线啦](https://yq.aliyun.com/articles/62844)
[Redis GEO 源码注释](http://blog.huangz.me/diary/2015/annotated-redis-geo-source.html)