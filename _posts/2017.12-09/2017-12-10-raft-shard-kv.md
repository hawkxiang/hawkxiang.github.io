---
layout: post
title: 基于Raft的分片(shards)分布式K/V存储——2
author: hawker
permalink: /2017/12/raft-shards-kv.html
date: 2017-12-10 18:00:00
category:
    - 编程
tags:
    - raft
    - shards
    - distributed-storage
---
 上一篇文章已经介绍了数据分片(shards)存储基本情况，本文基于上文的shared master，实现分片存储k/v的replica groups。上篇文章主要介绍在数据分片中配置变动如何保证一致性，这篇的重点是数据本身存储的一致性以及不同分组之间的数据迁移。这部分的实现代码见：[代码实现](https://github.com/hawkxiang/Distributed-System/tree/master/shardkv)

## 涉及数据分组和迁移的数据结构

首先，看一看replica group如何存储不同shard数据的结构：`db [shardmaster.NShards]map[string]string`。简单说每个group建立了一个数组，shared是对应的下表。每个数组元素实际是一个map结构，表示内存k/v结构。这么定义的原因是为了数据迁移方便处理，实际性能一般，毕竟只是一个示范性project。

然后，当配置文件变化导致shard迁移时，实际的操作是：不同replica group之间的shard数据传递，以及保证client请求发送到正确的replica group。因而，数据迁移可以简化为shards的迁移，其结构如下：

{% highlight Go %}
//请求结构体
type GetShardArgs struct {
	Shards	[]int         //期望获取的shard列表
	CfgNum	int           //期望的配置文件编号
}

type GetShardReply struct {
	WrongLeader bool
	Err 	Err
	Content	[shardmaster.NShards]map[string]string //shards数据以数组形式组织map
	TaskSeq map[int64]int                          
}
{% endhighlight %}

最后，当一个replica group从远程获取最新的shards数据后，如何修改本地的k/v数据呢？这里涉及一致性问题，后面我们将讨论，这里也需要一个结构体：

{% highlight Go %}
//请求结构体
type ReconfigureArgs struct {
	Cfg	shardmaster.Config
	Content [shardmaster.NShards]map[string]string
	TaskSeq map[int64]int
}

type ReconfigureReply struct {
	Err	Err
}
{% endhighlight %}

## 周期性检测配置变动
上篇文章谈到，master主要维护配置文件的变动，replica group根据配置文件的变动修改本地的k/v数据。因而，每一个replica group需要一个周期性任务时刻关注配置文件变化，并根据变化进行shards数据迁移。

{% highlight Go %}
func (kv *ShardKV) pollConfig(masters []*labrpc.ClientEnd) {
	var timeoutChan <-chan time.Time
	for true {
		if _, isLeader := kv.rf.GetState(); isLeader {
			newCfg := kv.sm.Query(-1)
			for i := kv.config.Num + 1; i <= newCfg.Num; i++ {
				if !kv.Reconfigure(kv.sm.Query(i)) {
					break
				}
			}
		}
		timeoutChan = time.After(100 * time.Millisecond)
		<-timeoutChan
	}
}
{% endhighlight %}

这里每隔100ms进行一次检测，如果本地配置文件的序号（Num)比配置中心最新配置文件旧，发起更新shards数据操作。实际的处理者是Reconfigure函数，这里需要注意并不是直接对齐最新配置文件，而是逐步更新至最新配置文件。

{% highlight Go %}
func (kv *ShardKV) Reconfigure(newCfg shardmaster.Config) bool {
	//创建配置修改，一致性协商请求包
	//这个包会将从远程获取的shards进行组装，然后发起本地的一致性协商，保证在正确的时间修改本地配置和k/v数据
	ret := ReconfigureArgs{Cfg:newCfg}
	ret.TaskSeq = make(map[int64]int)
	for i := 0; i < shardmaster.NShards; i++ {
		ret.Content[i] = make(map[string]string)
	}
	isOK := true
	//更加新的配置文件和本地配置文件的区别，diff出需要从远程获取的具体shards，以及这些shards当前属于的replica group
	mergeShards := make(map[int][]int)
	for i := 0; i < shardmaster.NShards; i++ {
		if newCfg.Shards[i] == kv.gid && kv.config.Shards[i] != kv.gid {
			gid := kv.config.Shards[i]
			if gid != 0 {
				if _, ok := mergeShards[gid]; !ok {
					mergeShards[gid] = []int{i}
				} else {
					mergeShards[gid] = append(mergeShards[gid], i)
				}
			}
		}
	}
	//遍历缺失的shards列表，发送rpc向其他replica group请求对应的数据
	var retMu sync.Mutex
	var wait sync.WaitGroup
	for gid, value := range mergeShards {
		wait.Add(1)
		//一次可能需要向多个replica group发送请求，因此采用异步方式
		go func(gid int, value []int) {
			defer wait.Done()
			var reply GetShardReply
			//创建rpc参数，发送请求包
			if kv.pullShard(gid, &GetShardArgs{CfgNum: newCfg.Num, Shards:value}, &reply) {
				retMu.Lock()
				for shardIdx, data := range reply.Content {
					for k, v := range data {
						//从远程获得的数据，暂时存入一致性协商结构体中
						ret.Content[shardIdx][k] = v
					}
				}
				//除了k/v数据，还需要传输请求编号信息，这个具体原因请参考以前文章解释
				for cli := range reply.TaskSeq {
					if seq, exist := ret.TaskSeq[cli]; !exist || seq < reply.TaskSeq[cli] {
						ret.TaskSeq[cli] = reply.TaskSeq[cli]
					}
				}
				retMu.Unlock()
			} else {
				isOK = false
			}
		} (gid, value)
	}
	wait.Wait()
	//发起修改本地配置文件和k/v内容的一致性协商
	return isOK && kv.SyncConfigure(ret)
}
{% endhighlight %}

