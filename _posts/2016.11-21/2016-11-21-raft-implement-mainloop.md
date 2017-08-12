---
layout: post
title: Raft一致性算法实现1——主循环和阻塞队列
author: hawker
permalink: /2016/11/raft-implemnt1.html
date: 2016-11-12 19:00:00
category:
    - 编程
tags:
    - distributed-system
    - algorithm
---
[上一片文章](http://www.hawkers.cc/2016/11/raft-algorithm.html)简单的说明了Raft的基本原理，当然推荐大家去读一读[paper](https://pdos.csail.mit.edu/6.824/papers/raft-extended.pdf)更加深入的理解一致性算法的设计。本篇博客开始，将简单介绍如何使用Golang实现Raft算法，并在其基础上实现基本的分布式k/v存储。本篇将从整体性介绍程序的结构，以及基本的事件循环等内容，希望大家对整体代码框架有一个宏观的认识。

## 日志与状态机
在分布式一致性算法或者事物中，常常借助日志系统来记录一台机器上执行的操作，通过保证集群中各台集群的日志内容最终一致性，确保状态机一致性。Raft一致性算法中亦是如此，每台机器独立维护自己的Log系统，当收到commit指令后才会执行Log中的命令，改变状态机的状态。因此，我们可以简单的把集群中的设备看成下图所示的结构。
![Alt text](/upload/2016/11/state_machine.png "Log&State Machine")
还记得在上一篇Raft原理性简文中谈到，Raft中的节点有三种状态：leader,follower,candidator。这三种状态主要是为了实现各台机器中的日志最终一致性，对应图中每台节点下部。Raft算法中也谈到针对一条日志，当leader收到大部分回复后，将会提交该条状态，对应图中每台节点上部。


## 整体代码结构
针对上文谈到的日志系统和状态机，本文的代码主框架如下所示：一个主Goroutine专门进行事件循环（eventloop)根据节点角色变化，进行对应的一致性算法业务处理；另一个主Goroutine不断执行commit状态的日志命令，改变状态机内容。

{% highlight Go linenos%}
	//初始化
	rf := makeServer(peers, me, persister)
	if err := rf.Init(); err != nil {
		return nil
	}

	rf.stopped = make(chan bool)
	rf.ChangeState(Follower)
	rf.applyCh = applyCh

	//Goroutine1: enter event_loop
	rf.wg.Add(1)
	go func() {
		defer rf.wg.Done()
		rf.loop()
	}()

	//Goroutine2: apply Log to state machine
	rf.wg.Add(1)
	go func() {
		defer rf.wg.Done()
		rf.applyLoop()
	}()

	return rf
{% endhighlight %}

忽略开始阶段的相关初始化代码，可以看到主函数包含两个Goroutine，作用如上文所述。这样的好处：一，主函数不会阻塞，快速反馈用户初始化是否正确；二，分离代码业务功能，整体结构更加清晰。

首先，我们来看Goroutine1中事件循环的实际逻辑。显然，本文根据Raft算法中三种角色变换，切换事件循环业务处理。本文，着重介绍代码的主体结构，因此followerLoop、candidateLoop和leaderLoop具体的业务代码放在后续文章进行介绍。

{% highlight Go linenos%}
	func (rf *Raft) loop() {
	for rf.state != Stopped {
		switch rf.state {
		case Follower:
			rf.followerLoop()
		case Candidate:
			rf.candidateLoop()
		case Leader:
			rf.leaderLoop()
			}
		}
	}
{% endhighlight %}

谈完了整体的事件循环，我们在看一看Goroutine2中执行提交Log的代码实现。这段代码首先使用copy_on_write技术，将当前内存中的Log_List拷贝一份，快速释放锁。然后，遍历已经提交的Log并且包装成ApplyMsg进行提交。你可能要问为什么要copy_on_wirte，不是浪费很多内存嘛？其实，这里也是一种权衡。要知道，rf.Log在多个Goroutine中使用肯定需要锁，Goroutine1里的事件循环不断的对rf.Log进行增加/删除。如果，Goroutine2在遍历日志，提交过程中加锁，Goroutine1中的事件循环会被阻塞很久。而且，Goroutine2提交很耗时，而且可能阻塞很久。所以，权衡使用copy_on_write去避免Goroutine1里的事件循环阻塞。

