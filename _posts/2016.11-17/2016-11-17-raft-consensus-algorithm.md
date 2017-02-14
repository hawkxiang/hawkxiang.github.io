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
