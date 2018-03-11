---
layout: post
title: 基于Raft的分片(shards)分布式K/V存储系统
author: hawker
permalink: /2017/11/raft-shards1.html
date: 2017-11-09 18:00:00
category:
    - 编程
tags:
    - raft
    - shards
    - distributed-storage
---
数据分片(shards)存储是一种普遍需求，我将分成两篇文章介绍基于Raft的分片K/V存储系统基本原理以及实现。代码实现基于Golang，关于Raft分布式一致性算法本身的实现已经在前面的问题阐述了，在此不再赘述。

## 分片K/V存储系统概念
当然说实现前，我们需要先阐述分片K/V存储是什么，以及实现分片存储的必要性。

分片存储概念：我们在前面实现Raft算法时，用Raft实现了一个分布式K/V存储。分片其实就是逻辑上将整体的数据分成不同的部分、分片，每个数据分片负责一部分的K/V内容管理。举个🌰，可以将"a"为前缀的key作为一个shard；以"b"为前缀的key作为另一个shard。

分布存储目的：最重要的原因是提供存储系统性能，每个分片独立处理各自的读/写，并发性极大提高，进而达到提升系统整体吞吐的目的。此外，shard在实际工作中可以达到保护的目的，因为不同shared之间逻辑上分离，避免了整体的“单点”问题，一个分片出现问题，其他分片正常提供服务。

难点问题：数据分片的归属对应关系维护、不同数据分片迁移的一致性处理、用户如何从正确的数据分片获取内容。


## 如何实现分片K/V存储系统
上面阐述了分片存储的概念和意义，本节说明实现分片K/V存储的基本原理。系统将由两大部分组成：分片备份集群组（replica groups)；分片配置管理集群（shard master).

1. replica groups：存储数据的多个分片集群组，每个group独立使用一套Raft维护属于自己的数据分片一致性操作。这里看出为什么分片可以提高并发和系统整体吞吐量。
2. shard master：该部分决定了每个replica group负责的数据分片，维护分片与replica group的归属对应关系，其实是整个系统的配置文件管理中心。随着分片数据的迁移，配置文件一直在变化。clients从shard master获取key归属的replica group；replica group通过shard master确认自己负责提供哪些分片存储服务。

ok，那么看起来好像挺简单的，其实上节我已经阐述了具体的难点问题。整个系统需要支持不能shard在replica group迁移，并维护数据不丢失，用户能够通过shard master去正确的replica group获取内容。数据迁移的原因：

1. 某个replica group负责重了，需要迁移出去一部分shard降低负载。
2. replica group具有join和leave操作，需要重新分配shard；一方面实现负载均衡、另一方面保证leave后shard被迁移到运行中的group正常提供服务。

