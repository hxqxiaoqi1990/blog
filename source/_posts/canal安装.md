---
title: canal安装
tags: [canal]
categories: 中间件
date: 2020-01-19 10:00:00
---

# 介绍

**canal [kə'næl]**，主要用途是基于 MySQL 数据库增量日志解析，提供增量数据订阅和消费。支持数据同步到elasticsearch、redis、kafka等服务。

当前的 canal 支持源端 MySQL 版本包括 5.1.x , 5.5.x , 5.6.x , 5.7.x , 8.0.x

搭建参考地址：https://github.com/alibaba/canal/wiki/AdminGuide

<font color='32CD32'>canal 工作原理</font>

- canal 模拟 MySQL slave 的交互协议，伪装自己为 MySQL slave ，向 MySQL master 发送dump 协议
- MySQL master 收到 dump 请求，开始推送 binary log 给 slave (即 canal )
- canal 解析 binary log 对象(原始为 byte 流)

# canal安装

**主机环境：**

- canal+mysql：192.168.40.100

**系统环境：**

- jdk1.8.0_221
- MySQL5.5.64

[**canal下载地址**](https://github.com/alibaba/canal/releases)

- canal 服务端：canal.deployer-1.1.4.tar.gz	
- canal 客户端测试：canal.example-1.1.4.tar.gz	
- canal web管理界面：canal.admin-1.1.4.tar.gz		
- canal 适配器：canal.adapter-1.1.4.tar.gz	 

**安装步骤：**

1. 安装mysql，并开启bin-log
2. 安装jdk
3. 安装canal.deployer，修改配置
4. 安装canal.example测试服务是否成功

## mysql安装

[mysql安装跳转](https://hxqxiaoqi.gitee.io/2019/10/23/centos7安装mariadb/)

canal的原理是基于mysql binlog技术，所以这里一定需要开启mysql的binlog写入功能，并且配置binlog模式为row，重启mysql。

```bash
vim /etc/my.cnf
```

```bash
[mysqld]  
log-bin=mysql-bin #添加这一行就ok  
binlog-format=ROW #选择row模式  
server_id=1 #配置mysql replaction需要定义，不能和canal的slaveId重复  
```

登录mysql查看bin-log是否开启

```sql
mysql> show variables like 'binlog_format';
mysql> show variables like 'log_bin';
```

配置canal服务使用的账号权限

```sql
# 创建用户授权
mysql> CREATE USER canal IDENTIFIED BY 'canal';    
mysql> GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';  
mysql> FLUSH PRIVILEGES; 
# 查看授权
mysql> show grants for 'canal';
```

## jdk安装

[**jdk安装跳转**](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/)

## canal.deployer安装

```bash
# 下载地址
wget https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.deployer-1.1.4.tar.gz
mkdir -p /data/{canal.deployer,canal.admin,canal.example}
tar xf canal.deployer-1.1.4.tar.gz -C /data/canal.deployer
```

**配置文件说明：**

- conf/canal.properties：canal全局配置文件
- conf/example/instance.properties：一个数据源配置文件
- logs/canal/canal.log：canal.deployer服务日志
- logs/example/example.log：数据源日志

**修改配置**

```bash
vim conf/example/instance.properties
```

```bash
# 数据库连接地址
canal.instance.master.address=127.0.0.1:3306

# 数据库账号
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal

# mysql 数据解析关注的表，Perl正则表达式.多个正则之间以逗号(,)分隔，转义符需要双斜杠(\\)
# 常见例子：
# 1.  所有表：.*   or  .*\\..*
# 2.  canal schema下所有表： canal\\..*
# 3.  canal下的以canal打头的表：canal\\.canal.*
# 4.  canal schema下的一张表：canal\\.test1
# 5.  多个规则组合使用：canal\\..*,mysql.test1,mysql.test2 (逗号分隔)
# table regex：白名单，指定收集的库或表
canal.instance.filter.regex=.*\\..*
# table black regex：黑名单
canal.instance.filter.black.regex=
```

<font color='32CD32'>注：</font> 

1. 一个example目录为一个数据源，如果需要配置多个数据源，可以复制example为其它名字，再修改
2. example目录为canal默认数据源，不能改名或删除，否正会报错
3. canal每5秒，自动加载conf下的数据源目录，配置数据源不需要重启服务

```bash
# 启动服务端口为11111，11112
./bin/startup.sh
./bin/stop.sh 
```

## canal.example安装

```bash
# 下载地址
wget https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.example-1.1.4.tar.gz
tar xf canal.example-1.1.4.tar.gz -C /data/canal.example
cd /data/canal.example
./bin/startup.sh

# 监控example日志
tailf logs/example/entry.log
```

<font color='32CD32'>注：</font> canal.example服务只能收集example数据源操作数据，如果是新增的数据源，需要java编写客户端获取数据。

**测试服务**

1. 监控example日志
2. 操作mysql增加、删除、更新
3. example日志显示相应的bin-log日志
4. 至此，最简单canal服务搭建成功

