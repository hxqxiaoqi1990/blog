---
title: canal高可用安装
tags: [canal]
categories: 中间件
date: 2020-01-20 16:00:00
---

# 介绍

canal高可用模式是通过zookeeper注册canal当前服务状态来实现的。

搭建参考地址：https://github.com/alibaba/canal/wiki/AdminGuide

# 搭建步骤

**主机环境：**

- canal+admin+zookeeper+jdk：192.168.40.100
- mysql+jdk+canal：192.168.40.101

**搭建步骤：**

1. 安装mysql
2. 安装jdk
3. 安装zookeeper
4. 安装2个canal
5. 安装admin管理界面

## mysql安装

**[mysql安装]([https://hxqxiaoqi.gitee.io/2019/10/23/centos7%E5%AE%89%E8%A3%85mariadb/](https://hxqxiaoqi.gitee.io/2019/10/23/centos7安装mariadb/))** 在192.168.40.101安装

## jdk安装

**[参照canal安装](https://hxqxiaoqi.gitee.io/2020/01/19/canal安装/)** 在192.168.40.100和192.168.40.101安装

## zookeeper安装

**[参照zookeeper安装](https://hxqxiaoqi.gitee.io/2019/09/03/zookeeper安装/)** 在192.168.40.100安装

## canal.admin安装

**在192.168.40.100安装**

```bash
# 下载地址
wget https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.admin-1.1.4.tar.gz

tar xf canal.admin-1.1.4.tar.gz -C /data/canal.admin
vim /data/canal.admin/conf/application.yml
```

```yaml
server:
  port: 8089
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8

spring.datasource:
  address: 127.0.0.1:3306	#数据地址
  database: canal_manager	#数据库名
  username: canal		#数据库账号，数据库授权需要写权限
  password: canal		#数据库密码
  driver-class-name: com.mysql.jdbc.Driver
  url: jdbc:mysql://${spring.datasource.address}/${spring.datasource.database}?useUnicode=true&characterEncoding=UTF-8&useSSL=false
  hikari:
    maximum-pool-size: 30
    minimum-idle: 1

# canal-server需要凭此账号密码连接到admin上，不是登录密码，是服务间的连接校验
canal:
  adminUser: admin	
  adminPasswd: admin
```

**登录数据库，加载初始化配置**

```bash
mysql -uroot -p123123
```

```sql
mysql> source /data/canal.admin/conf/canal_manager.sql;
```

**启动**

```bash
# 启动
/data/admin/bin/startup.sh
# 关闭
/data/admin/bin/stop.sh
```

**访问web**

```bash
curl http://192.168.40.100:8089
```

**web界面配置**

1. 访问 http://192.168.40.100:8089
2. 登录：账号：admin，密码：123456
3. 点击集群管理 --> 新建集群 --> 填写zookeeper信息
4. 在新建的集群右侧点击操作 --> 主配置 -->  载入模板 -->  把`canal.user`与`canal.passwd`注释 --> 修改`canal.admin.manager`为IP地址，注释`canal.instance.global.spring.xml = classpath:spring/file-instance.xml` 开启`canal.instance.global.spring.xml = classpath:spring/default-instance.xml` --> 添加`canal.zkServers`地址 -->保存
5. server管理：为canal.deployer服务配置
6. instance管理：为example数据源配置

## canal 安装

[canal安装教程链接](https://hxqxiaoqi.gitee.io/2020/01/20/canal高可用安装/)

canal.deployer配置中声明admin配置，才可以注册到admin服务上

修改canal_local.properties配置，canal_local.properties为canal最简化的配置，高可用模式使用该配置，单机模式可以使用默认文件canal.properties

```bash
vim /data/canal/conf/canal_local.properties
```

```yaml
# 本机IP
canal.register.ip =

# canal admin config
canal.admin.manager = 192.168.40.100:8089
canal.admin.port = 11110
# 该账号密码对应admin配置，密码由mysql生成：select password('admin')，去掉*号
canal.admin.user = admin
canal.admin.passwd = 4ACFE3202A5FF5CF467898FC58AAB1D615029441
# 自动注册到admin服务
canal.admin.register.auto = true
# 如果是高可用，填写zookeeper地址，如果是单机，可不填写
canal.admin.register.cluster = zk
```

**启动**

```bash
# 必须先启动admin服务，也可直接使用canal_local.properties重命名覆盖canal.properties文件，即可正常启动
cd /data/canal.admin/
sh bin/startup.sh local

# 查看日志
tailf logs/admin.log
```

<font color='32CD32'>注：</font> 

- 启动canal后，在admin界面即可查看到注册信息
- 多台注册，配置与上面一样
- 如果注册到同一个集群，配置文件为统一配置
- 可在admin界面管理instance，直接创建，不需要在服务器上手动创建
- zookeeper上也可查看到集群信息



