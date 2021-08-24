---
title: iptables常用命令
tags: [iptables]
categories: linux基础
date: 2020-04-02 21:29:43
---

**1、删除现有规则**

在开始建立新的规则之前，您可能需要清理所有默认规则和现有规则。使用**iptables**如下所示的命令来做这个。

```bash
iptables -F 　　　　　　#警告：当前命令后将切断linux对外的一切端口请求，请确保你还能连接到vnc或主机上
(or)
iptables --flush

# 删除单条记录
iptables -nL --line-number
iptables -D INPUT 2
```

**2、设置默认链策略**

默认的**iptables**策略是ACCEPT。将此更改为如下所示。

```bash
iptables -P INPUT DROP
iptables -P FORWARD DROP
iptables -P OUTPUT DROP
```

当将INPUT和OUTPUT默认策略作为DROP时，对于每一个防火墙规则要求都应该定义两条规则。即一个INPUT和一个OUTPUT。
如果信任内部用户，则可以忽略上面OUTPUT的设置。默认情况下，不要丢弃所有OUTPUT的数据包。在这种情况下，对于每一个防火墙规则的要求，你只需要定义一个规则。即定义规则传入，因为传出是接受所有数据包。

**3、阻止一个特定的IP地址**

在我们做其他规则前，如果你想阻止一个特定的IP地址，你应该先做如下所示。当您在日志文件中找到特定的IP地址时发现一些奇怪的活动，并希望在进一步研究时暂时阻止该IP地址，这将很有帮助。

```bash
BLOCK_THIS_IP="192.168.1.108"
iptables -A INPUT -s "$BLOCK_THIS_IP" -j DROP
```

你也可以使用下面的一种规则。

```bash
iptables -A INPUT -i eth0 -s "$BLOCK_THIS_IP" -j DROP           //规则禁止这个IP地址对我们服务器eth0网卡的所有连接
iptables -A INPUT -i eth0 -p tcp -s "$BLOCK_THIS_IP" -j DROP   //规则只禁止这个IP地址对我们服务器eth0网卡的tcp协议的连接
```

**4.允许所有传入SSH**

以下规则允许eth0接口上的所有传入的SSH连接。

```bash
iptables -A INPUT  -i eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

**5.只能从一个特定的网络允许传入的SSH**

下面的规则只允许从网络192.168.100.X传入SSH连接。

```bash
iptables -A INPUT  -i eth0 -p tcp -s 192.168.100.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

**6.允许传入的HTTP和HTTPS**

以下规则允许所有传入的网络流量。即HTTP流量的端口80。

```bash
iptables -A INPUT  -i eth0 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 80 -m state --state ESTABLISHED -j ACCEPT
```

下面的规则允许所有传入安全的网络流量。即HTTPS流量的端口443。

```bash
iptables -A INPUT  -i eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
```

**7.相结合多个规则使用多端口**

当你不是写为每个端口单独的规则，而是从外面的世界多个端口传入的连接，可以在一起使用多端口扩展。

下面的示例允许所有传入SSH，HTTP和HTTPS流量。

```bash
iptables -A INPUT  -i eth0 -p tcp -m multiport --dports 22,80,443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp -m multiport --sports 22,80,443 -m state --state ESTABLISHED -j ACCEPT
```

**8.允许OUTPUT SSH**

以下规则允许传出ssh连接。也就是说当你从内ssh到外部服务器。

```bash
iptables -A OUTPUT -o eth0 -p tcp --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT  #允许双方新建立的OUTPUT链通信
iptables -A INPUT  -i eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT      #只允许双方已经建立的INPUT链通信 
```

**9.允许拨出SSH到特定网络**

下面的规则只允许特定的网络传出的ssh连接。即你的SSH只有从内部网络192.168.100.0/24。

```bash
iptables -A OUTPUT -o eth0 -p tcp -d 192.168.100.0/24 --dport 22 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT  -i eth0 -p tcp --sport 22 -m state --state ESTABLISHED -j ACCEPT
```

**10.允许拨出HTTPS**

以下规则允许传出安全的Web流量。当你想允许互联网流量的用户，这是很有帮助。在服务器上，当你想使用wget从外部下载一些文件，这些规则也有帮助。

```bash
iptables -A OUTPUT -o eth0 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A INPUT  -i eth0 -p tcp --sport 443 -m state --state ESTABLISHED -j ACCEPT
```

注：对于传出HTTP Web流量，增加两个额外的规则就像上面，并改变443 80。

**11.负载均衡传入的Web流量**

您也可以加载使用**iptables**防火墙规则平衡传入的网络流量。

