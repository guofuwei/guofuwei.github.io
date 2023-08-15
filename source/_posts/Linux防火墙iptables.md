---
title: Linux防火墙iptables
---

# Linux 防火墙 iptables

## 简单介绍

Linux 防火墙多种多样，如 debian 系列默认使用的`uwf`，redhat 使用的`firewall`，以及今天主要介绍的`iptables`

同一台机器上面可以出现多种防火墙系统，上面的三种可能会同时出现，任何一个防火墙未放行时都会导致丢包

`iptables`是最古老的 Linux 防火墙，而`ufw`和`firewall`都是为了简化`iptables`的操作而出现的较新型防火墙，

`iptables`也不是真正的防火墙，它更像是一个代理程序，将接收到的命令转发到`netfilter`

`netfilter`是 linux 操作系统核心层内部的一个数据包处理模块，主要功能包括：

1.网络地址转换(Network Address Translate)

2.数据包内容修改

3.以及数据包过滤的防火墙功能

## iptables

### 一个简单的业务场景

![](https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/Linux-firewall-1.png)

请求内部的 Web 服务:

Prerouting->Input->Web 服务

转发报文:

Prerouting->Forward->PostRouting->下一跳

Web 服务器返回响应

Output->Postrouting

### 链和表

`Prerouting`,`Input`,`Output`,`forward`,`Postrouting`五个关卡都有一定的规则，而这些规则是以链状的形式

![](https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/Linux-firewall-2.png)

**相同的规则放入同一张表中**

分为以下四张表：

1.filter 表：负责过滤功能，防火墙；内核模块：iptables_filter

2.nat 表：network address translation，网络地址转换功能；内核模块：iptable_nat

3.mangle 表：拆解报文，做出修改，并重新封装 的功能；iptable_mangle

4.raw 表：关闭 nat 表上启用的连接追踪机制；iptable_raw

**并不是每个关卡都有所有的表，每个关卡可以有的表如下**

Prerouting: raw->mangle->nat

Input: mangle->filter

Forward: mangle->filter

Output: raw->mangle->nat->filter

Postrouting: mangel->nat

每道关卡都会有几张表，下面是这些表的匹配优先级：

**raw –> mangle –> nat –> filter**

下面是完整的 iptables 工作流程

<img src="https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/20221029200141.png"/>

![](https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/Linux-firewall-3.png)

**注意：ip_forward 在 linux 中默认没有开启，可以在两块网卡之间转发数据**

```shell
# 查看当前有无开启ip_forward
cat /proc/sys/net/ipv4/ip_forward
# 临时开启ip_forward 将/proc/sys/net/ipv4/ip_forward里的内容改为1
sudo sysctl -w net.ipv4.ip_forward=1
# 永久开启ip_forward
修改/etc/sysctl.conf文件，添加net.ipv4.ip_forward = 1
sysctl -p /etc/sysctl.conf #立即生效
```

### 规则

![](https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/Linux-firewall-4.png)

上图展示了 INPUT 关卡的 filter 表

**Policy ACCEPT 表示该关卡默认的动作是 ACCEPT（放行）**

**匹配条件**

上图的源地址 source 和目的地址 destination，扩展匹配条件源端口 Source Port 和目的端口 Destination Port

**处理动作**

匹配成功后对该数据包的处理动作

**ACCEPT**：允许数据包通过。

**DROP**：直接丢弃数据包，不给任何回应信息，这时候客户端会感觉自己的请求泥牛入海了，过了超时时间才会有反应。

**REJECT**：拒绝数据包通过，必要时会给数据发送端一个响应的信息，客户端刚请求就会收到拒绝的信息。

**SNAT**：源地址转换，解决内网用户用同一个公网地址上网的问题。

**MASQUERADE**：是 SNAT 的一种特殊形式，适用于动态的、临时会变的 ip 上。

**DNAT**：目标地址转换。

**REDIRECT**：在本机做端口映射（两台机的端口转发是 SNAT 和 DNAT)。

```shell
# 本机80端口转发到8080
iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-ports 8080
```

