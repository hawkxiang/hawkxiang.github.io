---
layout: post
title: NLSR配置及安全证书链生成
author: hawker
permalink: /2016/04/nlsr-security-certchain.html
date: 2016-04-10 18:00:00
category:
    - 编程
tags:
    - ndn
    - security
---
NLSR是由NDN项目组开发的路由协议，现已在Testbed上进行使用。NLSR的安装文档可以在官网上查看：[传送门](http://named-data.net/doc/NLSR/current/)。基本的安装流程网站上有详细的说明，一步步安装即可。NLSR的配置主要分为两个部分，邻居路由器节点的配置及安全证书链。

## 邻居节点的配置

基本邻居路由的配置，文档的说明还是挺详细的。这部分的配置需要关注以下配置项：

1. general：层次化的命名架构，NLSR是由三部分组成(network, site, router)。需要注意前两层需要同对应的安全证书一致。
2. neighbors: 在这个配置项目中，要注意neighbor的配置。有多少直接相连的路由节点，就需要配置相应个数的neighbor。
3. advertising: 测试NLSR是否能正常同步路由表信息。如果配置成功，在其它节点通过`nfd-status`查看状态时，可以在RIB及FIB中查看到此处宣告的命名前缀。
4. log: general中有日志信息选项，建议测试配置时将log-level设置为ALL方便查看错误，实际使用时可以不需要日志。

基本的配置就说这么多，配置文件在那儿？如何启动nlsr？其它细节问题，多多查看官方说明。

## Security配置
NLSR麻烦的地方在于，它需要为同步域构建一套完整的安全证书链。然而，奇怪的是官网只说了安装证书链的层次要求，没有详细的证书生成及配置说明。按照官网的思路，我们也先了解一下什么是证书链。

#### 安全证书链
我们将整个网络视为一个同步域，路由协议主要的工作，其实是同步这个网络中路由节点的数据集（路由表信息）。目前NLSR数据同步时需要一套层次化的证书结构，进行身份验证。自顶向下整个证书链是一个树形的结构，包括四种安全证书：`rout`，`site`，`operator`，`router`，`route\NLSR`。

```
                                root
                                  |
                   +--------------+---------------+
                 site1                          site2
                   |                              |
         +---------+---------+                    +
      operator1           operator2            operator3
         |                   |                    |
   +-----+-----+        +----+-----+        +-----+-----+--------+
router1     router2  router3    router4  router5     router6  router7
   |           |        |          |        |           |        |
   +           +        +          +        +           +        +
 NLSR        NSLR     NSLR       NSLR     NSLR        NSLR     NSLR
```
我们在生成证书之前，需要根据网络拓扑构造一个合适的树形结构。总结一下，主要关注以下几点：

1. `root`证书只需在一台节点上生成，整个同步域公用同一个`root`证书。
2. `site`证书与`operator`证书建议在同一个节点上生成；它们将整个网络化为多个子域，它们的个数根据实际网络拓扑决定。当然`site`的产生是依赖`root`；`operator`依赖`site`。
3. 每个路由节点都拥有自己的`router`证书，`route\NLSR`证书当运行nlsr时会自动生成。
4. 证书的名字，应该符合域中节点的层次化命名规则，具体要求如下表:

<table>
    <tr>
        <td>Entity</td>
        <td>Identity</td>
        <td>Name</td>
        <td>Example	Certificate Name Example</td>
    </tr>
    <tr>
        <td>root</td>
        <td>/&lt;network&gt;</td>
        <td>/ndn</td>
        <td>/ndn/KEY/ksk-1/ID-CERT/%01</td>
    </tr>
    <tr>
        <td>site</td>
        <td>/&lt;network&gt;/&lt;site&gt;</td>
        <td>/ndn/edu/pku</td>
        <td>/ndn/edu/pku/KEY/ksk-2/ID-CERT/%01</td>
    </tr>
    <tr>
        <td>operator</td>
        <td>/&lt;network&gt;/&lt;site&gt;/%C1.Operator/&lt;operator-name&gt;</td>
        <td>/ndn/edu/pku/%C1.Operator/op1</td>
        <td>/ndn/edu/pku/%C1.Operator/op1/KEY/ksk-3/ID-CERT/%01</td>
    </tr>
    <tr>
        <td>router</td>
        <td>/&lt;network&gt;/&lt;site&gt;/%C1.Router/&lt;router-name&gt;</td>
        <td>/ndn/edu/pku/%C1.Router/rt1</td>
        <td>/ndn/edu/pku/%C1.Router/rt1/KEY/ksk-4/ID-CERT/%01</td>
    </tr>
    <tr>
        <td>router</td>
        <td>/&lt;network&gt;/&lt;site&gt;/%C1.Router/&lt;router-name&gt;/NLSR</td>
        <td>/ndn/edu/pku/%C1.Router/rt1/NLSR</td>
        <td>/ndn/edu/pku/%C1.Router/rt1/NLSR/KEY/ksk-5/ID-CERT/%01</td>
    </tr>
</table>

希望，大家留意上表中的命名层次。我们将在接下来的证书生成过程中，实践这一规则。

#### 证书链的生成
安装ndn-cxx库之后，会生成一系列证书相关命令。可以通过`ndnsec -help`查询用法，在此不在赘述。我们将以两个路由节点的“栗子”说明证书链产生的流程，和涉及的相关命令。两个路由节点分别为`router1`和`router2`。我们将在`router1`节点上产生整个同步域需要的`root`、`site`、`operator`证书；并为`router1`和`router2`分别产生各自的`router`证书。使用的各条命令参数及用法，通过`-help`都有详细说明。具体流程如下：

{% highlight bash%}
1. 在router1上，为域内所有的路由节点产生公用的root证书，它是自签名的。
ndnsec-key-gen -n /ndn
ndnsec-sign-req /ndn > root.cert

2.1 在router1上，为子域所有路由节点产生公用的site证书key，和未签名的site证书。
ndnsec-key-gen -n /ndn/edu/pku > unsigned_site.cert
2.2 用root证书的key，签名site证书。
ndnsec-cert-gen -N "project_name" -s /ndn -p /ndn/edu/pku -r unsigned_site.cert > site.cert
2.3 建议安装证书，一方面避免同名冲突，另一方面方便查看证书的依赖关系。
ndnsec-cert-install -f site.cert

3.1 在router1上，为子域所有路由节点产生公用的operator证书key，和未签名的operator证书。
ndnsec-key-gen -n /ndn/edu/pku/%C1.Operator/op > unsigned_operator.cert
3.2 用site证书的key，签名operator证书。
ndnsec-cert-gen -N "project_name" -s /ndn/edu/pku -p /ndn/edu/pku/%C1.Operator/op -r unsigned_operator.cert > operator.cert
3.3 建议安装证书。
ndnsec-cert-install -f operator.cert

4.1 为router1产生属于自己的router证书key，及未签名证书。
ndnsec-key-gen -n /ndn/edu/pku/%C1.Router/router1 > unsigned_router1.cert
4.2 用operator证书的key签名刚刚产生的未签名router证书。
ndnsec-cert-gen -N "project_name" -s /ndn/edu/pku/%C1.Operator/op -p /ndn/edu/pku/%C1.Router/router1 -r unsigned_router1.cert > router1.cert
4.3 建议安装证书。
ndnsec-cert-install -f router1.cert

///////////////////////////////////登陆至router2主机//////////////////////////////////////////

5.1 为router2产生属于自己的router证书key，及未签名证书。
ndnsec-key-gen -n /ndn/edu/pku/%C1.Router/router2 > unsigned_router2.cert
5.2 将上面产生的未签名证书unsigned_router2.cert上传到router1节点，通过operator证书对其进行签名。
ndnsec-cert-gen -s /ndn/edu/pku/%C1.Operator/op -p /ndn/edu/pku/%C1.Router/router2 -r unsigned_router2.cert > router2.cert
5.3 将签名后的router2.cert拷贝回router2节点。
5.4 建议安装证书。
ndnsec-cert-install -f router2.cert
{% endhighlight %}

理论上上面操作成功后，两个节点的证书链已经生成成功。两个节点上运行nlsr:

{% highlight bash%}
//设置节点的默认证书，并启动NLSR.
router1: ndnsec-set-default /ndn/edu/centaur/%C1.Router/router1; nlsr -f nlsr.conf.router1
router2: ndnsec-set-default /ndn/edu/centaur/%C1.Router/router2; nlsr -f nlsr.conf.router2
{% endhighlight %}

并通过`nfd-status`查看每个节点的RIB信息是否有对方节点的adversing prefix，如果未同步成功，我建议同步以下步骤debug。

### 证书链的错误排查
nlsr是有运行日志的，默认文件是存储在`/var/log/nlsr/nlsr.log`。在基本设置时，建议你开启所有的日志选项，方便查错。查看日志，了解错误原因，如果是基本设置错误，重新设计一遍。如果看到不能获取某个证书的ERROR（尤其的广播证书）：

{% highlight bash%}
"Validation Error: Cannot fetch cert: /ndn/broadcast/KEYS/ndn/edu/pku/%C1.Router/router2/KEY/ksk-1459155962653/ID-CERT "
{% endhighlight %}

出现类似上面的错误，基本可以确定你的证书链产生中有错误，按以下步骤排查。

首先，建议你回顾一下生成证书链及运行nlsr命令整个流程中，是否出现权限不一致的问题即：是否有些命令是通过sudo运行的；有些只是权限产生的。目前，nlsr认为sudo产生的证书并不属于普通的用户。如果存在这样的问题，你需要重新产生一次证书链，并保持整个流程权限一致。针对这个bug，下文会有详细的说明。

其次，我们需要检查证书链的层次依赖关系是否正确。我们将使用`ndnsec-list`及`ndnsec-cert-dump`进行查询，注意这些命名也需要***保持权限一致***，不要用`sudo $COMMAND`去查看普通用户证书，反之亦然。查看节点上证书key的列表，正常的输出应该如下：

{% highlight bash%}
router1上：
* /ndn/edu/pku/%C1.Router/router1         //默认证书
  /localhost/operator
  /ndn
  /ndn/edu/pku
  /ndn/edu/pku/%C1.Operator/op
  /ndn/edu/pku/%C1.Router/router1/NLSR    //执行NLSR后，会自动生成
  
router2上：
* /ndn/edu/pku/%C1.Router/router2
  /localhost/operator
  /ndn/edu/pku/%C1.Router/router2/NLSR  //执行NLSR后，会自动生成
{% endhighlight %}

如果你在证书生成时手动安装了每一个证书（执行`ndnsec-cert-install -f $CERT_NAME`），那么证书之间可以看到明显的依赖关系。你可以通过`ndnsec-cert-dump -i $IDENTITY_NAME -p | grep -A 1 "Certificate name:\|Key Locator:"`查看每个节点上证书的依赖关系，这里的`$IDENTITY_NAME`是证书名字。当然如果你没有手动安装每个证书，可以通过查看路由节点上的`router.cert`与`NLSR.cert`的关系。我们只在router1上看一下二者的关系，当然我建议你检查所有证书。

{% highlight bash%}
查看router.cert:
router1@debian:~$ ndnsec-cert-dump -i /ndn/edu/pku/%C1.Router/router1 -p | grep -A 1 "Certificate name:\|Key Locator:"
Certificate name:
  /ndn/edu/pku/%C1.Router/router1/KEY/ksk-1460797675386/ID-CERT/%FD%00%00%01T%1E_c1
--
  Key Locator: (Self-Signed) /ndn/edu/pku/%C1.Router/router1/KEY/ksk-1460797675386/ID-CERT
//注意：我没有手动安装router.cert，所以"Key Locator:"中显示是自签名的，摒弃key-id是自身的证书ID；如果执行了手动安装(Self-Signed)会变为(Name)，并且ked-id会变为它所依赖证书的签名即operator的ID。
  
查看NLSR.cert:
router1@debian:~$ ndnsec-cert-dump -i /ndn/edu/centaur/%C1.Router/router1/NLSR -p | grep -A 1 "Certificate name:\|Key Locator:"
Certificate name:
  /ndn/edu/centaur/%C1.Router/router1/NLSR/KEY/ksk-1461036117420/ID-CERT/%FD%00%00%01T%2C%89%EF%C3
--
  Key Locator: (Name) /ndn/edu/centaur/%C1.Router/router1/KEY/ksk-1460797675386/ID-CERT
//注意：NLSR.cert是执行nlsr后自动生成并安装的，开始时没有。此外观察它的"Key Locator:"信息，key-id不是自己的，而是router.cert的。
{% endhighlight %}

再次强调不要用sudo执行查看命令，你会得到证书不存在错误`ERROR: certificate not found
`。当然有时并没有错误，而且成功打印出证书信息，因为你曾经用`sudo`权限产生过该证书。请你仔细察看证书信息，会发现与普通权限执行`ndnsec-cert-dump`显示的信息并不相同，这是两个不同的证书。当然，如果你在证书链生成过程中，为每个证书执行了手动安装会避免同名冲突。

基本上NLSR可能存在的坑，我已经详细说明了。如果你在配置过程中还存在问题，请仔细察看日志文件，定位错误。如果，不能解决我们可以进行交流。

#### 自动生成证书链的脚本
好吧！作为程序猿，这么麻烦的操作过程，当然需要写一个自动脚本喽！下面放出两个证书链生成脚本：masterGA.sh，slaverGA.sh。前者用于在一台机器上产生`root.cert`，`site.cert`，`operator.cert`，`router.cert`；后者用于在其它路由节点上产生`unsigned_root.cert`，并传至主机器上签名后拷贝回来。

{% highlight vim%}
#!/bin/bash
#author: ZhangXiang

#description: 在master节点产生root.cert operator.cert site.cert router.cert并签名安装。脚本中的-N参数是项目名称，可以根据实际改变。

#检测是否需要创建新的certs文件夹
if [ ! -d "certs" ];then
mkdir certs
fi
cd certs

#证书链的key名字，可以更加实际情况修改，需要注意层次结构
root_key="/ndn"
site_key="/ndn/edu/pku"
operator_key="/ndn/edu/pku/%C1.Operator/op"
router_index=3
router_key="/ndn/edu/pku/%C1.Router/router$router_index"

ndnsec-key-gen -n $root_key
ndnsec-sign-req $root_key > root.cert

ndnsec-key-gen -n $site_key > unsigned_site.cert
ndnsec-cert-gen -N "project_name" -s $root_key -p $site_key -r unsigned_site.cert > site.cert
ndnsec-cert-install -f site.cert
#注意：我只手动安装了site.cert，其它证书建议手动安装

ndnsec-key-gen -n $operator_key > unsigned_operator.cert
ndnsec-cert-gen -N "project_name" -s $site_key -p $operator_key -r unsigned_operator.cert > operator.cert

ndnsec-key-gen -n $router_key > unsigned_router.cert
ndnsec-cert-gen -N "project_name" -s $operator_key -p $router_key -r unsigned_router.cert > router.cert

ndnsec-set-default $router_key
cd ..
{% endhighlight %}

{% highlight vim%}
#!/bin/bash
#author: ZhangXiang

#description: 在slaver节点上产生未签名的unsigned_router.cert；将unsigned_router.cert拷贝到master节点上进行签名。最后将签名后的router.cert和master节点上的root.cert、site.cert拷贝至该节点。

if [ ! -d "certs" ];then
mkdir certs
fi
cd certs

#产生一个未签名的unsigned_router.cert
operator_key="/ndn/edu/pku/%C1.Operator/op"
router_index=4
router_key="/ndn/edu/pku/%C1.Router/router${router_index}"
ndnsec-key-gen -n $router_key > unsigned_router.cert

#主服务器的ip地址，登陆名，及证书存储位置根据实际情况修改
slaver_router="unsigned_router_slaver.cert"
dir="~/Workspace/certs/"
project="project_name"

master_ip="219.223.192.229"
username="centaur"
#拷贝至主机器上签名，并拷回
scp unsigned_router.cert ${username}@${master_ip}:${dir}${slaver_router}
sign_cmd="cd $dir; ndnsec-cert-gen -N $project -s $operator_key -p $router_key -r $slaver_router > router${router_index}.cert"
ssh -t -p 22 ${username}@${master_ip} ${sign_cmd}
scp -r ${username}@${master_ip}:${dir} .

ndnsec-set-default $router_key
cd ..
{% endhighlight %}

大功告成？好像遗忘了什么，我们还没为每个路由节点的nlsr.conf配置相关证书位置呢！
