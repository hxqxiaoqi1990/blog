---
title: RocketMQ分布式部署
tags: [RocketMQ]
categories: 大数据
date: 2020-01-21 16:00:00
---

# 介绍

RocketMQ的多Master多Slave的模式在Linux服务器部署案例进行详细的说明。

# 部署步骤

**主机环境：**

- Nameserver1 + Master1 + Master2-Slave：192.168.40.100
- Nameserver2 + Master2 + Master1-Slave：192.168.40.101

**系统环境：**

- 64位JDK 1.8+;
- Maven 3.6.x;
- git；

**操作步骤：**

1. 全部安装jdk
2. 全部安装maven
3. 全部安装RocketMQ，修改配置
4. 安装web管理服务

## jdk安装

[**jdk安装跳转**](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/)

## maven安装

[**maven安装跳转**](https://hxqxiaoqi.gitee.io/2019/09/01/maven-3.6.1安装脚本/)

## RocketMQ分布式安装

**下载**

[RocketMQ安装参考](https://hxqxiaoqi.gitee.io/2020/01/21/RocketMQ安装/) 两台主机按文档安装

**创建数据目录**

在192.168.40.100操作

```bash
# Master1数据目录
mkdir -p /data/rocketmq/data/{commitlog,consumequeue,index}
# Master2-Slave数据目录
mkdir -p /data/rocketmq/datas/{commitlog,consumequeue,index}
```

在192.168.40.101操作

```bash
# Master2数据目录
mkdir -p /data/rocketmq/data/{commitlog,consumequeue,index}
# Master1-Slave数据目录
mkdir -p /data/rocketmq/datas/{commitlog,consumequeue,index}
```

**修改配置文件**

在192.168.40.100操作

```bash
# 修改Master1配置
vim /data/rocketmq/conf/2m-2s-async/broker-a.properties
```

```yaml
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=192.168.40.100:9876;192.168.40.101:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
haListenPort=10912
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=18
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/mnt/rocketmq-all-4.6.0-bin-release/data
#commitLog 存储路径
storePathCommitLog=/mnt/rocketmq-all-4.6.0-bin-release/data/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/mnt/rocketmq-all-4.6.0-bin-release/data/consumequeue
#消息索引存储路径
storePathIndex=/mnt/rocketmq-all-4.6.0-bin-release/data/index
#checkpoint 文件存储路径
storeCheckpoint=/mnt/rocketmq-all-4.6.0-bin-release/data/checkpoint
#abort 文件存储路径
abortFile=/mnt/rocketmq-all-4.6.0-bin-release/data/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
#强制指定本机IP，需要根据每台机器进行修改。官方介绍可为空，系统默认自动识别，但多网卡时IP地址可能读取错误
brokerIP1=192.168.40.100
```

```bash
# 修改Master2-Slave配置
vim /data/rocketmq/conf/2m-2s-async/broker-b-s.properties
```

```yaml
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
brokerId=1
#nameServer地址，分号分割
namesrvAddr=192.168.40.100:9876;192.168.40.101:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10923
haListenPort=10924
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=18
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/mnt/rocketmq-all-4.6.0-bin-release/datas
#commitLog 存储路径
storePathCommitLog=/mnt/rocketmq-all-4.6.0-bin-release/datas/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/mnt/rocketmq-all-4.6.0-bin-release/datas/consumequeue
#消息索引存储路径
storePathIndex=/mnt/rocketmq-all-4.6.0-bin-release/datas/index
#checkpoint 文件存储路径
storeCheckpoint=/mnt/rocketmq-all-4.6.0-bin-release/datas/checkpoint
#abort 文件存储路径
abortFile=/mnt/rocketmq-all-4.6.0-bin-release/datas/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
#强制指定本机IP，需要根据每台机器进行修改。官方介绍可为空，系统默认自动识别，但多网卡时IP地址可能读取错误
brokerIP1=192.168.40.100
```

在192.168.40.101操作

```bash
# 修改Master2配置
vim /data/rocketmq/conf/2m-2s-async/broker-b.properties
```

```yaml
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-b
#0 表示 Master，>0 表示 Slave
brokerId=0
#nameServer地址，分号分割
namesrvAddr=192.168.40.100:9876;192.168.40.101:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10911
haListenPort=10912
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=18
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/mnt/rocketmq-all-4.6.0-bin-release/data
#commitLog 存储路径
storePathCommitLog=/mnt/rocketmq-all-4.6.0-bin-release/data/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/mnt/rocketmq-all-4.6.0-bin-release/data/consumequeue
#消息索引存储路径
storePathIndex=/mnt/rocketmq-all-4.6.0-bin-release/data/index
#checkpoint 文件存储路径
storeCheckpoint=/mnt/rocketmq-all-4.6.0-bin-release/data/checkpoint
#abort 文件存储路径
abortFile=/mnt/rocketmq-all-4.6.0-bin-release/data/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=ASYNC_MASTER
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
#强制指定本机IP，需要根据每台机器进行修改。官方介绍可为空，系统默认自动识别，但多网卡时IP地址可能读取错误
brokerIP1=192.168.40.101
```

```bash
# 修改Master1-Slave配置
vim /data/rocketmq/conf/2m-2s-async/broker-b.properties
```

```yaml
#所属集群名字
brokerClusterName=rocketmq-cluster
#broker名字，注意此处不同的配置文件填写的不一样
brokerName=broker-a
#0 表示 Master，>0 表示 Slave
brokerId=1
#nameServer地址，分号分割
namesrvAddr=192.168.40.100:9876;192.168.40.101:9876
#在发送消息时，自动创建服务器不存在的topic，默认创建的队列数
defaultTopicQueueNums=4
#是否允许 Broker 自动创建Topic，建议线下开启，线上关闭
autoCreateTopicEnable=true
#是否允许 Broker 自动创建订阅组，建议线下开启，线上关闭
autoCreateSubscriptionGroup=true
#Broker 对外服务的监听端口
listenPort=10923
haListenPort=10924
#删除文件时间点，默认凌晨 4点
deleteWhen=04
#文件保留时间，默认 48 小时
fileReservedTime=18
#commitLog每个文件的大小默认1G
mapedFileSizeCommitLog=1073741824
#ConsumeQueue每个文件默认存30W条，根据业务情况调整
mapedFileSizeConsumeQueue=300000
#destroyMapedFileIntervalForcibly=120000
#redeleteHangedFileInterval=120000
#检测物理文件磁盘空间
diskMaxUsedSpaceRatio=88
#存储路径
storePathRootDir=/mnt/rocketmq-all-4.6.0-bin-release/datas
#commitLog 存储路径
storePathCommitLog=/mnt/rocketmq-all-4.6.0-bin-release/datas/commitlog
#消费队列存储路径存储路径
storePathConsumeQueue=/mnt/rocketmq-all-4.6.0-bin-release/datas/consumequeue
#消息索引存储路径
storePathIndex=/mnt/rocketmq-all-4.6.0-bin-release/datas/index
#checkpoint 文件存储路径
storeCheckpoint=/mnt/rocketmq-all-4.6.0-bin-release/datas/checkpoint
#abort 文件存储路径
abortFile=/mnt/rocketmq-all-4.6.0-bin-release/datas/abort
#限制的消息大小
maxMessageSize=65536
#flushCommitLogLeastPages=4
#flushConsumeQueueLeastPages=2
#flushCommitLogThoroughInterval=10000
#flushConsumeQueueThoroughInterval=60000
#Broker 的角色
#- ASYNC_MASTER 异步复制Master
#- SYNC_MASTER 同步双写Master
#- SLAVE
brokerRole=SLAVE
#刷盘方式
#- ASYNC_FLUSH 异步刷盘
#- SYNC_FLUSH 同步刷盘
flushDiskType=ASYNC_FLUSH
#checkTransactionMessageEnable=false
#发消息线程池数量
#sendMessageThreadPoolNums=128
#拉消息线程池数量
#pullMessageThreadPoolNums=128
#强制指定本机IP，需要根据每台机器进行修改。官方介绍可为空，系统默认自动识别，但多网卡时IP地址可能读取错误
brokerIP1=192.168.40.101
```

**启动Nameserver**

在192.168.40.100操作

```bash
cd /data/rocketmq/
nohup sh bin/mqnamesrv &
```

在192.168.40.101操作

```bash
cd /data/rocketmq/
nohup sh bin/mqnamesrv &
```

**启动Broker**

在192.168.40.100操作

```bash
cd /data/rocketmq/bin
sh mqbroker -c /data/rocketmq/conf/2m-2s-async/broker-a.properties
sh mqbroker -c /data/rocketmq/conf/2m-2s-async/broker-b-s.properties
```

在192.168.40.101操作

```bash
cd /data/rocketmq/bin
sh mqbroker -c /data/rocketmq/conf/2m-2s-async/broker-a-s.properties
sh mqbroker -c /data/rocketmq/conf/2m-2s-async/broker-b.properties
```

# RocketMQ监控平台部署

**下载**

在192.168.40.100操作

```bash
yum -y install git
cd /opt
git clone https://github.com/apache/rocketmq-externals.git
```

**配置**

```bash
cd rocketmq-externals/rocketmq-console/src/main/resources/
vim application.properties
# 修改以下内容
```

```yaml
rocketmq.config.namesrvAddr=192.168.40.100:9876;192.168.40.101:9876
```

**编译**

```bash
mvn clean package -Dmaven.test.skip=true
```

**运行**

```bash
cd /opt/rocketmq-externals-master/rocketmq-console/target
java -jar target/rocketmq-console-ng-1.0.1.jar
```

**访问**

```bash
curl http://192.168.40.100:8080/
```

在监控平台查看：集群选项，即可看到集群信息。