**LOG**：在/var/log/messages 文件中记录日志信息，然后将数据包传递给下一条规则，也就是说除了记录以外不对数据包做任何其他操作，仍然让下一条规则去匹配。

**RETURN**：返回上一级链，如果已经是根链，执行默认策略

**其他 target**：转到用户自己定义的子链

### 常见操作

查看

```shell
iptables -t filter -nxvL --line
```

删除

```shell
iptabels -t filter -D INPUT [num]/[条件]
```

清空

```shell
iptables -t filter -F INPUT
```

修改

```shell
iptables -t filter -R INPUT [num] [条件] # 修改[num]条的规则
iptables -t filter -P DROP # 修改默认的策略为DROP
```

[条件]

```shell
-s 源地址 # 192.168.1.234  10.1.0.0/16
-d 目的地址
-p 协议tcp, udp, udplite, icmp, esp, ah, sctp
-j 执行的target
-i 指定网卡流入
-o 指定网卡流出
-m 指定扩展模块 # 常和-dport -sport一起使用 在指定-p的情况下可省略
	tcp/udp
		-dport 目的端口
		-sport 源端口
	icmp
		--icmp-type
-m multiport # 多个离散或连续端口 80,90 10000:11000
	--sports
	--dports
-m iprange
	–src-range # 192.168.1.127-192.168.1.148
	-dst-range
-m time # 多个限制的默认关系是与
	--timestart # 09:00:00
	--timestop
	--weekdays #6,7 Mon, Tue, Wed, Thu, Fri, Sat, Sun
	--monthdays # 22,23
-m connlimit # 连接限制
	–connlimit-above # 2
-m limit # 限制连接速率
	--limit # 3/minute /second /hour /day
	--limit-burst # 令牌桶
--tcp-flags # 匹配tcp的flags SYN,ACK,FIN,RST,URG,PSH
	--tcp-flags # SYN,ACK,FIN,RST,URG,PSH SYN 第一部分是需要匹配的标志位 第二部分是哪些位必须为1
	--syn # === SYN,RST,ACK,FIN  SYN
```

**补充**

在我们在 80 端口发起对服务器的请求的时候，服务器会以 80 端口回复我们的请求，我们理应放行 80 端口，但是，如果别人也可以主动通过 80 端口向我们发起连接，如果我们想屏蔽别人的主动连接，则可以使用`-m state`来完成

state 规定所有的报文可以分为 5 种状态

**NEW**：连接中的第一个包，状态就是 NEW，我们可以理解为新连接的第一个包的状态为 NEW。

**ESTABLISHED**：我们可以把 NEW 状态包后面的包的状态理解为 ESTABLISHED，表示连接已建立。

**RELATED**：从字面上理解 RELATED 译为关系，例如 ftp 连接中数据连接是控制连接的关系包。

**INVALID**：如果一个包没有办法被识别，或者这个包没有任何状态，那么这个包的状态就是 INVALID。

**UNTRACKED**：报文的状态为 untracked 时，表示报文未被追踪，当报文的状态为 Untracked 时通常表示无法找到相关的连接。

```shell
-m state --state RELATED,ESTABLISHED # 只放行被标记为RELATED,ESTABLISHED的包
```

### 自定义链

将针对相同服务的规则放入同一个自定义链中，方便规则的管理

![](https://hanshansite-1307452666.cos.ap-shanghai.myqcloud.com/site-img/Linux-firewall-5.png)

```shell
# 新建一个自定义链
iptables -t filter -N IN_WEB # 新建一个IN_WEB的filter自定义链
# 添加一条规则
iptables -t filter -I IN_WEB -s 192.168.1.245 -j REJECT
# 将自定义链添加到默认链引用
iptables -t filter -p tcp --dport 40000 -j IN_WEB
# 链重命名
iptables -t filter -E IN_WEB WEB # 会自动更改所有被引用的地方
# 删除链 注意：必须清空链的引用和规则
iptables -t filter -X WEB
# 删除规则
iptables -t filter -D WEB -s 192.168.1.245 -j REJECT/1
iptables -t filter -D INPUT -p tcp --dport 40000 -j WEB
```