```bash
This uses the iptables nth extension. The following example load balances the HTTPS traffic to three different ip-address. For every 3th packet, it is load balanced to the appropriate server (using the counter 0).
iptables -A PREROUTING -i eth0 -p tcp --dport 443 -m state --state NEW -m nth --counter 0 --every 3 --packet 0 -j DNAT --to-destination 192.168.1.101:443
iptables -A PREROUTING -i eth0 -p tcp --dport 443 -m state --state NEW -m nth --counter 0 --every 3 --packet 1 -j DNAT --to-destination 192.168.1.102:443
iptables -A PREROUTING -i eth0 -p tcp --dport 443 -m state --state NEW -m nth --counter 0 --every 3 --packet 2 -j DNAT --to-destination 192.168.1.103:443
```

**12.允许从外部到内部Ping**

以下规则允许外部用户能够ping您的服务器。

```bash
iptables -A INPUT  -p icmp --icmp-type echo-request -j ACCEPT
iptables -A OUTPUT -p icmp --icmp-type echo-reply -j ACCEPT
```

**13.允许从内部到外部ping**

以下规则允许您从内部ping到任何外部服务器。

```bash
iptables -A OUTPUT -p icmp --icmp-type echo-request -j ACCEPT
iptables -A INPUT  -p icmp --icmp-type echo-reply -j ACCEPT
```

**14.允许环回访问**

您应该允许在服务器上进行完全环回访问。即使用127.0.0.1访问

```bash
iptables -A INPUT -i lo -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
```

**15.允许内部网络到外部网络**

在防火墙服务器上，一个以太网卡连接到外部网络，另一个以太网卡连接到内部服务器，请使用以下规则允许内部网络与外部网络通信。

```bash
#在此示例中，eth1连接到外部网络（互联网），eth0连接到内部网络（例如：192.168.1.x）。
iptables -A FORWARD -i eth0 -o eth1 -j ACCEPT
```

**16.允许出站DNS**

以下规则允许传出DNS连接。

```bash
iptables -A OUTPUT -p udp -o eth0 --dport 53 -j ACCEPT
iptables -A INPUT  -p udp -i eth0 --sport 53 -j ACCEPT
```

**17.允许NIS连接**

如果您正在运行NIS（网络信息服务）来管理您的用户帐户，您应该允许NIS连接。即使允许SSH连接，如果您不允许NIS相关的ypbind连接，用户将无法登录。

NIS端口是动态的。即当ypbind启动时，它分配端口。（*ypbind*是定义NIS服务器的客户端进程。）

```bash
#首先做一个rpcinfo -p如下所示并获取端口号。在此示例中，它使用端口853和850。
rpcinfo -p | grep ypbind

#现在允许到端口111的入站连接，以及ypbind使用的端口。
iptables -A INPUT -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -p udp --dport 111 -j ACCEPT
iptables -A INPUT -p tcp --dport 853 -j ACCEPT
iptables -A INPUT -p udp --dport 853 -j ACCEPT
iptables -A INPUT -p tcp --dport 850 -j ACCEPT
iptables -A INPUT -p udp --dport 850 -j ACCEPT
```

以上不会工作，当你重新启动ypbind，因为它在不同的时间会有不同的端口号。

有两个解决方案：1）使用静态ip地址为您的NIS，或2）使用一些聪明的shell脚本技术来自动抓取动态端口号从“rpcinfo -p”命令输出，并使用上述**iptables**规则。

**18.允许Rsync来自特定网络**

以下规则仅允许来自特定网络的rsync。

```bash
iptables -A INPUT  -i eth0 -p tcp -s 192.168.101.0/24 --dport 873 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 873 -m state --state ESTABLISHED -j ACCEPT
```

**19.仅允许来自特定网络的MySQL连接**

如果你正在运行MySQL，通常你不想允许从外部直接连接。在大多数情况下，您可能在运行MySQL数据库的同一服务器上运行Web服务器。

但是DBA和开发人员可能需要使用MySQL客户端从他们的笔记本电脑和桌面直接登录到MySQL。在这种情况下，您可能希望允许内部网络直接与MySQL通信，如下所示。

```bash
iptables -A INPUT  -i eth0 -p tcp -s 192.168.100.0/24 --dport 3306 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 3306 -m state --state ESTABLISHED -j ACCEPT
```

**20.允许Sendmail或Postfix流量**

以下规则允许邮件通信。它可以是sendmail或postfix。

```bash
iptables -A INPUT  -i eth0 -p tcp --dport 25 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 25 -m state --state ESTABLISHED -j ACCEPT
```

**21.允许IMAP和IMAPS**

以下规则允许IMAP / IMAP2流量。

```bash
iptables -A INPUT -i eth0 -p tcp --dport 143 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 143 -m state --state ESTABLISHED -j ACCEPT
```

以下规则允许IMAPS流量。

