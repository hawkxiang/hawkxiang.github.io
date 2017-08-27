---
layout: post
title: Raft一致性算法原理与实现————日志压缩快照技术
author: hawker
permalink: /2016/11/raft-snapshot.html
date: 2016-11-29 17:00:00
category:
    - 编程
tags:
    - distributed-system
    - raft
    - algorithm
---
[上篇文章](http://www.hawkers.cc/2016/11/raft-state.html)谈到了使用Raft一致性协议实现分布式K/V存储。承接上文，本篇谈一谈Raft中的日志压缩与快照(snapshot)实现。

## Log Compaction 
在一个实际的分布式存储系统中，不可能让节点中的日志无限增加。冗长的日志导致系统重启时需要花费很长的时间进行回放，影响系统整体可用性。Raft与Chubby、Zookeeper等类似，都采用了snapshot技术进行日志压缩，丢弃snapshot之前的日志项目。

![Alt text](/upload/2016/11/state_snapshot.jpg "SnapShot Log Compaction")

Raft中每个节点独立的对自己的系统状态进行snapshot操作，当然只能对已经committed日志项(已经apply到了状态机)进行snapshot。snapshot有一些元数据，包括last_included_index，即snapshot覆盖的最后一条committed日志项的index，以及last_included_term，即这条日志的termid。这两个值在snapshot之后的第一条log entry的AppendEntriesRPC的consistency check的时候会被用上。一旦这个节点做完了snapshot，就可以把这条日志及之前的日志项目删除，压缩日志长度。

snapshot的缺点就是不是增量的，即使内存中某个值没有变，下次做snapshot的时候同样会被dump到磁盘。

当leader需要发给某个follower的log entry被丢弃了(因为leader做了snapshot)，leader会将snapshot发给落后太多的follower。或者当新加进一台机器时，也会发送snapshot给它。

做snapshot有一些需要注意的性能点，1. 不要做太频繁，否则消耗磁盘带宽。 2. 不要做的太不频繁，否则一旦节点重启需要回放大量日志，影响availability。系统推荐当日志达到某个固定的大小做一次snapshot。3. 做一次snapshot可能耗时过长，会影响正常log entry的replicate。这个可以通过使用copy-on-write的技术来避免snapshot过程影响正常log entry的replicate。


## Snapshot实现

snapshot技术主要解决日志太长的问题，因而当server节点的日志长度超过阈值时启动快照技术。参考以下代码，首先检查是否启动snapshot功能以及节点日志长度，满足条件进行日志压缩与状态保存。序列化节点目前的状态信息，启动新的goroutine传入状态信息、压缩到的日志下标、等恢复信息进行shnapshot处理。

{% highlight Go linenos%}
if maxraftstate != -1 && persister.RaftStateSize() > maxraftstate {
  recover := maxraftstate
  maxraftstate = -1
  
  w := new(bytes.Buffer)
  e := gob.NewEncoder(w)
  e.Encode(kv.db)
  e.Encode(kv.chk)
  data := w.Bytes()
  
  go func(snapstate []byte, preindex int, maxraftstate *int, recover int){
      kv.rf.TakeSnatshot(snapstate, preindex)
      *maxraftstate = recover
  }(data, entry.Index, &maxraftstate, recover)
}
{% endhighlight %}

实际的shnapshot是由TakeSnatshot函数实现，其逻辑如下方代码所示。首先，检查传入的日志压缩下表是否合法；然后，序列化日志压缩处的index等上下文信息，并append传入的状态信息，持久化快照信息；最后，截断日志项，压缩日志尺寸。

{% highlight Go%}
func (rf *Raft) TakeSnatshot(snapstate []byte, preindex int) {
	rf.mu.Lock()
	defer rf.mu.Unlock()
	if preindex <= rf.BaseIndex() || preindex > rf.lastApplied {
		return
	}
	//snapshot
	w := new(bytes.Buffer)
	e := gob.NewEncoder(w)
	//meta
	e.Encode(preindex)
	e.Encode(rf.Log[preindex-rf.BaseIndex()].Term)
	data := w.Bytes()
	data = append(data, snapstate...)
	rf.persister.SaveSnapshot(data)
	//compaction, drop rf.Log through preindex, garbage collection
	//index 0 is guard, eliminate slice index out range
	rf.Log = rf.Log[preindex-rf.BaseIndex():]
	rf.persist()
}
{% endhighlight %}

可以看到上面是日志压缩的处理过程，那么如何将快照和raft的一致性结合在一起呢？另外，当系统重启时，压缩后的快照状体信息如何回放到状态机中呢？带着这两个问题，来看看加入snapshot后Raft一致性协议实现。整个过程都进行了加锁，存在性能问题，可以考虑COPY ON WRITE提高性能。

## Raft快照处理

当添加snapshot压缩功能后，leader发起一致性协商时存在以下情况：leader需要发送给followers的日志内容已经被压缩，因此只能通过下面的代码RPC形式，将leader本身的快照状态发送给followers。

{% highlight Go linenos%}
var args SnatshotArgs
args.Term = rf.CurrentTerm
args.LeaderId = rf.me
args.Data = rf.persister.ReadSnapshot()
args.LastIncludedIndex, args.LastIncludedTerm = rf.readMeta(args.Data)
go func() {
    r := new(SnatshotReply)
    ok := rf.sendInstallSnapshot(server, args, r)
    if ok {
    	snapChan <- r
    }
}()
{% endhighlight %}

Followers端响应该RPC请求的处理函数是InstallSnapshot，其逻辑符合Raft基本的设计逻辑：将任务放入阻塞队列，等待事件循环消费处理。

{% highlight Go linenos%}
func (rf *Raft) InstallSnapshot(args SnatshotArgs, reply *SnatshotReply) {
	ok := rf.deliver(&args, reply)
	if ok != nil {
		reply = nil
	}
}
{% endhighlight %}

Followers的事件循环中实际处理Snatshot任务的handle如下：首先，检查RPC传来的参数的合法性；然后，用传输过来的快照替换本地的快照，并从截断内存中的日志，进行日志压缩。新的日志list第一项会放一个无效日志作为哨兵（guard），方便进行安全检查。这里没有使用copy on write的策略，一方面是我没有想到好的策略，另一方面是关键代码段区域较短，耗时的操作移出了加锁范围。

{% highlight Go%}
func (rf *Raft) handleInstallSnapshot(args *SnatshotArgs) (SnatshotReply, bool) {
	if args.Term < rf.CurrentTerm {
		return SnatshotReply{Term: rf.CurrentTerm, PeerId: rf.me, LastInclude: 0}, false
	}
	if args.Term > rf.CurrentTerm {
		rf.updateCurrentTerm(args.Term, args.LeaderId)
	} else {
		rf.VotedFor = args.LeaderId
	}
	
	//snapshot
	rf.persister.SaveSnapshot(args.Data)
	//compaction, drop rf.Log through preindex, garbage collection
	rf.mu.Lock()
	var newLog []Entry
	//rf.Log always has a guard
	newLog = append(newLog, Entry{args.LastIncludedIndex, args.LastIncludedTerm, nil})
	for i := len(rf.Log)-1; i >= 0; i-- {
		if rf.Log[i].Index == args.LastIncludedIndex && rf.Log[i].Term == args.LastIncludedTerm {
			newLog = append(newLog, rf.Log[i+1:]...)
			break
		}
	}
	rf.Log = newLog
	reply := SnatshotReply{Term: rf.CurrentTerm, PeerId: rf.me, LastInclude: rf.LastIndex()}
	rf.persist()
	rf.mu.Unlock()

	rf.commitIndex = args.LastIncludedIndex
	rf.lastApplied = args.LastIncludedIndex

	rf.applyCh <- ApplyMsg{UseSnapshot: true, Snapshot: args.Data}
	return reply, true
}
{% endhighlight %}

Followers正确处理快照RPC后回复leader，leader收到响应后的处理流程在此不再赘述。最后，我们来看看机器如何使用快照中的状态信息，进行回放。上面的handleInstallSnapshot在正确保存leader发送来的快照后，会将快照通过channel发送给本节点的状态机。K/V分布式存储的server端事件循环检查channel中的消息，如果是快照消息，使用readSnatshot解析消息，更改状态机状态。

{% highlight Go %}
if entry.UseSnapshot {
	kv.readSnatshot(entry.Snapshot)
} 
{% endhighlight %}

{% highlight Go linenos%}
func (kv *RaftKV) readSnatshot(data []byte) {
	var lastIncludeIndex, lastIncludeTerm int

	r := bytes.NewBuffer(data)
	d := gob.NewDecoder(r)
	d.Decode(&lastIncludeIndex)
	d.Decode(&lastIncludeTerm)
	kv.mapmu.Lock()
	d.Decode(&kv.db)
	d.Decode(&kv.chk)
	kv.mapmu.Unlock()
}
{% endhighlight %}

日志压缩与快照就介绍到这里，相关处理流程请参考[源码实现](https://github.com/hawkxiang/Distributed-System/blob/master/kvraft/server.go)。
