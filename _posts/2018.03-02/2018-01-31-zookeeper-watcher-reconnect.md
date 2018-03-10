---
layout: post
title: Linux大量异常ESTABLISHED TCP连接问题排查
author: hawker
permalink: /2018/03/connect-established.html
date: 2018-03-02 20:30:00
category:
    - 编程
tags:
    - TCP/IP
    - Linux
---
最近在工作中遇到一个关于TCP/IP中连接异常的问题，加深了对与TCP/IP整套机制的理解。在此，进行分享希望对遇到同样问题的朋友有所帮助。

## TCP连接异常问
工作上一个服务端程序，对外提供短连接RPC响应。服务本身有多个节点，偶然发现其中一个节点的ESTABLISHED达到6万，并且一直无法降低（其他节点1000左右，qps均衡）。遂跟进排查这个奇怪的现象。在RPC通信中，一般client调用完毕后主动clode connection，服务端被动进行关闭。根据TCP/IP四次挥手的流程，机器不应该保存有大量的ESTABLISHED状态连接。因此，我分为以下几个流程进行排查：

1. 通过awk对这些异常连接进行排序、去重、计算client端分布，以及每个client端有异常连接个数，希望能够查处规律。结果，没有发现明显特征。
2. 通过tcpdump确认异常ESTABLISHED连接，是否正常：发现连接其实已经失效。
3. 为了进步验证2的“连接生效”结论，随机登陆一台client机器；检查是否存在对应服务端的异常连接，发现确实不存。因此，可以确认client已经消耗了connection，但是服务端因为某种情况保留了这些异常的connection，并且维持状态是：ESTABLISHED。

问题比较明确，接下来就是分析原因，找出解决方案来。

## 原因分析和问题处理
通过查询资料，找到一篇类似想象的文章：https://superuser.com/questions/1021988/connection-remains-flagged-as-established-even-if-host-is-unconnected/1022002
其实，文章中对应上面的问题现象和原因分析的比较明确，也给出了合理的处理方案。

### 原因分析
TCP是一种可靠传输连接，如果server端没有收到client发送的FIN包，将会一直保持ESTABLISHED状态，不论client是否正常、网络中的路由节点是否正常工作。此外，本身tcp没有心跳探活机制，因此如果client端异常关闭连接，另一端是无法感知到，并且从协议上理解也没必要感知到对端关闭。这也解释，为何TCP优雅关闭connection时需要采用四次挥手机制。

所以，造成服务端大量异常ESTABLISHED连接，并且无法销毁的可能性原因是：

1. client异常“关闭”了connection：这里的“异常”有多种可能性，client机器奔溃了、client端程序异常退出，这两种情况都没有调用close，发送FIN。另外，也有可能server端机器的中断机制异常，导致对应FIN包的处理出现问题，没有顺利处理TCP的四次挥手接下来流程。总结存在某个“异常”case导致client实际关闭了connection，但是server端未进行响应。
2. 由于是RPC通信，server端只是被动的响应client请求；并不会通过connection主动向client发送数据。因此，当client实际关闭了connct，如果这条连接一直没有数据通信，永远无法感知到connect的异常。所以server将保持connect的ESTABLISHED状态。

### 问题处理
通过上面的解释，解决方案也已经比较明确。TCP/IP针对通过实际关闭的连接发送数据时，会返回错误，并销毁connect。所以，我们只需要实现一次“发包”，其实就能处理这个问题。当然，TCP协议本身就提供了这个功能，可以在linux中设置tcp_keepalive_time参数时间，它将在timeout后进行“探活”发包，资源这个问题。需要注意的是，设置这个系统参数后，需要重启异常程序，后续不会出现异常ESTABLISHED状态连接的问题。