```bash
iptables -A INPUT -i eth0 -p tcp --dport 993 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 993 -m state --state ESTABLISHED -j ACCEPT
```

**22.允许POP3和POP3S**

以下规则允许POP3访问。

```bash
iptables -A INPUT  -i eth0 -p tcp --dport 110 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 110 -m state --state ESTABLISHED -j ACCEPT
```

以下规则允许POP3S访问。

```bash
iptables -A INPUT  -i eth0 -p tcp --dport 995 -m state --state NEW,ESTABLISHED -j ACCEPT
iptables -A OUTPUT -o eth0 -p tcp --sport 995 -m state --state ESTABLISHED -j ACCEPT 
```

**23.防止DoS攻击**

以下**iptables**规则将帮助您防止对您的Web服务器的拒绝服务（DoS）攻击。

```bash
iptables -A INPUT -p tcp --dport 80 -m limit --limit 25/minute --limit-burst 100 -j ACCEPT
```

在上面的例子中：
　　-m limit：这使用限制**iptables**扩展
　　-limit 25 /分钟：每分钟最多只能连接25个请求包。根据您的具体要求更改此值
　　-limit-burst 100：此值指示仅在连接的总数达到limit之后才实施限制/分钟

注意“-m limit”只匹配数据包而不是连接，所以上方例子中你将匹配25包每分钟。

如果是想限制每分钟下connect次数呢。限制连接数的解决方案是使用connlimit匹配。

```bash
iptables -A INPUT -p tcp –syn –dport 80 -m connlimit –connlimit-above 15 –connlimit-mask 32 -j REJECT –reject-with tcp-reset
```

它将拒绝来自一个源IP的15以上的连接 - 一个很好的规则来保护Web服务器。

此外，当与“hashlimit” 结合后在保护免受DDoS攻击时效果更好。

使用“limit”匹配，您可以限制每个时间间隔的数据包的全局速率，但是使用“hashlimit”，您可以限制每个IP，每个组合IP +端口等。

所以一个Web服务器的例子将是这样：

```bash
iptables -A INPUT -p tcp –dport 80 -m hashlimit –hashlimit 45/sec –hashlimit-burst 60 –hashlimit-mode srcip–hashlimit-name DDOS –hashlimit-htable-size 32768 –hashlimit-htable-max 32768 –hashlimit-htable-gcinterval 1000 –hashlimit-htable-expire 100000 -j ACCEPT
```

**24. 端口转发**

例：将来自422端口的流量全部转到22端口。

这意味着我们既能通过422端口又能通过22端口进行ssh连接。启用DNAT转发。

```bash
iptables -t nat -A PREROUTING -p tcp -d 192.168.102.37 --dport 422 -j DNAT --to 192.168.102.37:22
```

除此之外，还需要允许连接到422端口的请求

```bash
iptables -A INPUT -i eth0 -p tcp --dport 422 -m state --state NEW,ESTABLISHED -j ACCEPTiptables -A OUTPUT                   -o eth0 -p tcp --sport 422 -m state --state ESTABLISHED -j ACCEPT
```

**25. 记录丢弃的数据表**

```bash
iptables -N LOGGING   #1.新建名为LOGGING的链
iptables -A INPUT -j LOGGING  #2.将所有来自INPUT链中的数据包跳转到LOGGING链中
iptables -A LOGGING -m limit --limit 2/min -j LOG --log-prefix "IPTables Packet Dropped: " --log-level 7  #3.为这些包自定义个前缀，命名为”IPTables Packet Dropped”
iptables -A LOGGING -j DROP   #4.丢弃这些数据包 
```

**26. 例子**

```bash
[root@iZ233pgwujzZ ~]# iptables-save
# Generated by iptables-save v1.4.7 on Wed May  6 14:27:04 2020
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [670583527:153247221001]
# 允许指定IP访问所有服务
-A INPUT -s 172.16.30.0/24 -p tcp -j ACCEPT
# 访问端口范围
-A INPUT -s 192.168.40.1/32 -p tcp -m tcp --dport 1:65535 -j ACCEPT
# 意思是允许进入的数据包只能是刚刚我发出去的数据包的回应，ESTABLISHED：已建立的链接状态。RELATED：该数据包与本机发出的数据包有关。
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT 
# 允许所有IP访问22端口
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
# 允许icmp和io
-A INPUT -p icmp -j ACCEPT 
-A INPUT -i lo -j ACCEPT 
# 这两内条的意思是在INPUT表和FORWARD表中拒绝所有其他不符合上述任何一条规则的容数据包。并且发送一条host prohibited的消息给被拒绝的主机。
-A INPUT -j REJECT --reject-with icmp-host-prohibited 
-A FORWARD -j REJECT --reject-with icmp-host-prohibited 
COMMIT
# Completed on Wed May  6 14:27:04 2020
```

