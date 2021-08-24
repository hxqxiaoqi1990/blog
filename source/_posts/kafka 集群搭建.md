---
title: kafka 集群搭建
tags: [kafka]
categories: 大数据
date: 2020-03-25 17:07:00
---

# 介绍

kafka 是一个分布式的消息队列系统，消息以topic分类传输，生产者往topic发送消息时，消息会被分散到topic的不同分区中，消费者以组的形式消费topic中的数据，一个topic的一个分区，只能被同一个组里的一个消费者消费，但同一个消费者，可以消费多个分区数据，每个组自己维护topic每个分区的消费偏移量。

kafka 不能保证不同分区消息的消费顺序，因此若对于消息消费顺序有要求，必须确保该类消息处于同一分区，可以通过发送消息时，指定相同key来处理。

# 搭建

## 实验环境

<font color='32CD32'>配置hosts文件，让集群服务器间互相使用主机名访问</font>

|    服务器ip    | 主机名 |          服务          | 安装目录 |
| :------------: | :----: | :--------------------: | :------: |
| 192.168.40.100 | kafka0 | kafka，zookeeper，jdk8 |   /opt   |
| 192.168.40.101 | kafka1 | kafka，zookeeper，jdk8 |   /opt   |
| 192.168.40.102 | kafka2 | kafka，zookeeper，jdk8 |   /opt   |

## jdk8 安装

所有节点安装：[jdk 安装教程](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/)

## zookeeper 集群安装

