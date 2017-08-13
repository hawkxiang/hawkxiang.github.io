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

{% highlight Go %}
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

代码很长，首先看一下case e := <-rf.blockqueue:这行以下的部分。这里使用了Go语言的类型断言机制和Switch功能，先判断是否可以将变量转发为特定类型，如果返回true，在switch对应的case中进行实际的类型转换。这里也可以看到，事件循环会不断从阻塞队列对应的channel中读取任务，根据断言后的任务类型选择事件的handle处理任何，最后返回结果，这些与之前的整体代码框架相同。


## Leader状态下的事件循环
上面的代码是上一篇文章谈到的事件循环中，节点leader状态所对应的循环处理逻辑。本文通过这个循环说明raft实现中处理请求的流程。当然，分别还有candidator、follower状态的循环，限于篇幅在此不逐一说明。分析这一段代码，首先是节点成为leader后的变量初始化。

{% highlight Go linenos%}
//初始化
heartbeats := true
commitSync := make(map[int]bool)
var timeoutChan <-chan time.Time

respChan := make(chan *AppendEntriesReply, 64)
snapChan := make(chan *SnatshotReply, 32)
{% endhighlight %}

**初始化:**“是否发送心跳包flag”的heartbeats、“收集follower反馈状态”的map、“心跳包发送时钟”timeoutChan和两个“收集followers返回ID”的channels。接下来是循环模块，可以看到每次循环都需要判断当前机器是否仍处于leader状态。

**心跳：**进入循环后判定heartbeats状态，如何需要发送心跳包给followers，则广播心跳消息，告知所有followers目前本节点的leader身份。发送完RPC消息后，需要将heartbeats设置为false，避免每次循环都发送心跳包，浪费I/O。接着，设置下一次心跳发送的时钟，当事件触发时会将heartbeats设置为true。

**select事务处理：**select作用不赘述，我们用它监听多个channel的状态变化。首先，剥离select中最大的阻塞队列读取任务模块，看一看其他case的作用。第一个case监听系统推出指令；最后一个case设置心跳flag状态；第二个和第三个case作用类似，当leader为一个Log向followers发起一致性请求时，需要等待大部分followers返回消息，才能最终提交该条Log。根据followers RPC返回的状态，将身份ID放入两个channel中。handleResponseAppend处理这些followers返回的状态，决定follower是正常添加了日志，抑或需要重传日志，其内部逻辑本文最后会进行梳理。handleResponseAppend成功返回后，会触发leader的日志提交事件。提交重点看看leaderCommit如何统计一句同步状态的follower个数。

{% highlight Go %}
func (rf *Raft) leaderCommit(commitSync map[int]bool, PeerId int) {
	commitSync[PeerId] = true
	//set Leader commitIndex
	if len(commitSync) >= rf.QuorumSize() {
		//If there exists an N such that N > commitIndex, a majority
		//of matchIndex[i] ≥ N, and log[N].term == currentTerm:
		//set commitIndex = N
		var matchLog []int
		for key, _ := range commitSync {
			matchLog = append(matchLog, rf.matchIndex[key])
		}
		sort.Sort(sort.Reverse(sort.IntSlice(matchLog)))
		//select this QuorumSize()-1 index as this commit point
		candIdx := matchLog[rf.QuorumSize()-1]
		
		if candIdx > rf.commitIndex && rf.Log[candIdx-rf.BaseIndex()].Term == rf.CurrentTerm {
			rf.commitIndex = candIdx
			//apply this commit to state machine
			rf.applyNotice <- true
		}
	}
}
{% endhighlight %}

可以看到leaderCommit的功能与raft论文中描述基本相同，某个Follower返回后，将其状态设置为true，如果过半数后就提交该Log。但是与论文不同的在于，这里进行了优化，不是对每个提案都分别维护一个channel，等等follower返回状态，进行计数统计。这里只使用一个channel跨越多个提案进行统计计数，那么为什么可以这样优化呢？首先为每次一致性协商单独维护一个map很麻烦，很难正确跟踪，开销也高。当commitSync中有超过半数的true项目时，进入判定环节。首先，遍历这些已经反馈的followers，返回的最后写入日志的内容下表是序号。将这些序号进行降序排序，选择第QuorumSize对应的日志下表(idx_quorum)。可以证明idx_quorum对应的日志项目必定被发不发followers通过，因此可以commit该项日志及以前的日志项目。这段代码就是基于这样的一个事实：我们不去计数某一次协商followers的状态返回；转变为追踪所有曾经返回过状态的followers，它们最后返回的可提交的日志序号，排名quorum小位置的序号，表明大部分followers都正确写入了该项日志，所以可以提交。


## 任务处理

最后，我们来看一看第一部分没有谈完的阻塞队列取出任务后，实际的任务处理。我以AgreementArgs类型的一致性协商任务为例子，进行详细说明。看看case里面的处理流程，首先将interface{}形式的任务转变为实际的类型，并使用handle进行对应处理。先不进入handle函数内部，接着设置自己的日志写入状态，重新设置心跳时钟和旗帜。

{% highlight Go linenos%}
case *AgreementArgs:
rp, _ := (e.reply).(*AgreementRely)
*rp = rf.handleRequestAgreement(req, respChan, snapChan)

commitSync[rf.me] = true
timeoutChan = random(HeartbeatInterval, HeartbeatInterval)
heartbeats = false
{% endhighlight %}

回头看看handleRequestAgreement实际业务代码。首先，本次日志记录命令；然后，广播给followers发起一致性协商；最后，设置本地日志长度，和下一条日志位置，返回用户**如何这条日志能够顺利提交，那么它所处的Term和日志位置**。