整个shard迁移可以看作是shard master的配置变化(reconfiguration)，需要保证reconfiguration时client发起关于K/V的增、删、改、查在不同replica group达成一致性要求，不会造成数据破坏、client能够拿到正确内容。reconfiguration时不同replica group之间的shard迁移需要合理安排策略，避免内部迁移交互出错。想象实现说明可以参考：[传送门](https://pdos.csail.mit.edu/6.824/labs/lab-shard.html)

我们将花两篇文章的内容，介绍实现shard master以及replica groups整个交互逻辑。

## 实现shard master

本篇文章我们先谈一谈如何实现shard master，配置中心的逻辑与之前实现的分布式K/V逻辑类似。主要提供给client四个接口：Join, Leave, Mov,和 Query。这些接口的逻辑比较简单，所以不再赘述，可以直接参考代码实现：[传送门](https://github.com/hawkxiang/Distributed-System/blob/master/shardmaster/server.go)

接下来我主要介绍与reconfiguration相关的基础实现逻辑。首先，每次涉及配置文件reconfiguration的Join, Leave发生时，需要更新配置文件，配置文件的结构体如下：

{% highlight Go %}
type Config struct {
	Num    int              // config number
	Shards [NShards]int     // shard -> gid
	Groups map[int][]string // gid -> servers[]
}
{% endhighlight %}

`Num`：配置文件编号，在整个系统运行过程中递增，每次reconfiguration时递增编号。
`Shards`：存储每个shard归属的repica group，通过数组维护多对一的关系。
`Groups`：每个repica group集群包括的具体节点。

每次配置文件变动时，主要操作是递增配置文件编号、深度拷贝旧配置文件内容，将新配置存入数组中：

{% highlight Go %}
func (sm *ShardMaster) ChangeConfig() *Config {
	oldCfg := &sm.configs[sm.lastIdx]  //数组形式保存所有的配置
	var newCfg Config
	newCfg.Num = oldCfg.Num + 1
	newCfg.Shards = [NShards]int{}
	newCfg.Groups = make(map[int][]string)
	//deep copy 深度拷贝
	for i, g := range oldCfg.Shards {
		newCfg.Shards[i] = g
	}

	for g, srvs := range oldCfg.Groups {
		newCfg.Groups[g] = srvs
	}
	//加锁进入数组，旧的配置需要保留，数据迁移时需要使用
	sm.muCfg.Lock()
	defer sm.muCfg.Unlock()
	sm.lastIdx++
	sm.configs = append(sm.configs, newCfg)
	return &sm.configs[sm.lastIdx]
}
{% endhighlight %}

ok，说了这些还是没有谈到如何在reconfiguration后实现数据迁移，以及如何保证不同replica group之间的负载均衡。负载均衡的算法有很多，本实现重点是分布式shard与迁移数据一致性，这里采用简单的逻辑实现分片迁移时的负载均衡。

首先，扫描配置中每个replica group拥有的shard个数，能够获取到拥有最多、最少分片的两个group_id对（from, to)，如下：

{% highlight Go %}
func GidMovePair(c *Config) MovePair {
	min_id, min_num, max_id, max_num := 0, int(^uint(0) >> 1), 0, -1
	counts := make(map[int]int)
	for g := range c.Groups {
		counts[g] = 0
	}
	//统计每个group拥有的shard个数
	for _, g := range c.Shards {
		counts[g]++
	}
	//找到最大、最小shards的group id
	for g := range counts {
		_, ok := c.Groups[g]
		if  ok && min_num > counts[g] {
			min_id, min_num = g, counts[g]
		}
		if ok &&  max_num < counts[g] {
			max_id, max_num = g, counts[g]
		}
	}
	//初始化case
	for _, g := range c.Shards {
		if 0 == g {
			max_id = 0
		}
	}
	return MovePair{max_id, min_id}
}
{% endhighlight %}

然后，进行rebalance主要分成两个case：

1. join：根据两个group_id对从最多的group里面选择一个shard，迁移到最少的group中：from --> to，循环这个过程直到shard平均分布。
2. leave：从被摘除的group中，不断的向拥有最少shard的group迁移数据，直到被摘除group内容迁移完毕。

{% highlight Go %}
func (sm *ShardMaster) rebalance(gid int, isLeave bool) {
	c := &sm.configs[sm.lastIdx]
	//不断的循环直到重新负载均衡
	for i := 0; ; i++ {
	    //每次都需要重新计算，最大、最小shard的分组id
		pair := GidMovePair(c)
		//分为两种case进行shard迁移
		if isLeave {
			s := GetShardByGid(gid, c)
			if -1 == s {
				return
			}
			c.Shards[s] = pair.To
		} else {
			if i == NShards / len(c.Groups) {
				return
			}
			s := GetShardByGid(pair.From, c)
			c.Shards[s] = gid
		}
	}
{% endhighlight %}

最后，以leave的例子来看看，rebalance具体是如何使用的。

{% highlight Go %}
func (sm *ShardMaster) doLeave(gids []int) {
    //深度拷贝产生新的配置内容
	cfg := sm.ChangeConfig()
	for _, gid := range gids {
		if _, ok := cfg.Groups[gid];ok{
			//移除leave分组
			delete(cfg.Groups, gid)
			//rebalance
			sm.rebalance(gid, true)
		}
	}
}
{% endhighlight %}

下篇文章我们将讨论replica group的实现，以及不同group直接shard实际迁移的流程，和一致性维护。
