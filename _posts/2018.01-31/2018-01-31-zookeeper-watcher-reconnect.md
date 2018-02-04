---
layout: post
title: Zookeeper Client架构分析——ZK链接重连失败排查
author: hawker
permalink: /2018/01/zookeeper-watcher-reconnect.html
date: 2018-01-31 20:00:00
category:
    - 编程
tags:
    - zookeeper
    - curator
    - architecturear
---
本篇将通过工作遇到的“Zookeeper无法重连”问题，深入Zookeeper Client和Curator进行架构分析，解析出现改问题的原因。网上很多文章介绍了Zookeeper Client特别是Curator的用法，但是鲜有资料分享两者的内部实现和工作原理。既然，本篇文章是为了解决针对性问题，故而文章分为三段进行阐述：What、Why、How。

## What：遇到了什么问题
工作上，最近刚刚接手了一个新的组件维护工作，该服务与Zookeeper Server进行数据同步，维护大量的Watcher，当然也会周期性的从ZK手动拉去数据。这个服务当数据量小时一切正常，当我接手时整个数据量级已经很大，这时服务存在一个严重的问题：当到Zookeeper server的链接断开后难以重连成功。我们总结一些问题表现：
1. 服务和Zookeeper之间保持大量的Watchers。
2. 服务于Zookeeper的链接断开后，无法自愈。
3. 服务以前不存在链接断开无法自愈的问题。
4. Curator针对Session_Expired可以通过重建方式，恢复session+connect

我觉得问题已经描述的足够清楚，接下来我将花大量篇幅描述问题产生的原因；在这个过程中将介绍Zookeeper Client的架构以及Curator这个开源库相关实现。因为，博主本身不懂Java语言描述中错误处请读者见谅，同时给予纠正。


## Why：问题产生的原因
首先，我们根据背景描述，简单的分析问题，梳理排查方向。Zookeeper本身具有底层链接重连机制，这个大家都是清楚的；同时，组件以前也是能够通过Zookeeper自愈机制恢复链接。现在出现问题，很明显我们会将问题瞄准“数据量变大，Watcher变多”这个方向。所以，博主想到需要阅读Zookeeper Client源码，了解链接重建时，Watchers的恢复机制。但是，为了文章叙述的方便，不会直接阐明这个恢复流程，我将首先介绍Zookeeper Client端的架构以及交互流程，方便后续的说明。

#### Zookeeper Client架构
为了叙述直观方便，博主根据自己的理解手动画了一个Zookeeper Client架构图，仅仅代表本人的了解，限于本人能力，存在的问题请大家指出。
![Alt text](/upload/2018/01/zk_client.jpg "ZK_Client")

不论我们是直接使用Zookeeper Client库抑或Curator这样封装后的工具，当我们进行ZK操作时，都需要获取一个对象，我们将其称作zkClient。zkClient包括很多成员，抛开细节选取与问题相关的重要成员进行讲解：

1. defaultWatcher就是一个Watcher在创建zkClient都会内部创建一个，如果使用了Curator，他会分装一个connectState对象，作为defaultWatcher传入zkClient创建过程。
2. ClientCnxn是zkClient重要成员，可以说它实现了大量Zookeeper Client的功能逻辑。我们接下来详细介绍它那边的成员和工作原理。

ClientCnxn又包括两个对象，或者说工作线程：eventThread和sendThread。

1. eventThread: 主要维护一个队列eventQueue，里面存放Watcher触发后的事件。可以把eventThread内部看成死循环，一直Loop从Watcher取出事件，最终调用业务代码指定Watcher时传入的process函数。
2. sendThread：维护两个队列，sendQueue发送队列，zkClient所有发完Zookeeper集群的数据都会先包装成自定义的Packet形式，放入发送队列。底层的NIO会从发送队列取包，不断发送出去。pendingQueue也比较好理解，里面存放已经发送但是服务还未响应的包。sendThread也有一个死循用，处理整个zkClient的主业务，并进行业务通讯。
3. ClientCnxNIO：zkClient底层的网络通讯依赖ClientCnxNIO这个对象进行，这里不详细说明IO复用相关知识。

介绍完zkClient基本情况，我们返回本节开始时谈到的内容：Zookeeper Client连接恢复时，做了什么事情。Zookeeper本身需要维护一个Session的概念，其实重建链接后做的就是Session恢复的事情。zkClient中有一个叫做primeConnect的函数，主要负责这件事情，他会将client的Watcher，Auth，session其他数据打包成多个Packet放入sendQueue发送给server，进行sesion恢复。这里需要提及的是：

1. primeConnect打包多个Packet，尺寸128KB左右；原因是如果client端Watcher很多整体数据量容易超过1MB；Zookeeper Server要求数据包小于1MB，大包视为异常，会关闭链接。
2. 只要链接建立这个primeConnect都会被调用，同时会将本地的SessionId等信息也传给Server。

