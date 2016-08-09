---
layout: post
title: rsync容灾备份数据
author: hawker
permalink: /2016/16/rsync-backup.html
date: 2016-1-31 21:10:00
category:
    - 系统运维
tags:
    - rsync
    - backup
---
服务器数据同步备份是常见的容灾需求，我的实验室一台Web服务器需要定时备份网站相关数据至备份服务器上。今天谈谈同步备份的实现。介绍中我们将待备份的Web服务器视作`ClientA`，存储备份数据的服务器称作`ServerB`。

### SSH免密码登录设置

在介绍同步实现细节之前，我们有必要讨论一下如何实现SSH免密码登录。在周期运行的脚本中`(crontab)`，实现远程登录时手动输入登录口令是不可取的。在我们的同步脚本中，`ClientA`需要远程无密码的登录至`ServerB`。首先，在`ClientA`上生成用户sjd_backup登录用的公、私密钥，并将公钥传输给`ServerB`。

{% highlight Vim %}
#创建新的密钥
[clientA_backup@localhost ~]$ ssh-keygen
#对于出现的选项，一直回车，使用默认设置创建密钥。
	
#查看密钥是否创建成功
[clientA_backup@localhost ~]$ ls -ld ~/.ssh; ls -l ~/.ssh
drwx------ 2 clientA_backup clientA_backup 4096 1月 30 02:36 /home/sjd_backup/.ssh
-rw------- 1 clientA_backup clientA_backup 1679 1月 30 02:36 id_rsa
-rw-r--r-- 1 clientA_backup clientA_backup 410 1月 30 02:36 id_rsa.pub
{% endhighlight %}

id_rsa是私钥，用户sjd_backup自己拥有读写权限；id_rsa.pub是公钥，将被传输给`ServerB`。注意创建的密钥文件权限需要正确，~/.ssh/目录应该是700。下面需要将id_rsa.pub传给`ServerB`即可：
	
{% highlight Vim%}
[clientA_backup@localhost ~]$ sftp serverB_backup@xxx.xxx.xxx.xxx  <==ServerB的IP地址。
> put ~/.ssh/id_rsa.pub

{% endhighlight %}
	
接下来我们登录到`ServerB`,更具它的sshd_config中的AuthorizedKeysFile配置选项，找到authorized_keys文件，并将`ClientA`传递过来的公钥id_rsa.pub内容附加到authorized_keys中。

{% highlight Vim %}
#查看ssh配置文件，确定authorized_keys位置
[serverB_backup@localhost ~]$ vim /etc/ssh/sshd_config
	
#如果是新用户，不存在authorized_keys文件，需要手动创建.ssh目录存放authorized_keys文件
[serverB_backup@localhost ~]$ mkdir .ssh; chmod 700 .ssh
	
#确认ClientA发来的公钥id_rsa.pub是否已经收到
[serverB_backup@localhost ~]$ ls -l *pub
-rw-r--r-- 1 serverB_backup serverB_backup 410 1月 30 02:36 id_rsa.pub <==确实有存在
	
#将公钥id_rsa.pub内容附加到authorized_keys中。
[serverB_backup@localhost ~]$ cat id_rsa.pub >> .ssh/authorized_keys
[serverB_backup@localhost ~]$ chmod 644 .ssh/authorized_keys
{% endhighlight %}
	
完成上面`ServerB`的设置后，确认权限是否正确。此时，回到`ClientA`切换到clientA_backup身份即可实现无需密码SSH登录到`ServerB`。

{% highlight Vim %}
[clientA_backup@localhost ~]$ ssh serverB_backup@xxx.xxx.xxx.xxx  <==ServerB的IP地址。
Last login: .............          <==确认可以无密码登录
{% endhighlight %}
	
### 同步备份
我们将使用`rsync`配合`crontab`进行两台机器间的数据备份。`rsync`的基本用法如下，具体的选项和参数可以参考man中的说明：

{% highlight Vim %}
[clientA_backup@localhost ~]$ rsync [-avrlptgoD] [source] [destination]
{% endhighlight %}
	
我们将备份`ClientA`中文件夹`/var/www/html`下所有的文件，以及mysql数据库文件，备份脚本参考下面

{% highlight Vim %}
[clientA_backup@localhost ~]$ vim backup_www.sh

#!/bin/bash
remotedir=/var/www
localdir=/var/www/html
remoteip="xxx.xxx.xxx.xx"      <==根据自己的情况设置
	
mysqldump -uuser -ppassword sjd > /var/www/html/sjd.sql  <==导出数据库文件
#同步备份命令
rsync -avl --delete ${localdir} serverB_backup@${remoteip}:${remotedir}
{% endhighlight %}
	
运行脚本，查看是否可以成功将`ClientA`中文件夹`/var/www/html`下所有的文件，备份至`ServerB`下的`/var/www/html`中。需要主要的是，`rsync`中涉及的用户应该是对应同步目录的拥有者，可以避免备份时时间修改存在的权限问题。

验证脚本正常后，运行`crontab -e`将脚本设置为每天凌晨循环执行的作业。
{% highlight Vim %}
0 1 * * * /home/clientA_backup/backup_www.sh
{% endhighlight %}
	
![程序内存分配](/upload/2016/03/c_memory.png)
OK！大功告成，一切都可以正常的运行，数据可以正常的容灾备份啦！
	
	
