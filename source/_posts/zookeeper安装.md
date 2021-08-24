---
title: zookeeper安装
tags: [zookeeper]
categories: 中间件
date: 2019-09-03 14:00:00
---


# 介绍
官方文档上这么解释zookeeper，它是一个分布式服务框架，是Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。
# 安装步骤

 1. 下载，解压
 2. 配置文件
 3. 配置环境变量
 4. 启动

## 下载，解压

``` bash
# 下载
wget http://mirror.bit.edu.cn/apache/zookeeper/stable/apache-zookeeper-3.5.5-bin.tar.gz

tar xf apache-zookeeper-3.5.5-bin.tar.gz -C /usr/local/
```

## 配置文件

``` bash
cd /usr/local/apache-zookeeper-3.5.5-bin/conf/

# 如果只是运行，配置文件默认即可
mv zoo_sample.cfg zoo.cfg
```

## 配置环境变量

``` bash
# 配置环境变量，直接使用启动命令
echo '
# zookeeper
export ZK_HOME=/usr/local/apache-zookeeper-3.5.5-bin
export PATH=$ZK_HOME/bin:$PATH
' >> /etc/profile

source /etc/profile
```

## 启动

``` bash
zkServer.sh start
zkServer.sh stop
zkServer.sh status
```
# 8080端口占用
zookeeper最近的版本中有个内嵌的管理控制台是通过jetty启动，也会占用8080 端口。
``` bash
vim /data/zookeeper-3.5.5-bin/conf/zoo.cfg
	# 添加
	admin.serverPort=8888
```