从Session恢复的处理来看，不存在异常问题。而且，1MB这个限制已经做了底层处理，不会导致Server主动关闭链接，致使zkClient重连失败。我们还需要进一步排查。

#### SessionExpired事件
在深入介绍前，我想先简单回顾一下Zookeeper Session的状态图：
![Alt text](/upload/2018/01/sesionState.png "sessionState")

整个ZK session的状态转换，包括以下几个状态。

1. CONNECTED：链接建立成功
2. CONNECTION_LOSS: 链接断开
3. CONNECTING: 链接建立中
4. SESSION_EXPIRED: session过期，这是一个重要的状态，出现它时Zookeeper Client本身无法自愈恢复链接和session。

看起来SESSION_EXPIRED是一个导致我们不能重建链接成功的原因，错误日志也验证这一点。但是Curator本身可以通过重建zkClient的方式，应付这个状态，保证重建链接成功。但是，很明显这个操作没有成功，因此我们需要跟踪为什么没有去重建zkClient，而且是在Watchers增多后无法进行这个操作。因为，博主已经验证watchers少量时，其实可以顺利恢复链接。

那么我们的真对组件无法重建zk链接的排查面临两个疑惑：

1. Curator提供了SESSION_EXPIRED重新创建Client，达到重建session+connect的效果，为什么没有生效。
2. NIO底层会一直重试网络链接，恢复旧的connect，为什么恢复失败？

首先，我们来看看第一个困惑点。当然继续讲解前，我们首先回归一下Zookeeper中的，Watcher触发类型：

1. EventType.Node：与链接相关的状态会触发这个事件，Disconnected、SyncConnected、AuthFailed、Expired。并且所有的watcher都会收到这个事件。
2. EventType.NodeCreate：节点创建，关注这个的watcher收到事件。
2. EventType.NodeDeleted：节点删除，关注这个的watcher收到事件。
2. EventType.NodeDataChanged：节点数据变动，关注这个的watcher收到事件。
2. EventType.NodeChildrenChanged：子节点节点变动，关注这个的父节点Watcher收到事件。


回到上面的zkClient架构图，注意图中ClientCnxNIO在网络恢复后，会进行重连，并把本地旧的session信息传给server。因为超时，server会告诉client一个状态，SessionExpired。这个事件在重连成功后，因为是EventType.Node，会触发zkClient中所有的Watchers，它们都会收到这个事件。业务层的wacher并不会做出特殊处理；但是讲架构时我们说过zkClient会产生一个内部Watcher；使用了Curator的工具库后，这个内部Watcher就是connectState，这个对象的process函数会发起重建session的操作。

按照这个思路，应该可以顺利建立链接成功；为什么实际中一直无法重建链接成功呢？回到架构图，注意ClientCnxNIO抛出的SessionExpired事件，会进入eventQueue这个队列。这时博主注意到，业务代码中watcher自己的process函数在处理完后都会进行自注册，避免watcher丢失。这身无可厚非，但是这里的AddWatcher采用了同步方式，那么链接已经断开的情况下每个重新注册watcher的操作，会一直重试，直到超时（分钟级别超时），才能退出process。在介绍zkClient时，说过zkClient触发watcher的process其实是在一个循环中进行的，如果前面watcher的回调函数处理完毕才能处理后面的watcher的回调事件。

这就有问题了，大家想想时链接断开时，会抛出一个Disconnected状态，他是EventType.Node类型事件，所有的watcher都会接受到。那么eventQueue就会一下加入很多元素，博主的组件可能会有几十万watcher。这些watcher事件在队列中排队处理，如果每个1min，想想处理完Disconnected需要多久？？？接下来链接恢复时，因为sessionExpired抛出的事件，才进入队列尾部；等到前面Disconnected事件处理完毕才会处理Expired相关的watcher回调，当然这个包括上文谈到的默认watcher进行session重建的回调。

所以，在使用同步AddWatcher时，大量watcher场景下，永远无法出发Curator提供给zkClient进行session重建的默认watcher。虽然Curator无法通过重建session创建新的链接，但这里不能解释第二个困惑点：NIO底层其实会一直重建网络链接，理论上网络恢复后旧的链接会顺利恢复才对呀。

#### 网络恢复与SessionExpired事件回调处理

解释NIO无法恢复旧的connect失败前，我们来看一看重连后client接到server告知SessionExpired后，做了那些事情。博主在原先的架构上，进行了一定的修改SessionExpired架构图如下：
![Alt text](/upload/2018/01/sessionExpired.jpg "sessionExpired")

首先，看下sendThread里面的主循环做了什么，正如上面说的链接正常时它是不断发包、收包，循环处理。当链接断开时，主循环不断尝试reConnected，当然会有sleep，避免busyLoop。

