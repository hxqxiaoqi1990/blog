---
title: zookeeper 集群搭建
tags: [zookeeper]
categories: 中间件
date: 2020-03-26 15:27:43
---

# 介绍

**Zookeeper原理简介**

ZooKeeper是一个开放源码的分布式应用程序协调服务，它包含一个简单的原语集，分布式应用程序可以基于它实现同步服务，配置维护和命名服务等。

**Zookeeper设计目的**

- 最终一致性：client不论连接到那个Server，展示给它的都是同一个视图。
- 可靠性：具有简单、健壮、良好的性能、如果消息m被到一台服务器接收，那么消息m将被所有服务器接收。
- 实时性：Zookeeper保证客户端将在一个时间间隔范围内获得服务器的更新信息，或者服务器失效的信息。但由于网络延时等原因，Zookeeper不能保证两个客户端能同时得到刚更新的数据，如果需要最新数据，应该在读数据之前调用sync()接口。
- 等待无关（wait-free）：慢的或者失效的client不得干预快速的client的请求，使得每个client都能有效的等待。
- 原子性：更新只能成功或者失败，没有中间状态。
- 顺序性：包括全局有序和偏序两种：全局有序是指如果在一台服务器上消息a在消息b前发布，则在所有Server上消息a都将在消息b前被发布；偏序是指如果一个消息b在消息a后被同一个发送者发布，a必将排在b前面。

**Zookeeper工作原理**

在zookeeper的集群中，各个节点共有下面3种角色和4种状态：

角色：leader,follower,observer
状态：leading,following,observing,looking

Zookeeper的核心是原子广播，这个机制保证了各个Server之间的同步。实现这个机制的协议叫做Zab协议（ZooKeeper Atomic Broadcast protocol）。Zab协议有两种模式，它们分别是恢复模式（Recovery选主）和广播模式（Broadcast同步）。当服务启动或者在领导者崩溃后，Zab就进入了恢复模式，当领导者被选举出来，且大多数Server完成了和leader的状态同步以后，恢复模式就结束了。状态同步保证了leader和Server具有相同的系统状态。

为了保证事务的顺序一致性，zookeeper采用了递增的事务id号（zxid）来标识事务。所有的提议（proposal）都在被提出的时候加上了zxid。实现中zxid是一个64位的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个新的epoch，标识当前属于那个leader的统治时期。低32位用于递增计数。

每个Server在工作过程中有4种状态：

LOOKING：当前Server不知道leader是谁，正在搜寻。

LEADING：当前Server即为选举出来的leader。

FOLLOWING：leader已经选举出来，当前Server与之同步。

OBSERVING：observer的行为在大多数情况下与follower完全一致，但是他们不参加选举和投票，而仅仅接受(observing)选举和投票的结果。

# 安装

## 实验环境

<font color='32CD32'>配置hosts文件，让集群服务器间互相使用主机名访问</font>

|     主机IP     | 主机名 |    安装服务     | 安装目录 |
| :------------: | :----: | :-------------: | :------: |
| 192.168.40.100 |  zk0   | jdk8，zookeeper |   /opt   |
| 192.168.40.101 |  zk1   | jdk8，zookeeper |   /opt   |
| 192.168.40.102 |  zk2   | jdk8，zookeeper |   /opt   |

## jdk8 安装

所有节点安装：[jdk 安装教程](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/)

## zookeeper 安装

<font color='32CD32'>在 192.168.40.100上安装</font>

```bash
# 下载
wget https://mirror.bit.edu.cn/apache/zookeeper/zookeeper-3.5.7/apache-zookeeper-3.5.7.tar.gz

tar xf apache-zookeeper-3.5.7.tar.gz -c /opt

# 重命名配置文件，zoo_sample.cfg为配置文件模板，默认不生效
cd /opt/apache-zookeeper-3.5.7/conf
cp zoo_sample.cfg zoo.cfg

# 创建日志和数据目录
mkdir -p /opt/apache-zookeeper-3.5.7/{logs,data}
```

<font color='32CD32'>修改 zoo.cfg</font>

```bash
# tickTime这个时间是作为zookeeper服务器之间或客户端与服务器之间维持心跳的时间间隔,也就是说每个tickTime时间就会发送一个心跳。
tickTime=2000

# initLimit这个配置项是用来配置zookeeper接受客户端（这里所说的客户端不是用户连接zookeeper服务器的客户端,而是zookeeper服务器集群中连接到leader的follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。
initLimit=10

# syncLimit这个配置项标识leader与follower之间发送消息,请求和应答时间长度,最长不能超过多少个tickTime的时间长度,总的时间长度就是5*2000=10秒。
syncLimit=5

# 日志存储目录
dataLogDir=/opt/apache-zookeeper-3.5.7/logs

# 数据存储目录
dataDir=/opt/apache-zookeeper-3.5.7/data

# 端口
clientPort=2181

# 这个参数指定了需要保留的文件数目。默认是保留3个。
autopurge.snapRetainCount=500

# 3.4.0及之后版本，ZK提供了自动清理事务日志和快照文件的功能，这个参数指定了清理频率，单位是小时，需要配置一个1或更大的整数，默认是0，表示不开启自动清理功能。
autopurge.purgeInterval=24

# server.A=B:C:D中的A是一个数字,表示这个是第几号服务器,B是这个服务器的IP地址，C第一个端口用来集群成员的信息交换,表示这个服务器与集群中的leader服务器交换信息的端口，D是在leader挂掉时专门用来进行选举leader所用的端口。
server.1=192.168.40.100:2888:3888
server.2=192.168.40.101:2888:3888
server.3=192.168.40.102:2888:3888
```

<font color='32CD32'>创建集群标识文件</font>

```bash
# 这里的"1"，就是zoo.cfg文件中server.1的"1"
echo "1" > /opt/apache-zookeeper-3.5.7/data/myid
```

<font color='32CD32'>把zookeeper目录传到其它节点</font>

```bash
scp -r /opt/apache-zookeeper-3.5.7 root@192.168.40.101:/opt
scp -r /opt/apache-zookeeper-3.5.7 root@192.168.40.102:/opt
```

<font color='32CD32'>在其它节点上操作</font>

1. 创建数据目录和日志目录，与192.168.40.100上操作一样
2. 创建集群标识文件，与配置文件中对应

<font color='32CD32'>启动</font>

```bash
/opt/apache-zookeeper-3.5.7/bin/zkServer.sh start

# 查看状态
/opt/apache-zookeeper-3.5.7/bin/zkServer.sh status
```

<font color='32CD32'>登录</font>

```bash
# 登录后，可以使用help查看操作命令
/opt/apache-zookeeper-3.5.7/bin/zkCli.sh
```

## zkui 安装

<font color='32CD32'>提请安装好 git 和 maven</font>

```bash
# 下载
git clone https://github.com/DeemOpen/zkui.git

# 编译
cd zkui
mvn clean install

# 修改配置文件，修改zk地址和登录的账号密码
vim config.cfg

# 启动
java -jar target/zkui-2.0-SNAPSHOT-jar-with-dependencies.jar

# 访问
curl http://192.168.40.100:9090
```