这个函数主要做四件事情：

1. 创建一个暂存rpc通信返回数据的结构体，这个结构体当收到所有数据后发起修改本地数据的一致性协商。
2. 通过diff本地配置和修改后配置的内容，发现需要从远程获取哪些shards，以及这些shards当前归属的replica group。
3. 异步发送多个rpc请求，获取缺失的shards数据，以及请求编号数据等内容。
4. 数据全部返回后，发起修改本地数据的一致性协商请求，在合适的时机进行数据内容和配置修改。

关于第三点，对端如果响应rpc请求，可以看如下实现。过程很直接，接到请求把内容返回了，这里其实并没有保证一致性等要求；保证一致性其实是第四步进行相关的处理。

{% highlight Go %}
func (kv *ShardKV) GetShard(args *GetShardArgs, reply *GetShardReply) {
	kv.mu.Lock()
	defer kv.mu.Unlock()
	if kv.config.Num < args.CfgNum {
		reply.Err = ErrNotReady
		return
	}
	reply.Err = OK
	reply.TaskSeq = make(map[int64]int)
	for i := 0; i < shardmaster.NShards; i++ {
		reply.Content[i] = make(map[string]string)
	}
	for _, shardIdx := range args.Shards {
		for k, v := range kv.db[shardIdx] {
			reply.Content[shardIdx][k] = v
		}
	}
	for cli := range kv.taskSeq {
		reply.TaskSeq[cli] = kv.taskSeq[cli]
	}
}
{% endhighlight %}

为什么需要第四点，或者说整个数据迁移流程为什么这么复杂，接下来将进行说明。

## 本地数据修改的一致性协商

在真个分布式shards分片存储中，数据从远程迁移到本地其实不是难事。麻烦的地方在于，在shard数据迁移前后的用户请求应该如何响应？为了保证数据迁移，用户实际对k/v的操作直接的一致性，必须将因为配置变动导致k/v修改和本地配置文件的修改，与用户的操作通过raft进行一致性协商。这样才能保证一致性需求，这当然就是上一节第四点做的事情。我们来看看SyncConfigure实际功能：

{% highlight Go %}
func (kv *ShardKV) SyncConfigure(args ReconfigureArgs) bool {
idx, _, isLeader := kv.rf.Start(Op{Meth:"Reconfigure", ReCfg:args})
if !isLeader {
	return false
}

kv.muMsg.Lock()
ch, ok := kv.messages[idx]
if !ok {
	ch = make(chan Result, 1)
	kv.messages[idx] = ch
}
kv.muMsg.Unlock()

select {
case msg := <- ch:
	if ret, ok := msg.args.(ReconfigureArgs); !ok {
		return ret.Cfg.Num == args.Cfg.Num
	}
case <- time.After(150 * time.Millisecond):
	continue
}
}
return false
{% endhighlight %}

这里可以看到，所谓的SyncConfigure就是把更新k/v的需求，放入到raft中等待底层一致性算法发起协商。可以说reconfigure对本地状态的修改，与用户发起的修改请求处理说类似的。这样才能保证上面谈到的shards迁移，和用户操作本身直接不一致问题。那么实际shards迁移生效和配置修改在那儿生效呢，当然就是这条状态commit后啦！代码实现大家直接看[代码实现](https://github.com/hawkxiang/Distributed-System/tree/master/shardkv)

总结一下整体的shard分片处理流程：

1. 每个独立的replica group周期性与配置中心检测配置文件变动
2. diff配置文件，找出需要迁移的shards，发送RPC获取内容
3. 数据收到后，不能直接修改本地K/V，必须通过raft向自己发送一致性协商；避免与用户的读/写需要产生“冲突”

当然实际的实现过程中还有一些诸如“请求序号”传输等需求，请大家参考[代码实现](https://github.com/hawkxiang/Distributed-System/tree/master/shardkv)