当网络恢复时，NIO通过doIO收取数据包，对于reconnect事件会教给readConnectResult函数进行处理。readConnectResult继续调用sendThread出入的onConnected回调事件。server返回SessionExpired状态，其实已经超时了，onConnected里面会将zkClient的状态置为States.CLOSED。这个状态特别重要，因为sendThread里面维护的主循环break的条件就是这个；sendThread跳出主循环后就是做clean的操作，其实就是关闭自己的链接，注意链接成功的链接会被关闭。同时，由于sendThread已经退出了主循环，所以不会在发起reConnected的操作。这解释了第二个疑问，为什么NIO在网络恢复后，不能恢复旧的链接。

ok，到这里好像链接不能重建的问题已经解释清楚，我们只能走重建session一条路，同时业务代码避免使用同步AddWatcher方式，真是如此吗？

#### Curator的session重建机制

真对以上两个结论，博主将业务代码触发的process中wacher重新注册改为异步后，确实能够重建Session+session成功，大家是不是认为问题已经解决。别急，虽然链接问题解决了，但是又遇到新的问题：通过重建session后watcher有时会丢失，接下来我们继续进行问题排查和架构分析。分析watcher丢失原因前，我们先看看Curator重建Session的实际流程吧。我们还是参考上面SessionExpired架构图进行分析。

分析Curator提供的默认watcher即connectState发现，它的回调处理中针对sessionExpired事件调用了reset操作。如图，当sessionExpired触发进入队列头部，取出触发connectState的process进而进行reset处理。reset首先调用底层zkClient内部的disconnect函数，它做两件事情：

1. sendThread：关闭主循环，清空万里sendQueue、pendingQueue以及关闭socket；设置zkClient状态为CLOSE。这时其实已经无法zkClient进行主逻辑处理。
2. eventThread：在eventQueue末尾加入一个新的事件eventofDeath，此外zkClient的默认watcher替换成dummyWatcher，就是不进行任何事件处理。等到eventofDeath被处理时，eventQueue不在接受任何事件入队，知道队列排空后推出该事件队列。

disconnect其实就是做一些清理工作，也是关键一步。接着Curator会重建一个zkClient达到重建session+connect的目的，最终恢复功能。ok，至此销毁到重建流程梳理清楚。那为什么重建session后会watcher会丢失呢，我们业务层一直进行自注册理论上通过新的zkClient可以将旧watcher注册进来，不会丢失。

#### Curator主动销毁Session

针对这个问题，博主继续分析代码。发现Curator在每次调用zk操作时，需要先检测链接状态，如下图所示：
![Alt text](/upload/2018/01/statejudget.jpg "statejudget.jpg")

每次业务代码进行zk操作时都要通过getZookeeper获取handle，这个函数会检测connect状态。如果已经断开，它会接着检测是否超时（超时值在业务代码创建zkClient传入，通过与server协商最终确定，大概分钟级别）。如果超时了，注意这里会主动调用reset操作，销毁session，并尝试重建一个handle返回给用户。

结合博主的业务代码逻辑，服务中会有一个异步线程每个一段时间执行zk.getData()操作。这里会触发getZookeeper调用，进行状态检测。如果此时网络未恢复，或者说eventQueue中的SessionExpired还未被触发。那么就会进行Curator主动的reset操作，它销毁了旧的session，由于这个流程不在watcher中出发，旧的watcher不会进行自注册，所以会有丢失情况出现。

因此，这里进行session销毁的总结：
1. 被动：网络恢复时，sessionExpired事件可以出发默认watcher调用reset进行session重建。
2. 主动：网络断开时，业务其他业务点名进行zk操作，通过getZookeeper调用会触发状态检测，调用reset重建session。

感觉所有问题都解决了，好开心。不对，回过头看一下上面第二种session重建的情况。这种情况下就算同步watcher堵塞了eventQueue的消费，导致通过默认watcher进行的被动session重建，无法成功。如果Curator可以主动重建session，那为什么在以前同步watcher场景下，没有进行session重建？

好吧，突然开始怀疑自己之前的排查思路，绝望！！！！只能继续分析Curator源码。既然主动reset依赖状态检测，那就重点看这里的实现，原来isConnected在false情况下才会进行reset操作。而这个isConnected状态的修改，也是通过默认Watcher进行触发的，基本流程如下图所示：

![Alt text](/upload/2018/01/statechange.jpg "statejudget.jpg")

同步watcher机制情况下，默认watcher一直被堵住，所以isConnect的状态一直是true，所以一直没有主动进行session重建操作。

## How：如何解决呢

支持，问题的原因我们已经弄清楚了，解决方案和业务代码相关在此不在介绍。虽然我不会java，但是很开心排查出个Zookeeper链接无法重建的问题。

后记：永远保持一颗怀疑心！
