---
layout: post
title: Raft一致性算法原理与实现——主循环和阻塞队列
author: hawker
permalink: /2016/11/raft-state.html
date: 2016-11-25 14:00:00
category:
    - 编程
tags:
    - distributed-system
    - raft
    - algorithm
---
前面的文章说明了raft的基本原理，以及如何用Golang实现raft分布式一致性协议。本篇我们将通过一个K/V的例子，向大家展示如何基于raft协议实现分布式存储。这种需要在工业上应用非常广泛，通过这个基本的分布式存储例子，希望大家对一致性算法在实际应用方面有更加深刻的认识。

## C/S模式的分布式存储
进入具体代码实现之前，简单介绍分布式存储示例工作方式，主要分为两个模块，遵循传统的客户端、服务器C/S模式。客户端模拟用户向分布式存储系统发起指令：读、写。服务端主要用来接收用户发来的指令，如果是leader身份开始处理用户请求，否则反馈用户非leader身份。leader收到用户请求后，会发起一致性协商，最终实现分布式存在的一致性，并将结果返回给用户。这是一致性算法与分布式存储运用的基本示例，通过它可以形象的了解分布式一致性的工作形式。


## 客户端实现
我们来看一看分布式键值存储客户端的实现，首先是基本的初始化流程。指定上一次成功通信的leader编号，当然这个编号如果在本次通信中失败，会替换为当前可用的leader编号。客户端随机生成的身份标识符、以及发出的指令编号。

{% highlight Go linenos%}
func MakeClerk(servers []*labrpc.ClientEnd) *Clerk {
	//初始化
	ck := new(Clerk)
	ck.servers = servers
	ck.lastLeader = 0         //上一次可用leader
	ck.identity = nrand()     //client自身标识符
	ck.seq = 0				  //命令编码
	return ck
}
{% endhighlight %}

客户端的初始化比较简单，接着看一下客户发出指令的方式，以添加一条记录为例：

{% highlight Go linenos%}
func (ck *Clerk) PutAppend(key string, value string, op string) {
	args := PutAppendArgs{Key: key, Value: value, Op: op, Client: ck.identity}
	ck.mu.Lock()
	args.Seq = ck.seq
	ck.seq++
	ck.mu.Unlock()
	for i, n := ck.lastLeader, len(ck.servers); ; i = (i + 1) % n {
		var reply PutAppendReply
		ok := ck.servers[i].Call("RaftKV.PutAppend", &args, &reply)
		if ok && !reply.WrongLeader {
			ck.lastLeader = i
			return
		}
	}
}
{% endhighlight %}

代码逻辑很清晰，将请求的Key/Value包装成一个结构体，原子操作递增指令序号，向leader发送RPC指令。循环主要是为了找到当前可用的leader，并且会将该leader的身份记录下来，以便下一次可以快速访问成功。Ok，这里就是client端基本的逻辑。

## 服务端实现

记得在这一系列文章中的[第二篇](http://www.hawkers.cc/2016/11/raft-implemnt.html)中谈到：可以将一台机器分为状态机和一致性日志两部分。用户的指令主要期望获取状态或者修改状态，Raft算法利用日志保证集群中多台机器的一致性状态。因此，服务端(leader)的逻辑也相对清晰：接受客户端的RPC指令，根据指令发起Raft一致性协商，等待状态提交后获取返回值，交付用户。

{% highlight Go linenos%}
func (kv *RaftKV) PutAppend(args *PutAppendArgs, reply *PutAppendReply) {
	command := Op{Meth: args.Op, Key: args.Key, Value: args.Value, Client: args.Client, Seq: args.Seq}
	if ok := kv.DuplicateLog(command); !ok {
		reply.WrongLeader = true
	} else {
		reply.WrongLeader = false
		reply.Err = OK
	}
}
{% endhighlight %}

接收用户指令的handle中，将指令包装成Raft通用的结构体，通过DuplicateLog函数发起一致性协商。DuplicateLog通过Start函数发起Raft协商，回忆Raft算法的实现中所介绍的内容，Start会返回机器的身份以及如果命名被正确协商执行，所处的日志索引。如果该机器是leader，会将指令包装成通用的interface任务，放入Blocking Queue等待事件循环进行处理。DuplicateLog函数中kv.reflect是一个[日志索引]-->[channel]的map，主要用来接收提交后的状态返回值。该函数，最后是等待kv.reflect中的返回值，或者超时(600ms)。

{% highlight Go %}
func (kv *RaftKV) DuplicateLog(entry Op) bool {
	idx, _, isLeader := kv.rf.Start(entry)
	if !isLeader {
		return false
	}

	kv.mu.Lock()
	//kv.reflect: map[int]chan Op
	ch, ok := kv.reflect[idx]
	if !ok {
		ch = make(chan Op, 1)
		kv.reflect[idx] = ch
	}
	kv.mu.Unlock()
	
	//wait to commit
	select {
	case op := <-ch:
		return op == entry
	case <-time.After(600 * time.Millisecond):
		return false
	}
}
{% endhighlight %}

那么Raft提交状态后，服务端如何将内容填充到kv.reflect返回channel中呢。在服务端初始化时，会启动一个Goroutine异步的从Raft读取提交的状态。其功能如下：

{% highlight Go %}
func (kv *RaftKV) loop(maxraftstate int, persister *raft.Persister) {
	for entry := range kv.applyCh {
		command := entry.Command.(Op)
		kv.mapmu.Lock()
		//apply change task to state machine
		if command.Meth != "Get" && !kv.StaleTask(command.Client, command.Seq){
				switch command.Meth {
				case "Put":
					kv.db[command.Key] = command.Value
				case "Append":
					kv.db[command.Key] += command.Value
				}
				kv.chk[command.Client] = command.Seq
		}
		kv.mapmu.Unlock()

		kv.mu.Lock()
		//double check
		ch, ok := kv.reflect[entry.Index]
		if ok {
			select {
				//drain bad data
				case <-kv.reflect[entry.Index]:
				default:
			}
			ch <- command
		}
		kv.mu.Unlock()
}
{% endhighlight %}

我们在介绍Raft算法实现时谈到，会用一个独立的Goroutine不断的将commit日志写入一个channel，实际应用这些指令到状态机。这里的loop就是从channel读取提交的日志命令，并且根据命名类型修改机器中的状态。修改状态后，通过kv.reflect将修改后的状态值返回给对应的用户。注意，这里kv.reflect[entry.Index]在写入返回状态前理论上应该为空，为了保证这一点使用select首先排空可能存在的“bad value”，default语句保证kv.reflect[entry.Index]确实为空时不会阻塞，马上结束select。最后，将修改后的状态写入channel，上文中DuplicateLog的读取操作得到响应，返回用户结果。

ok，这就是一个构建在Raft算法之上的分布式K/V存储系统，下一节我将介绍如何使用快照（snapshot）保持内存状态，避免Raft机器每次重启长时间的Log载入过程。

