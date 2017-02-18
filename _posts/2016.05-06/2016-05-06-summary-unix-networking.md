---
layout: post
title: Unix网络编程基本概述
author: hawker
permalink: /2016/05/unix-networking.html
date: 2016-05-06 18:00:00
category:
    - 编程
tags:
    - Unix
    - Networing Programming
---
Unix网络编程主要需要注意socket的阻塞和非阻塞的处理，避免死锁。同时，socket属性的设置也是需要注意的基本问题。

## 两种类型Socket

Unix下Socket主要有两种阻塞和非阻塞，服务端开发一般使用非阻塞socket+IO复用技术进行网络通讯。如果需要使用阻塞socket，请注意一下两点，防止client和server在数据通讯时发生死锁。

1. client发送数据给server，server发送数据给client，如果数据都很大时，那么两端一起发送数据时，会阻塞在send操作上，由于每一个连接的接受端缓冲区的大小都是有限的，当接受端的接收缓冲区满时，发送端将一直阻塞在send操作上，这样一来，两端都没有读取接收缓冲区的数据，导致一直阻塞在send操作上。从而出现死锁。
2. client发送很大的数据给server，server只是echo给client，由于数据比较大，client会阻塞在send上，此时echo给client的数据占满了client的接收缓冲区，那么server的send操作将会阻塞，此时不会读取client发送的数据，这样会导致server的接收缓冲区满了，此时client的send也会一直阻塞，从而出现死锁。
这种原因是因为应用层没有缓冲，如果两端都先接收好完整的数据存在应用层的buffer上，那么就不会出现死锁了。两端先发送需要发送的数据的大小，接下来接收到完整数据后，就可以发送给对端了。
