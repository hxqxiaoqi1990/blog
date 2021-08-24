---
title: redis主从与哨兵模式搭建
tags: [redis]
categories: 数据库
date: 2019-10-18 15:47:00
---

# 主从介绍
主从：主节点负责写数据，从节点负责读数据，主节点定期把数据同步到从节点保证数据的一致性
# 哨兵机制介绍
**主要功能如下**
 1. 集群监控：负责监控redis master和slave进程是否正常工作 
 2. 消息通知：如果某个redis实例有故障，那么哨兵负责发送消息作为报警通知给管理员 
 3. 故障转移：如果master node挂掉了，会自动转移到slave node上 
 4. 配置中心：如果故障转移发生了，通知client客户端新的master地址

**哨兵的核心知识**

 1. 故障转移时，判断一个master node是宕机了，需要大部分的哨兵都同意才行，涉及到了分布式选举的问题
 2. 哨兵至少需要3个实例，来保证自己的健壮性
 3. 哨兵 + redis主从的部署架构，是不会保证数据零丢失的，只能保证redis集群的高可用性

# 部署主从
**环境说明：**

 1. master安装：redis主   192.168.40.100:6739
 2. node1安装：redis从    192.168.40.101:6739
 2. node2安装：哨兵1和哨兵2    192.168.40.102:16739与192.168.40.101:26739
 3. 安装目录均在 /data 下

**安装redis主：**

``` bash
wget http://download.redis.io/redis-stable.tar.gz

tar xf redis-stable.tar.gz -C /data

cd /data/redis-stable/

make && make install
```
**配置文件**

[redis配置文件详解](https://hxqxiaoqi.gitee.io/2019/10/19/redis%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E8%AF%A6%E8%A7%A3/)

``` bash
daemonize yes
protected-mode no
port 6379
pidfile "/var/run/redis.pid"
logfile "/var/log/redis.log"

dbfilename "dump.rdb"
dir "./data"
save 900 1
save 300 10
save 60 10000
```
**启动**

``` bash
cd /data/redis-stable/
./src/redis-server ./redis.conf
```
**安装redis从**
全部与安装redis主一致，只是配置文件中多一条配置，之后启动从，日志中可以看出已经连接master，并同步数据

``` bash
# 从服务指定主服务的IP和port
slaveof 192.168.40.100 6379
```

# 部署哨兵

**安装哨兵1**
安装redis与上面步骤一致

``` bash
cd /data/redis-stable/
vim sentinel.conf
```

**sentinel.conf配置文件**
[redis配置文件详解](https://hxqxiaoqi.gitee.io/2019/10/19/redis%E9%85%8D%E7%BD%AE%E6%96%87%E4%BB%B6%E8%AF%A6%E8%A7%A3/)

``` bash
# 设置成：protected-mode no；保护模式关闭，如果你不关闭保护模式，启动哨兵的时候，无法正常运行。还有个解决办法就是你设置密码，但是一般都不设置redis的密码。麻烦，我每次连接还得输入密码。
protected-mode no
daemonize yes
port 16379
dir "/tmp"
logfile "/var/log/redis_sen1.log"
sentinel monitor redis01 192.168.40.100 6379 2
sentinel down-after-milliseconds redis01 10000
sentinel failover-timeout redis01 100000
sentinel parallel-syncs redis01 1
sentinel config-epoch redis01 5
```
**启动**

``` bash
cd /data/redis-stable/
./src/redis-sentinel ./sentinel.conf
```

**安装哨兵2**
安装步骤与哨兵1一致，注意区分端口和日志文件位置
查看日志，如果有主从服务的相关信息，则哨兵模式部署完成

# 测试主从与哨兵

## 验证主从

登录主服务器，主从模式，只有主可写入数据，从不可写
``` bash
# 登录主
cd /data/redis-stable/
./src/redis-cli

# 查看主从状态
127.0.0.1:6379> info Replication

# 写入数据
127.0.0.1:6379> set test 12312312

# 获取数据
127.0.0.1:6379> get test
```
登录从服务器

``` bash
# 查看主从状态
127.0.0.1:6379> info Replication

# 查看主写入的数据是否同步到从
127.0.0.1:6379> get test

# 写入数据，会提示无法写入，即是从
127.0.0.1:6379> set test1 343434
```

## 测试哨兵模式

 1.实时查看哨兵1和哨兵2日志
 2.关闭redis主服务，等待10s，哨兵日志提示，主服务下线，从服务切换为主
 3.登录原从服务器，查看身份状态，身份改为主
 4.测试是否可以写入

# 数据备份
**备份**
``` bash
# 该命令将在 redis 安装目录中创建dump.rdb文件
redis 127.0.0.1:6379> SAVE 

# 创建 redis 备份文件也可以使用命令 BGSAVE，该命令在后台执行。（推荐）
redis 127.0.0.1:6379> BGSAVE 

```
**恢复**

如果需要恢复数据，只需将备份文件 (dump.rdb) 移动到 redis 安装目录并启动服务即可。获取 redis 目录可以使用 CONFIG 命令，如下所示：

``` bash
redis 127.0.0.1:6379> CONFIG GET dir
```

# 启动警告

警告一：

```bash
WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
```

解决方法：

```bash
echo "net.core.somaxconn = 1024" >> /etc/sysctl.conf
sysctl -p
```

警告二：

```bash
WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect
```

解决方法：

```bash
echo "vm.overcommit_memory = 1" >> /etc/sysctl.conf
sysctl -p
```

警告三：

```bash
WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
```

解决方法：

```bash
# 临时
echo never > /sys/kernel/mm/transparent_hugepage/enabled
# 永久
echo "echo never > /sys/kernel/mm/transparent_hugepage/enabled" >> /etc/rc.local
```

