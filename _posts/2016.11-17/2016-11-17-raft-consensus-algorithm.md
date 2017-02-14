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

1. Leader：Client向Leader发送请求，Leader将需要备份的日志信息广播给集群中其他Servers，同一个Terms集群中只有一个Leader。
2. Follower: 接收Leader或者Candidate发送的RPC消息，并进行相应的处理。选举时钟超时后，将切换为Candidate身份。新加入的Server都是Follower状态。
3. Candidate：由Follower超时转换而来的中间状态，获得集群中大部分投票后转变为Leader。


下图为Raft中Server的状态转换图：

<center>![Alt text](/upload/2016/11/raftstate.png "Raft_State")</center>