{% highlight Go linenos%}
	func (rf *Raft) applyLoop() {
	for rf.state != Stopped {
		select {
		case <-rf.applyNotice:
			//copy on write........................
			var applyLog []Entry
			commitidx := -1
			rf.mu.Lock()
			if rf.lastApplied >= rf.BaseIndex() {
				applyLog = rf.Log
				commitidx = rf.commitIndex
			}
			rf.mu.Unlock()
			if applyLog != nil{
				for commitidx > rf.lastApplied {
					rf.lastApplied++
					rf.applyCh <- ApplyMsg{rf.lastApplied, 	 applyLog[rf.lastApplied-applyLog[0].Index].Command, false, nil}
				}
			}
		}
	}
}
{% endhighlight %}


## 阻塞队列与RPC响应

目前，我们还没有谈到用户一条命令到来后，处理的工作流程；以及机器之间消息交互的处理流程。在介绍处理流程之前，我先抛出处理流程实现所存在的一些难点问题，希望与大家一起思考。**主要问题：如何保证每台机器上接受到的消息，按照正确顺序进行处理？分布式系统中没有统一的时钟，而且每台机器同时监听多个不同消息的到达，每条消息处理的耗时也不同。我们需要保证消息处理完的顺序，应该与消息到达顺序相同；同时还要保证用户等待尽量短暂。**

1. 使用时间信息保证顺序性，很明显这是不可行的做法。
2. 在每个等待外部请求的handle处加锁，互斥处理每个请求。这个方案理论上没问题，但是代码整体会有很多需要加锁和解锁的地方，非常难以维护，严格的加锁也会存在性能问题。
3. Blocking Queue：我是一个懒惰的人，因此不想思考复杂的加锁带来的同步处理。因此，采取阻塞队列的方式将多种消息请求，包装成一种通用的任务，并且将任务放入队列，等待事件循环顺序性的消费阻塞队列中的任务。处理逻辑如下图所示。

![Alt text](/upload/2016/11/blocking.png "Blocking_Queue")

如图所示，RCP等待消息请求的到达，收到请求后使用deliver()包装成统一的任务放入阻塞队列，事件循环不断的消费任务，并返回最终结果。等待消息请求，请求入队列和消费请求三步简单明了的解决以上描述的问题。不用，考虑复杂的加锁和释放问题，请求到达也不会被阻塞甚至丢弃。接下来举例说明这三步的代码实现。

### RCP通信

Go语言提供了RPC通信支持，简单来看Raft实现中RPC的发送与影响接口。如下代码所示，重点关注RPC响应接口，使用了deliver函数保证了请求。每台机器，每个RPC可能同时接受多条请求，因此加锁不是一个好的处理方式；此外，每种角色需要同时监听多种RPC请求，这里限于篇幅仅仅例举了一种。

{% highlight Go linenos%}
//RPC请求
func (rf *Raft) sendAppendEntries(server int, args AppendEntriesArgs, reply *AppendEntriesReply) bool {
	ok := rf.peers[server].Call("Raft.AppendEntries", args, reply)
	return ok
}
//RPC响应
func (rf *Raft) AppendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) {
	ok := rf.deliver(&args, reply)
	if ok != nil {
		reply = nil
	}
}
{% endhighlight %}

ok，看完RPC处理逻辑，再来看看重点deliver函数的实现。注意参数形式，是interface{}接口形式；刚刚说了有多种RPC类型，因此我们需要用一种通用的类型传递这些请求消息，interface{}符合要求。

{% highlight Go linenos%}
	func (rf *Raft) deliver(value interface{}, r interface{}) error {
	task := &message{args: value, reply: r, err: make(chan error, 1)}

	//deliver the task to event loop
	select {
	case rf.blockqueue <- task:
	case <-rf.stopped:
		return StopError
	}
	// wait for the task been handle over, and return
	select {
	case <-rf.stopped:
		return StopError
	case suc := <-task.err:
		return suc
	}
}
{% endhighlight %}

注意函数中的两个select，他们的作用类似c中的（poll）同时等待多个channel，避免互相阻塞。第一select是为了将任务放入阻塞队列，同时监听期间是否有系统发出的退出指令；第二select是等待任务处理后的返回值和退出指令。

第三步，事件循环消费任务放在下一篇进行接受，结合事件循环整体进行说明。最后，抛出一个问题：上文说了将所有不同类型的消息包装成没有类型的interface{}，那么事件循环消费任务时，如何识别类型，使用正确的handle进行处理呢？
