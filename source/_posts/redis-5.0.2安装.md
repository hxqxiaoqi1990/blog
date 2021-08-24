---
title: redis安装 
tags: [redis]
categories: 数据库
date: 2019-09-04 10:00:00
---


# 介绍
REmote DIctionary Server(Redis) 是一个由Salvatore Sanfilippo写的key-value存储系统。

Redis是一个开源的使用ANSI C语言编写、遵守BSD协议、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。

它通常被称为数据结构服务器，因为值（value）可以是 字符串(String), 哈希(Hash), 列表(list), 集合(sets) 和 有序集合(sorted sets)等类型。
# 安装步骤

 1. 安装环境，下载，解压，编译
 2. 修改配置
 3. 启动，登录

## 安装
以下为一键安装脚本
``` bash
#!/bin/bash
# redis自动安装脚本

redis_dir=/usr/local
redis_version=redis-stable

# 下载
wget http://download.redis.io/$redis_version.tar.gz
yum -y install gcc

tar xf $redis_version.tar.gz -C $redis_dir
cd $redis_dir/$redis_version

# 编译
make && make install

mkdir $redis_dir/$redis_version/{data,logs}

# 导入配置
cat > redis.conf <<EOF
daemonize yes
protected-mode no
port 6379
pidfile "$redis_dir/$redis_version/logs/redis.pid"
logfile "$redis_dir/$redis_version/logs/redis.log"
dbfilename "dump.rdb"
dir "./data"
save 900 1
save 300 10
save 60 10000
EOF

# 启动
$redis_dir/$redis_version/src/redis-server $redis_dir/$redis_version/redis.conf
```

## 启动，登录

``` bash
cd /usr/local/redis-stable

# 启动
./src/redis-server ./redis.conf

# 关闭
./src/redis-cli -p 6379 shutdown

# 登录
./src/redis-cli -h 127.0.0.1 -p 6379
```

# 自启动脚本
将以下文件保存为redis.service，并放在：/usr/lib/systemd/system/
每次更改文件必须重启：systemctl daemon-reload

``` bash
[Unit]
Description=Redis
After=network.target

[Service]
Restart=always
RestartSec=1
ExecStart=/usr/local/redis-stable/src/redis-server /usr/local/redis-stable/redis.conf  --daemonize no
ExecStop=/usr/local/redis-stable/src/redis-cli -h 127.0.0.1 -p 6379 shutdown

[Install]
WantedBy=multi-user.target
```
[Unit] 表示这是基础信息
Description 是描述
After 是在那个服务后面启动，一般是网络服务启动后启动
[Service] 表示这里是服务信息
Restart=always 定义何种情况 Systemd 会自动重启当前服务，可能的值包括always（总是重启）、on-success、on-failure、on-abnormal、on-abort、on-watchdog
RestartSec=1 定义 Systemd 停止当前服务之前等待的秒数
ExecStart 是启动服务的命令
ExecStop 是停止服务的指令
[Install] 表示这是是安装相关信息
WantedBy 是以哪种方式启动：multi-user.target表明当系统以多用户方式（默认的运行级别）启动时，这个服务需要被自动运行。

# redis-dump导入导出

```bash
#配置yum仓库
yum install centos-release-scl-rh -y

#安装其他工具，不安装后面可能会报错
yum install rh-ruby23*  -y

scl  enable  rh-ruby23 bash
#查看版本
ruby -v

gem install redis-dump -V
```



```bash
# 导出数据
redis-dump -u  192.168.0.32 -d 2 > test.json

# 导入
< test.json redis-load -u 192.168.0.31

# 修改导出的json文件，可以指定导入哪个库
sed -i 's/"db":2/"db":0/g' test.json
```



# 报错

``` bash
# 报错
WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.

# 解决
net.core.somaxconn= 1024  #sysctl.conf
vm.overcommit_memory=1	  #sysctl.conf
```