{% highlight Go %}
func (rf *Raft) handleRequestAgreement(req *AgreementArgs, respChan chan *AppendEntriesReply, snapChan chan *SnatshotReply) AgreementRely {
	//append this entry to local machine.
	entry := Entry{rf.nextIndex[rf.me], rf.CurrentTerm, req.Command}
	rf.mu.Lock()
	rf.Log = append(rf.Log, entry)
	//to agreement with followers
	for i := 0; i < len(rf.peers); i++ {
		if i != rf.me {
			rf.boatcastAppend(i, respChan, snapChan)
		}
	}
	rf.mu.Unlock()
	rf.persist()
	rf.matchIndex[rf.me] = rf.nextIndex[rf.me]
	rf.nextIndex[rf.me]++
	return AgreementRely{rf.CurrentTerm, rf.matchIndex[rf.me]}
}
{% endhighlight %}

那么介绍到这里，就需要看一看boatcastAppend。它的功能很明确，根据该follower上一次的反馈状态，决定需要发送它还没有的Logs。使用一个新的goroutine通过sendAppendEntries进行RPC日志发送处理，避免阻塞主线程逻辑。

{% highlight Go %}
func (rf *Raft) boatcastAppend(server int, respChan chan *AppendEntriesReply, snapChan chan *SnatshotReply) {
	if rf.nextIndex[server] > rf.BaseIndex() {
		prevIndex := rf.nextIndex[server] - 1
		var entries []Entry
		preTerm := -1
		if prevIndex <= rf.LastIndex() {
			entries = rf.Log[rf.nextIndex[server]-rf.BaseIndex():]
			preTerm = rf.Log[prevIndex-rf.BaseIndex()].Term
		}
		args := AppendEntriesArgs{rf.CurrentTerm, rf.me, prevIndex, preTerm, entries, rf.commitIndex}
		go func() {
			r := new(AppendEntriesReply)
			ok := rf.sendAppendEntries(server, args, r)
			if ok {
				respChan <- r
			}
		}()
	}
}
{% endhighlight %}

可以看到新的goroutine中，等待RPC调用成功返回后，会通过channel通知主线程。这个channel作用我们已经在上面解释过，可以回头看看leader_loop中的case第二项。目前，剩余的工作就是查看sendAppendEntries发送日志后，follower如何进行处理和返回结果。follower对应的RPC处理函数是：

{% highlight Go linenos%}
func (rf *Raft) AppendEntries(args AppendEntriesArgs, reply *AppendEntriesReply) {
	ok := rf.deliver(&args, reply)
	if ok != nil {
		reply = nil
	}
}
{% endhighlight %}

看到了嘛，在follower端也是使用deliver包装消息，放入阻塞队列中，等待事件循环取出任务进行处理。那我们来看看follower中事件循环处理该条消息的代码片段如下，逻辑与刚刚leader_loop相似。可以看到case2中是对append任务的处理，调用了handleAppendEntries函数。这里就不放出handleAppendEntries函数的内容，主要就是一些实现raft论文中添加日志的业务功能。感兴趣的可以去[我的github](https://github.com/hawkxiang/Distributed-System/blob/master/raft/append.go）上查看。

{% highlight Go %}
case e := <-rf.blockqueue:
switch req := e.args.(type) {
case *RequestVoteArgs:
	//get voteGranter from candidator, neead resert expired timer
	rp, _ := (e.reply).(*RequestVoteReply)
	*rp, update = rf.handleRequestVote(req)
case *AppendEntriesArgs:
	//get heartbears from leader, need reset expired timer
	rp, _ := (e.reply).(*AppendEntriesReply)
	*rp, update = rf.handleAppendEntries(req)
	//snatshot
case *SnatshotArgs:
	rp, _ := (e.reply).(*SnatshotReply)
	*rp, update = rf.handleInstallSnapshot(req)
default:
	err = NotLeaderError
}

e.err <- err
{% endhighlight %}

那说了这么多是不是说完了日志协商过程呢，其实还没有说event_loop中收到follower反馈后，进行处理的handleResponseAppend函数。我看可以看到这里基本逻辑和Raft的描述相同，首先判断follower端是否接受了传输过去的日志。如果success，调整leader为该follower所维护的日志下表字。；如果fail，判断是不是leader已经过期；如果没过期，根据维护的日志下标，重新获取需要同步的日志内容，发送给该follower。这里的判断条件，对应raft论文中谈到的多种临界case，读者可以读者paper和代码加深理解。

{% highlight Go %}
func (rf *Raft) handleResponseAppend(reply *AppendEntriesReply, respChan chan *AppendEntriesReply, snapChan chan *SnatshotReply) bool {
	if !reply.Success {
		if reply.Term > rf.CurrentTerm {
			rf.updateCurrentTerm(reply.Term, EmptyVote)
		} else if reply.PeerId != EmptyVote {
			//decrement nextIndex, resend append request to remote(reply.PeedId).
			/*rf.nextIndex[reply.PeerId]--*/
			rf.nextIndex[reply.PeerId] = reply.LastMatch + 1

			rf.mu.Lock()
			rf.boatcastAppend(reply.PeerId, respChan, snapChan)
			rf.mu.Unlock()
		}
	} else {
		rf.matchIndex[reply.PeerId] = reply.LastMatch
		rf.nextIndex[reply.PeerId] = reply.LastMatch + 1
	}
	return reply.Success
}
{% endhighlight %}

本文，通过节点leader身份下的事件循环，和任务处理说明了raft算法主要功能实现。实现过程中有很多权衡的地方，限于本人能力，可能存在实现方式不妥的地方，希望得到指正。下一篇文章会谈一谈如何基于目前的Raft算法实现分布式K/V存储系统。


