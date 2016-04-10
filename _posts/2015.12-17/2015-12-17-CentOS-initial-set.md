---
layout: post
title: CentOS7初始化设置（Initial setup of CentOS7）
author: hawker
permalink: /2015/12/CentOS-initial-setup.html
date: 2015-12-17 21:03:00
category:
    - 系统运维
tags:
    - Linux
---
CentOS一直是liunx server的稳定系统，前端时间安装新的CentOS 7，重启后一直无法进入图形界面，当然文字界面进入很正常。可能是以前给服务器装系统很少安装Gnome Or KDE 的缘故，初见这个问题时十分费解，也可能是CentOS 7的初始化设置太简陋的缘故，总之解决这个问题花费了一些精力。
&nbsp;

### 配置提示

安装带图形化的CentOS 7后，重启重进系统时，出现下面提示信息：

{% highlight c %}
Initial setup of CentOS Linux 7 (Core)
1) [!] License information
	(License not accepted)
Please make your choice from [ '1' to enter the License information spoke | 'q' to quit | 'c' to continue | 'r' to refresh]:
{% endhighlight %}

其实解决方法比较显然，与图形化的系统初始化设置相似。根据提示一步步操作即可：

1. 输入`1`，回车阅读授权;
2. 根据提示信息，接受授权，我是输入了`2`，根据提示信息输入即可;
3. 键入`q`，离开设置;
4. 根据提示保存设置，一般是输入`yes`。

即可进入CentOS的图形界面，当然，也可能需要重启后才能进入系统。
