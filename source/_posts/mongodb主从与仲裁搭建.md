---
title: mongodb主从与仲裁搭建
tags: [mongodb]
categories: 数据库
date: 2019-10-28 18:58:00
---


# 介绍
中文翻译叫做副本集。其实简单来说就是集群当中包含了多份数据，保证主节点挂掉了，备节点能继续提供数据服务，提供的前提就是数据需要和主节点一致。


默认设置下，主节点提供所有增删查改服务，备节点不提供任何服务。但是可以通过设置使备节点提供查询服务，这样就可以减少主节点的压力，当客户端进行数据查询时，请求自动转到备节点上。

仲裁节点是一种特殊的节点，它本身并不存储数据，主要的作用是决定哪一个备节点在主节点挂掉之后提升为主节点，所以客户端不需要连接此节点。这里虽然只有一个备节点，但是仍然需要一个仲裁节点来提升备节点级别。

# 部署
**部署环境：**
以下每台安装mongodb
 1. 192.168.40.100 master
 2. 192.168.40.101 slave
 3. 192.168.40.102 arbiter

安装mongodb请看该文档：[mongodb安装](https://hxqxiaoqi.gitee.io/2019/10/28/mongodb%E6%90%AD%E5%BB%BA/)
注：配置文件需要更改为以下配置启动。

**配置文件**

注：master，slave，arbiter配置都是一致的，配置中replSet=RS1为集群名称，之后根据该名称设置集群

``` bash
bind_ip=0.0.0.0
port=27017
# 持久化存储引擎
storageEngine=wiredTiger

# 内部缓存的大小
wiredTigerCacheSizeGB=2

# 是设置从内存同步到硬盘的时间间隔，默认为60秒，可以设置的少一些
syncdelay=30

# 设定压缩策略 snappy 是一种压缩速度非常快的压缩策略
wiredTigerCollectionBlockCompressor=snappy

# 集群名称
replSet=RS1

# 文件位置
dbpath=/data/mongodb-linux-x86_64-3.6.14/data
logpath=/data/mongodb-linux-x86_64-3.6.14/logs/mongodb.log

# mongodb操作日志文件的最大大小。单位为Mb，默认为硬盘剩余空间的5%
oplogSize=6144

# 追加方式写入日志
logappend=true

# 后台运行
fork=true

# 启用日志文件，默认启用
journal=true

# 为每一个数据库按照数据库名建立文件夹存放
directoryperdb=true
```
**启动**
全部启动
``` bash
./bin/monood -f mongodb.conf 
./bin/mongod -f mongodb.conf 
./bin/mongod -f mongodb.conf 
```

**配置主，备，仲裁节点**

可以通过客户端连接mongodb，也可以直接在三个节点中选择一个连接mongodb。

``` bash
# 登录
./bin/mongo

# 切换库
>use admin;

# 配置仲裁，根据实际IP地址配置
>cfg={ _id:"testrs", members:[ {_id:0,host:'192.168.40.100:27017',priority:2}, {_id:1,host:'192.168.40.101:27017',priority:1}, {_id:2,host:'192.168.40.102:27017',arbiterOnly:true}] };

# 查看状态：PRIMARY为主，SECONDARY为从，ARBITER为仲裁
rs.status()
```

cfg是可以任意的名字，当然最好不要是mongodb的关键字，conf，config都可以。最外层的_id表示replica set的名字，members里包含的是所有节点的地址以及优先级。优先级最高的即成为主节点，即这里的10.10.148.130:27017。特别注意的是，对于仲裁节点，需要有个特别的配置——arbiterOnly:true。这个千万不能少了，不然主备模式就不能生效。

# 实验

 1. 杀死主（db.shutdownServer();）
 2. 查看仲裁日志
 3. 查看从状态

# 添加节点

``` bash
# 添加节点
> rs.add("192.168.40.101:27017");

# 添加仲裁节点
> rs.addArb("192.168.40.102:27017");

# 移除节点
> rs.remove("192.168.40.101:27020");

```