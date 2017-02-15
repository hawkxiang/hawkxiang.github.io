---
layout: post
title: Raft一致性算法原理
author: hawker
permalink: /2016/11/raft-algorithm.html
date: 2016-11-17 18:00:00
category:
    - 编程
tags:
    - distributed-system
    - algorithm
---
一致性问题一直是分布式系统的核心问题，Raft针对该问题设计了基于日志的可容错的一致性算法。本文将详细说明Raft一致性算法的原理。[参考论文](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)

## 分布式系统的特性
由于空间分割、网络不稳定性、机器设备不可靠性，与集中式系统不同分布式系统有如下几个特性：
1. 分布性：多台普通机器在空间上随意分布，同时设备的分布情况也会随时改变。
2. 并发性：分布式网络中程序运行过程中的并发性操作是非常常见的行为，准确高效地协调分布式并发操作是整个分布式系统架构设计过程中主要挑战之一。
3. 缺乏全局时钟：分布式系统中由于缺乏一个全局的时钟系列控制，很难定义两个事件究竟谁先谁后。
4. 高故障率：由于分布式系统设备是一般的电脑，并且数量较大。故而设备的硬件、软件系统、网络等故障总是在不断发生。保持系统的可用性、稳定性、一致性、高性能是分布性系统主要问题。
google的一篇[分布系统介绍](http://www.hpcs.cs.tsukuba.ac.jp/~tatebe/lecture/h23/dsys/dsd-tutorial.html)的文章，对此有比较详细的介绍。


目前，解决分布式系统一致性问题的方法主要有：二阶段提交(2PC)、三阶段提交(3PC)、一致性算法(Paxos, Raft)等。


## 晦涩的Paxos
Paxos算法是莱斯利·兰伯特于1990年提出的一种基于消息传递的一致性算法。初期Lamport用比较诙谐的语言描述Paxos，但会议审稿者一致无法接收Lamport的幽默感，Lamport又于2001年用学术化的语言重新描述了Paxos，也算是Paxos一段趣闻。Chubby是google的一个面向松耦合分布系统的锁服务，它使用Paxos作为一致性算法。但是Paxos的主要问题是晦涩难以理解，同时难以程序实现。初学者需要花费大量时间去弄懂“Basic Paxos”，“Multi Paxos”更是令人云里雾里。最近，微信开源了一套C++实现的Paxos算法，感兴趣的朋友可以去了解一下。

与Paxos不同，Raft算法在设计之初就强调可理解性（Understandable），Understandable也是Raft设计过程中主要准则。

## Raft算法中服务器状态

Raft算法中将集群中的服务器分为3种角色：

1. Leader：Client向Leader发送请求，Leader将需要备份的日志信息广播给集群中其他Servers，同一个Terms集群中只有一个Leader。同时，Leader会周期性的广播心跳包，告诉其他Follower当前的Leader正常工作，重置其他Follower的选举时钟。
2. Follower: 接收Leader或者Candidate发送的RPC消息，并进行相应的处理。选举时钟超时后，将切换为Candidate身份。新加入的Server都是Follower状态。Follower长时间收不到Leader的心跳包，选举时钟超时，自己变成Candidate开启选举。
3. Candidate：由Follower超时转换而来的中间状态，获得集群中大部分投票后转变为Leader。


下图为Raft中Server的状态转换图：

![Alt text](/upload/2016/11/raftstate.png "Raft_State")

## Raft中的Terms
Raft中用Term的概念来表示一个工作周期。众所周知，分布式环境下缺乏统一的全局时钟，“时间同步”本身是一个很大的难题。同时，分布式系统中由网络的不可靠性存在大量的过期消息，为了识别过期消息，时间信息又是必不可少的。Raft针对这个问题，将集群的工作时间切分为一个个的Term，相当于一个周期这样的“时间概念”。如下图所示：

![Alt text](/upload/2016/11/term.png "Term")

Term开始于选举，终止与下一次选举，对于每一个节点而已Term都是单调递增的。

1. 一般情况下每个Term都要一个Leader，且只有一个Leader。
2. 当然，某些Term可能不存Leader，此时可能是该Term内的选举失败，超时进入下一个Term选举所导致的的。
3. 每个Server收到的RPC消息中都带有Term标识，如果收到的消息中Term大于本地的currentTerm，server将将currentTerm设为消息中的Term，并将自身的状态切换为Follower。

## 选举

选举时钟超时，Server的状态从Follower（或Candidate）切换为Candidate。自增本地的currentTerm，并将VoteFor设置为自身，重置选举定时器，向集群中的其他机器发起RequestVote RPC, 直到出现以下条件：

1. 获得集群中大部分参与者的投票，一般是半数以上，保证一个Term中选举仅有一个Leader产生。此时，状态转换为Leader，广播Heartbeats给其他Servers，终止他们的选举。
2. 接收到合法的AppendEntries RPC，即消息的Term大于等于自身，转换为Follower状态。
3. 选举超时，自增currentTerm进入下一个Term的选举。这种情况主要是由于集群中两个candidate同时发起选举（split vote）,导致两个candidate都无法获得足够的选票所致。所以，每次选举开始进行重置选举定时器时，时间都是随机值（150ms-300ms），这样保证下一次选举两个冲突的candidate不会再次同时开始选举。
4. Follower收到一个RequestVote RPC时，首先查看其Term是否大于本地的currentTerm（合法性）。然后检查RequestVote消息中携带的对方日志LastIndex和LastTerm是否比本地的日志更新。满足这些条件，Follower才返回投票，否则拒绝投票

## 日志备份与协商

分布式系统能够保证可靠性和容错的主要原因是：利用日志顺序记录改变State Machine的相关命令，并保持集群中的所有servers中的Logs都是一致的。下图是Raft添加日志和协商实现集群中Servers日志一致的过程：

![Alt text](/upload/2016/11/log.png "Append")

首先，请大家记住Raft一下几个属性，他们保证了Raft在日志同步，或者增加过程中始终能够保持一致。

1. Election Safety: at most one leader can be elected in a given term.(一个Term内，最多选举出一个Leader)
2. Leader Append-Only: a leader never overwrites or deletes entries in its log; it only appends new entries.(Leader本身不会重写或者删除自己本地的日志，仅仅会进行增加操作)
3. Log Matching: if two logs contain an entry with the same index and term, then the logs are identical in all entries up through the given index.（如果两台Server中的日志，相同的Index对应的日志项Term相同，那么两个日志Index前面的日志项都相同）
4. Leader Completeness: if a log entry is committed in a given term, then that entry will be present in the logs of the leaders for all higher-numbered terms.（如果某个Server提交了一个日志，即在执行了日志中对应的操作。其他的servers在对应的Index将会执行相同的日志项）
5. State Machine Safety: if a server has applied a log entry at a given index to its state machine, no other server will ever apply a different log entry for the same index.

为什么有了以上5条就能保持日志的一致性，请大家阅读Raft原文。证明比较长，而且担心我的描述会引起您理解歧义。故而推荐阅读原文。

当Leader收到来自Client的请求时，他将通过以下流程进行处理。

Leader:

1. Leader将日志Append到本地。
2. 发送AppendEntriesRPC给Followers，将日志发给它们。（这里需要使用nextIndex[i]获取每个Follower需要发送的日志项）
3. 如果超过大多数的Followers回复了Leader，则Leader可以提交这条日志。（不用等到所以Followers反馈，提高系统整体性能）
4. 如果Follower返回false，要么当前的Leader失效，需要转换为Follower状态；要么Leader发送的Log下表与Follower不匹配，这时需要递减nextIndex[i]直到遇到一个二者日志匹配的Index。

Follower:

1. 接收AppendEntriesRPC消息，如果消息本身携带的Term小于Follower节点的CurrentTerm返回false。
2. 如果AppendEntriesRPC消息携带的日志Index及日志Term，能在Follower本地日志Index处找到的日志项Term相同，则将收到的日志加入Follower本地日志中，否则返回false。

当然，以上描述的内容都是基本的日志添加处理流程。由于设备的重启、网络故障引起的一些极端的日志协商情况，需要一些特别的处理方法，我们在此不在详述。

## 日志压缩
随着系统的持续运转，节点中的日志信息不断膨胀，影响节点恢复时状态重放的效率，同时也制约日志查找性能。因此，日志压缩是比不可少的功能。分布式系统中有多种压缩策略，Raft采用了快照（Snapshot）的形式进行日志压缩。当上层应用apply日志时，会侦测到节点日志的长度，如果太大向Raft发送日志压缩命令。Raft在本地执行日志压缩，将已经提交并且Apply的日志截断之前的状态(state machine）作成快照，保留快照后面的日志信息。基本情况如下图：

![Alt text](/upload/2016/11/snapshot.png "Snapshot")

快照一方面有效的节省了存储空间，另一方面当新加入的节点状态落后Leader很多时，一次快照恢复即可快速追上当前Leader的状态，提高系统整体恢复性能。

## 其他问题

我在这里仅仅谈了一些Raft算法中比较核心的部分，还有诸如集群配置动态变化处理、客户端与Raft交互等内容没有介绍。感兴趣的朋友应该去阅读原文，系统的掌握Raft算法得工作流程。接下来的文章我将说明如何用Go语言实现一个Raft算法，并基于Raft算法实现一个分布式可容错的键值存储系统。
