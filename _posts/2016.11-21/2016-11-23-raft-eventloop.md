---
layout: post
title: Raft一致性算法实现2——事件循环
author: hawker
permalink: /2016/11/raft-eventloop.html
date: 2016-11-23 16:00:00
category:
    - 编程
tags:
    - distributed-system
    - algorithm
---
本文继续介绍Raft实现，[上一片文章](http://www.hawkers.cc/2016/11/raft-implemnt1.html)谈到了整体架构以及与任务相关的阻塞队列。针对上篇文章遗留的问题，本文着重介绍事件循环如何正确消费阻塞队列中的任务，以及Raft算法中节点处在三种不同角色时如何处理事件循环。

## 任务类型断言机制
上一篇文章谈到阻塞队列使用interface{}保存不同类型的消息任务，但是后端的事件循环在消费任务时需要根据消息类型进行处理，需要进行类型转化。GO语言和其他编程语言类似，强制类型转化可能造成异常，因此需要使用语言本身提供的类型断言机制和Switch实现安全的类型转化，和任务处理。ok，直接来看leader角色下的事件循环中如何进行类型判断和任务处理。

{% highlight Go%}
	func (rf *Raft) leaderLoop() {
    //初始化
	heartbeats := true
	commitSync := make(map[int]bool)
	var timeoutChan <-chan time.Time
	
	respChan := make(chan *AppendEntriesReply, 64)
	snapChan := make(chan *SnatshotReply, 32)

	for rf.state == Leader {
		if heartbeats {
			rf.mu.Lock()
			for i := 0; i < len(rf.peers); i++ {
				if i != rf.me {
					rf.boatcastAppend(i, respChan, snapChan)
				}
			}
			rf.mu.Unlock()
			timeoutChan = random(HeartbeatInterval, HeartbeatInterval)
			heartbeats = false
		}

		select {
		case <-rf.stopped:
			rf.ChangeState(Stopped)
			return

		case r := <-snapChan:
			if success := rf.handleSnapshotResponse(r); success {
				rf.leaderCommit(commitSync, r.PeerId)
			}

		case reply := <-respChan:
			if success := rf.handleResponseAppend(reply, respChan, snapChan); success {
				rf.leaderCommit(commitSync, reply.PeerId)
			}

		case e := <-rf.blockqueue:
			var err error
            //Switch结合类型断言机制
			switch req := e.args.(type) {
			case *AgreementArgs:
				rp, _ := (e.reply).(*AgreementRely)
				*rp = rf.handleRequestAgreement(req, respChan, snapChan)
				
				commitSync[rf.me] = true
				timeoutChan = random(HeartbeatInterval, HeartbeatInterval)
				heartbeats = false
                
			case *RequestVoteArgs:
				rp, _ := (e.reply).(*RequestVoteReply)
				*rp, _ = rf.handleRequestVote(req)
			
			case *AppendEntriesArgs:
				rp, _ := (e.reply).(*AppendEntriesReply)
				*rp, _ = rf.handleAppendEntries(req)
			}

			e.err <- err

		case <-timeoutChan:
			heartbeats = true
		}
	}
}
{% endhighlight %}

代码很长，首先看一下case e := <-rf.blockqueue:这行以下的部分。这里使用了Go语言的类型断言机制和Switch功能，先判断是否可以将变量转发为特定类型，如果返回true，在switch对应的case中进行实际的类型转换。这里也可以看到，事件循环会不断从阻塞队列对应的channel中读取任务，根据断言后的任务类型选择事件的handle