[zk集群安装教程](https://hxqxiaoqi.gitee.io/2020/03/26/zookeeper%20集群搭建/)

## kafka 集群安装

<font color='32CD32'>在 kafka0 主机上配置</font>

```bash
# 下载
wget http://archive.apache.org/dist/kafka/2.4.0/kafka_2.13-2.4.0.tgz

tar xf kafka_2.13-2.4.0.tgz -C /opt
cd /opt/kafka_2.13-2.4.0/config
```

<font color='32CD32'>修改 server.properties 文件</font>

```bash
# 集群识别号，每台服务器上的kafka都不能相同
broker.id=0

# 监听地址，填写本机地址
listeners=PLAINTEXT://kafka0:9092

# 外网监听地址，返回给客户端的地址，填写本机地址
advertised.listeners=PLAINTEXT://kafka0:9092

# 数据存储目录,默认/tmp下，建议不要设置到/tmp下，否则有可能服务崩溃
log.dirs=/data/kafka-logs

# zookeeper地址
zookeeper.connect=kafka0:2181,kafka1:2181,kafka2:2181
```

<font color='32CD32'>修改/opt/kafka_2.13-2.4.0/bin/kafka-server-start.sh 文件</font>

```bash
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-Xmx1G -Xms1G"
    # 添加以下这条设置，用于kafka-manager监控
    export JMX_PORT="9999"
fi
```

<font color='32CD32'>复制 kafka 目录到其它集群服务器上，并修改相应配置</font>

```bash
# 如果没有免密，需要输入密码
scp -r /opt/kafka_2.13-2.4.0 root@kafka1:/opt
scp -r /opt/kafka_2.13-2.4.0 root@kafka1:/opt
```

<font color='32CD32'>启动与关闭</font>

```bash
# 前台启动
/opt/kafka_2.13-2.4.0/bin/kafka-server-start.sh /opt/kafka_2.13-2.4.0/config/server.properties

# 后台启动
/opt/kafka_2.13-2.4.0/bin/kafka-server-start.sh -daemon /opt/kafka_2.13-2.4.0/config/server.properties

# 关闭
/opt/kafka_2.13-2.4.0/bin/kafka-server-stop.sh
```

## kafka-manager安装

kafka-manager用于监控kafka集群状态

```bash
wget https://github.com/yahoo/CMAK/archive/3.0.0.4.tar.gz
```



## 常用命令

```bash
cd /opt/kafka_2.13-2.4.0/bin/

# 创建topic，--replication-facto：副本，--partitions：分区
./kafka-topics.sh --create --zookeeper kafka0:2181,kafka1:2181,kafka2:2181 --replication-factor 3 --partitions 1 --topic kafka-test

# 查看top信息
./kafka-topics.sh --describe --zookeeper kafka0:2181,kafka1:2181,kafka2:2181 --topic kafka-test

# 启动生产者
./kafka-console-producer.sh --broker-list kafka0:9092,kafka1:9092,kafka2:9092 --topic kafka-test

# 启动消费者，--from-beginning：从头开始消费，不加 --from-beginning 从最新开始消费
./kafka-console-consumer.sh --bootstrap-server kafka0:9092,kafka1:9092,kafka2:9092 --topic kafka-test --from-beginning

# 查看topic列表
./kafka-topics.sh --list --zookeeper kafka0:2181,kafka1:2181,kafka2:2181

# 删除topic
./kafka-topics.sh --zookeeper kafka0:2181,kafka1:2181,kafka2:2181 --delete  --topic kafka-test

# 查看topic策略
bin/kafka-configs.sh --zookeeper 192.168.10.111:2181,192.168.10.112:2181,192.168.10.113:2181 --describe --entity-type topics --entity-name TOPIC_DW_USER_STATS

# 设置topic策略，保存时间
bin/kafka-configs.sh --zookeeper 192.168.10.111:2181,192.168.10.112:2181,192.168.10.113:2181 --entity-type topics --entity-name TOPIC_DW_USER_STATS --alter --add-config retention.ms=10000

# 取消topic策略
bin/kafka-configs.sh --zookeeper 192.168.10.111:2181,192.168.10.112:2181,192.168.10.113:2181 --entity-type topics --entity-name TOPIC_DW_USER_STATS --alter --delete-config retention.ms

# 查看所有消费者
bin/kafka-consumer-groups.sh --bootstrap-server 192.168.10.111:9092,192.168.10.112:9092,192.168.10.113:9092 --list

# 查看指定消费者信息
bin/kafka-consumer-groups.sh --describe --bootstrap-server 192.168.10.111:9092,192.168.10.112:9092,192.168.10.113:9092 --group member-star-member-count-job
```

## 动态添加partition副本

```bash
# test.json文件自己编写，参照以下例子
/data/kafka_2.13-2.4.0/bin/kafka-reassign-partitions.sh --zookeeper 192.168.10.111:2181,192.168.10.112:2181,192.168.10.113:2181 --reassignment-json-file test.json --execute
```

test.json例子

```json
{"version": 1,"partitions": [
    {"topic": "mytest1","partition": 0,"replicas": [1,0,3]},
    {"topic": "mytest2","partition": 0,"replicas": [3,0,1]}
]}

# partition根据实际topic分区填写
# replicas根据broker数填写，其中数字为broker ID，其中对顺也有要求，[1,0,3]中第一个数尽量与原分区Leader一致，否则	Preferred Leader将为false，但不影响使用

```

__consumer_offsets调整副本数示例：

```json
{"version":1,"partitions":
[
{"topic":"__consumer_offsets","partition":0,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":1,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":2,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":3,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":4,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":5,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":6,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":7,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":8,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":9,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":10,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":11,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":12,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":13,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":14,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":15,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":16,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":17,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":18,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":19,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":20,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":21,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":22,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":23,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":24,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":25,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":26,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":27,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":28,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":29,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":30,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":31,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":32,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":33,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":34,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":35,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":36,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":37,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":38,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":39,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":40,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":41,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":42,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":43,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":44,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":45,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":46,"replicas":[0,1,3]},
{"topic":"__consumer_offsets","partition":47,"replicas":[1,3,0]},
{"topic":"__consumer_offsets","partition":48,"replicas":[3,0,1]},
{"topic":"__consumer_offsets","partition":49,"replicas":[0,1,3]}
]}
```



## server.properties 配置文件详解

```bash
# broker在集群中的唯一标识，不能为负数
broker.id

# 数据存放的目录，这个目录可以配置为“，”逗号分割的表达式，下面的num.io.threads要大于这个目录的个数，如果配置多个目录，新创建topic消息持久化的地方是，当前以逗号分割的目录中，分区数最少的那一个
log.dirs

# 相当于下面的host.name+port
listeners	PLAINTEXT://hostname:port

# 外网客户端访问返回的地址 PLAINTEXT://hostname:port
advertised.listeners	

# broker的主机地址，若是设置了，那么会绑定到这个地址上，若是没有，会绑定到所有的接口上，并将其中之一发送到ZK
host.name	

# 监听端口
port

# 消息最大字节
message.max.bytes

# broker处理网络消息的最大线程数
num.network.threads

# broker处理磁盘IO的线程数，数值应该大于你的目录数
num.io.threads

# 处理后台任务的线程数，例如过期消息文件的删除等
background.threads

# 等待IO线程处理的请求队列最大数，若是等待IO的请求超过这个数值，那么会停止接受外部消息，应该是一种自我保护机制
queued.max.requests

# socket的发送缓冲区
socket.send.buffer.bytes

# socket的接受缓冲区
socket.receive.buffer.bytes

# socket请求的最大字节数值，防止serverOOM，message.max.bytes必然要小于socket.request.max.bytes，会被topic创建时的指定参数覆盖
socket.request.max.bytes

# topic的分区是以一堆segment文件存储的，这个控制每个segment的大小，会被topic创建时的指定参数覆盖
log.segment.bytes

# 在日志segment没有达到log.segment.bytes设置的大小，但超过设定时间，也会强制新建一个segment。会被 topic创建时的指定参数覆盖
log.roll.hours

# 日志清理策略选择有：delete和compact主要针对过期数据的处理，或是日志文件达到限制的额度，会被 topic创建时的指定参数覆盖
log.cleanup.policy

# 数据存储的最大时间超过这个时间会根据log.cleanup.policy设置的策略处理数据，也就是消费端能够多久去消费数据，log.retention.bytes和log.retention.minutes任意一个达到要求，都会执行删除，会被topic创建时的指定参数覆盖
log.retention.minutes

# topic每个分区的最大文件大小，一个topic的大小限制 =分区数*log.retention.bytes。-1没有大小限log.retention.bytes和log.retention.minutes任意一个达到要求，都会执行删除，会被topic创建时的指定参数覆盖
log.retention.bytes

# 文件大小检查的周期时间，是否执行 log.cleanup.policy中设置的策略
log.retention.check.interval.ms

# 是否开启日志压缩
log.cleaner.enable

# 日志压缩运行的线程数
log.cleaner.threads

# 日志压缩时每秒处理的最大大小
log.cleaner.io.max.bytes.per.second

# 日志压缩去重时候的缓存空间，在空间允许的情况下，越大越好
log.cleaner.dedupe.buffer.size

# 日志清理时候用到的IO块大小一般不需要修改
log.cleaner.io.buffer.size

# 日志清理中hash表的扩大因子一般不需要修改
log.cleaner.io.buffer.load.factor

# 检查是否触发日志清理的间隔
log.cleaner.backoff.ms

# 日志清理的频率控制，越大意味着更高效的清理，同时会存在一些空间上的浪费，会被topic创建时的指定参数覆盖
log.cleaner.min.cleanable.ratio

# 对于压缩的日志保留的最长时间，也是客户端消费消息的最长时间，与log.retention.minutes的区别在于一个控制未压缩数据，一个控制压缩后的数据。会被topic创建时的指定参数覆盖
log.cleaner.delete.retention.ms

# 对于segment日志的索引文件大小限制，会被topic创建时的指定参数覆盖
log.index.size.max.bytes

# 当执行一个fetch操作后，需要一定的空间来扫描最近的offset大小，设置越大，代表扫描速度越快，但是也更好内存，一般情况下不需要搭理这个参数
log.index.interval.bytes

# log文件”sync”到磁盘之前累积的消息条数,因为磁盘IO操作是一个慢操作,但又是一个”数据可靠性"的必要手段,所以此参数的设置,需要在"数据可靠性"与"性能"之间做必要的权衡.如果此值过大,将会导致每次"fsync"的时间较长(IO阻塞),如果此值过小,将会导致"fsync"的次数较多,这也意味着整体的client请求有一定的延迟.物理server故障,将会导致没有fsync的消息丢失.
log.flush.interval.messages

# 检查是否需要固化到硬盘的时间间隔
log.flush.scheduler.interval.ms

# 仅仅通过interval来控制消息的磁盘写入时机,是不足的.此参数用于控制"fsync"的时间间隔,如果消息量始终没有达到阀值,但是离上一次磁盘同步的时间间隔达到阀值,也将触发.
log.flush.interval.ms

# 文件在索引中清除后保留的时间一般不需要去修改
log.delete.delay.ms

# 控制上次固化硬盘的时间点，以便于数据恢复一般不需要去修改
log.flush.offset.checkpoint.interval.ms 

# 消息时间戳类型，CreateTime 和 LogAppendTime ；前者表示producer创建这条消息的时间；后者表示broker接收到这条消息的时间(严格来说，是leader broker将这条消息写入到log的时间
log.message.timestamp.type	

# 是否允许自动创建topic，若是false，就需要通过命令创建topic
auto.create.topics.enable 

# 每个topic的分区个数，会被topic创建时的指定参数覆盖
num.partitions

# 用于配置offset记录的topic的partition的副本个数
offsets.topic.replication.factor=3

# 事务主题的复制因子
transaction.state.log.replication.factor=3

# 覆盖事务主题的min.insync.replicas配置
transaction.state.log.min.isr=3

# 新建Topic时默认的分区数
default.replication.factor=3

```

