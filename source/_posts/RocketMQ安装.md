---
title: RocketMQ安装
tags: [RocketMQ]
categories: 大数据
date: 2020-01-21 10:00:00
---

# 介绍

RocketMQ是一款分布式、队列模型的消息中间件，是由阿里巴巴设计的，具有以下特点：

- 支持严格的消息顺序
- 支持Topic与Queue两种模式
- 亿级消息堆积能力
- 比较友好的分布式特性
- 同时支持Push与Pull方式消费消息
- RocketMQ是纯java编写，基于通信框架Netty

### **相关术语** 

1. `Producer`：消息生产者，负责产生消息，一般由业务系统负责产生消息。
2. `Producer Group`：一类Producer的集合名称，这类Producer通常发送一类消息，且发送逻辑一致。
3. `Consumer`：消息消费者，负责消费消息，一般是后台系统负责异步消费。
4. `Push Consumer`：Consumer的一种，应用通常向Consumer对象注册一个Listener接口，一旦收到消息，Consumer对象立 刻回调Listener接口方法。
5. `Pull Consumer`：Consumer的一种，应用通常主动调用Consumer的拉消息方法从Broker拉消息，主动权由应用控制。
6. `Consumer Group`：一类Consumer的集合名称，这类Consumer通常消费一类消息，且消费逻辑一致。
7. `Broker`：消息中转角色，负责存储消息，转发消息，一般也称为Server。
8. `Nameserver`：专为RocketMQ设计的轻量级名称服务。集群中Nameserver互相独立，彼此没有通信关系，单台Nameserver挂掉，不影响其他Nameserver，即使全部挂掉，也不影响业务系统使用。而且Nameserver不会有频繁的读写，所以性能开销非常小，稳定性很高。
9. `广播消费`：一条消息被多个Consumer消费，即使这些Consumer属于同一个Consumer Group，消息也会被Consumer Group中的每个Consumer都消费一次，广播消费中的Consumer Group概念可以认为在消息划分方面无意义。
10. `集群消费`：一个Consumer Group中的Consumer实例平均分摊消费消息。例如某个Topic有9条消息，其中一个Consumer Group有3个实例（可能是3个进程，或者3台机器），那么每个实例只消费其中的3条消息。
11. `顺序消息`：消费消息的顺序要同发送消息的顺序一致，在RocketMQ中，主要指的是局部顺序，即一类消息为满足顺序性，必须Producer单线程顺序发送，且发送到同一个队列，这样Consumer就可以按照Producer发送的顺序去消费消息。
12. `普通顺序消息`：顺序消息的一种，正常情况下可以保证完全的顺序消息，但是一旦发生通信异常，Broker重启，由于队列总数发生变化，哈希取模后定位的队列会变化，产生短暂的消息顺序不一致。如果业务能容忍在集群异常情况（如某个Broker宕机或者重启）下，消息短暂的乱序，使用普通顺序方式比较合适。
13. `严格顺序消息`：顺序消息的一种，无论正常异常情况都能保证顺序，但是牺牲了分布式Failover特性，即Broker集群中只要有一台机器不可用，则整个集群都不可用，服务可用性大大降低。如果服务器部署为同步双写模式，此缺陷可通过备机自动切换为主避免，不过仍然会存在几分钟的服务不可用。
14. `Message Queue`：在RocketMQ中，所有消息队列都是持久化，长度无限的数据结构，所谓长度无限是指队列中的每个存储单元都是定长，访问其中的存储单元使用Offset来访问，offset为java long类型，64位，理论上在100年内不会溢出，所以认为是长度无限，另外队列中只保存最近几天的数据，之前的数据会按照过期时间来删除。也可以认为Message Queue是一个长度无限的数组，offset就是下标。
15. `异步复制`：消息写入master节点，再由master节点异步复制到slave节点，类似mysql中的master-slave机制。
16. `同步双写`：消息同时写入master节点和slave节点。
17. `异步刷盘`：Broker的一种持久化策略，消息写入pagecache后，直接返回。由异步线程负责将pagecache写入硬盘。
18. `同步刷盘`：Broker的一种持久化策略，消息写入pagecache后，由同步线程将pagecache写入硬盘后，再返回。

### **集群部署模式**

RocketMQ作为消息中间件，其主要功能为消息的Publish/Subscribe。而Broker担任的消息转发和存储功能，其部署方式有很多种：

1. `单Master`

优点：除了配置简单没什么优点。

缺点：不可靠，该机器重启或宕机，将导致整个服务不可用。

2. `多Master`

优点：配置简单，性能最高。

缺点：可能会有少量消息丢失，单台机器重启或宕机期间，该机器下未被消费的消息在机器恢复前不可订阅，影响消息实时性。

3. `异步多Master多Slave`

每个Master配一个Slave，有多对Master-Slave，集群采用异步复制方式，主备有短暂消息延迟，毫秒级。

优点：性能同多Master几乎一样，实时性高，主备间切换对应用透明，不需人工干预。

缺点：Master宕机或磁盘损坏时会有少量消息丢失。

4. `同步多Master多Slave`

每个Master配一个Slave，有多对Master-Slave，集群采用同步双写方式，主备都写成功，向应用返回成功。

优点：服务可用性与数据可用性非常高。

缺点：性能比异步集群略低，当前版本主宕备不能自动切换为主。

# 单机安装

**jdk1.8安装**

[**jdk安装跳转**](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/)

**下载rocketmq**

```bash
wget https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.6.0/rocketmq-all-4.6.0-bin-release.zip

unzip rocketmq-all-4.6.0-bin-release.zip
mv rocketmq-all-4.6.0-bin-release /data/rocketmq
cd /data/rocketmq
```

**启动Nameserver**

```bash
# 要先启动Nameserver，之后启动Broker
nohup sh bin/mqnamesrv &
tail -f ~/logs/rocketmqlogs/namesrv.log
```

**启动Broker**

```bash
# 内存需要4G以上，否正启动报错，修改启动文件
nohup sh bin/mqbroker -n localhost:9876 &
tail -f ~/logs/rocketmqlogs/broker.log 
```

**Broker启动内存报错**

```bash
vim bin/runbroker.sh
# 修改
```

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m"
```

```bash
vim bin/runserver.sh 
# 修改
```

```bash
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

**测试消息发送和接收**

```bash
# 声明Nameserver地址
export NAMESRV_ADDR=localhost:9876
# 发送消息
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
# 接收消息
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

**关闭服务**

```bash
# 要先关闭broker，之后关闭Nameserver
sh bin/mqshutdown broker
sh bin/mqshutdown namesrv
```

常用命令
除了上面几个命令之外，还有如下一些较常用的命令，ip请以实际为准：

```bash
# 查看集群情况： 
./mqadmin clusterList -n 127.0.0.1:9876
# 查看 broker 状态： 
./mqadmin brokerStatus -n 127.0.0.1:9876 -b 172.20.1.138:10911
# 查看 topic 列表： 
./mqadmin topicList -n 127.0.0.1:9876
# 查看 topic 状态： 
./mqadmin topicStatus -n 127.0.0.1:9876 -t MyTopic (换成想查询的 topic)
# 查看 topic 路由： 
./mqadmin topicRoute -n 127.0.0.1:9876 -t MyTopic
```

