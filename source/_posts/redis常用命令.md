---
title: redis常用命令
tags: [redis]
categories: 数据库
date: 2019-10-19 15:29:00
---

# 介绍
方便自己查询使用

# 常用命令
## 登录查询命令

``` bash
# 登录redis
./redis-cli -h 主机IP -a 密码 -p 端口 [可执行命令]

# 查看redis信息
info

# 查看集群身份
info Replication

# 清除当前数据库的所有keys
flush db   

# 清除所有数据库的所有keys
flush all    

 # 查看所有keys
keys *    

# 查看前缀为"prefix_"的所有keys
keys prefix_*     

# 确认一个key是否存在
exists key      

# 设置key和value
set key value   

# 获取key的value
get key        

 # 删除一个key
del key        

 # 返回值的类型
type key       

# 返回满足给定pattern的所有key
keys pattern    

# 随机返回key空间的一个
random key      

 # 重命名key
key rename oldname newname   

 # 返回当前数据库中key的数目
db size        

# 选择第0~15中的库
select index   

# 移动当前数据库中的key到dbindex数据库
move key dbindex      

# 设置key过期时间为120秒
set key value ex 120

# 批量删除键
redis-cli -h 192.168.10.71 -p 6379  keys "Mpos:base:user:info:key:*" | xargs redis-cli -h 192.168.10.71 -p 6379 del
```

## 持久化存储命令

``` bash
# SAVE命令会占用当前的redis服务进程进行重写
127.0.0.1:6379> SAVE

# BGSAVE命令会以redis服务进程启动一个名为redis-rdb-bgsave的子进程,因此temp-PID.rdb中的PID即是该子进程redis-rdb-bgsave的PID,因此不会阻塞redis服务进程，redis正常对外服务
127.0.0.1:6379> BGSAVE

# 该命令会以redis服务进程打开一个名为redis-aof-rewrite的子程序
127.0.0.1:6379> BGREWRITEAOF
```

## 客户端连接命令

``` bash
# 查看当前连接到redis的会话：
redis-cli -h 192.168.77.100 -p 7000 client list

# 使用awk格式化输出，生成简易报表：
redis-cli  client list|sed 's/\r//'|awk -F' |:' '{print $1,$2,$5,$6,$9,$19}'|column -t
```

## 参数命令

``` bash
# 批量删除key
cat token.txt |awk '{print "redis-cli del""\t"$1 }'| sh

# 获取所有的参数
redis-cli -h 192.168.77.100 -p 7000 CONFIG GET "*"

# 获取最大内存参数
redis-cli -h 192.168.77.100 -p 7000 CONFIG GET maxmemory
# 返回值 6871947673

# 设置参数，当前生效，无需重启
redis-cli -h 192.168.77.100 -p 7000 CONFIG SET maxmemory 7000000000
redis-cli -h 192.168.77.100 -p 7000 CONFIG GET maxmemory
# 返回值 7000000000

cat redis_7000.conf |grep ^maxmemory
# maxmemory 6871947673
# 配置文件并没有被修改，redis重启参数设置回滚

# 将当前参数刷入配置文件
redis-cli -h 192.168.77.100 -p 7000 CONFIG REWRITE
cat redis_7000.conf |grep ^maxmemory
# maxmemory 7000000000
# 配置文件更新，参数永久设置

# 重置 INFO 命令中的某些统计数据
# 被重置的数据包括：
# Keyspace hits (键空间命中次数)
# Keyspace misses (键空间不命中次数)
# Number of commands processed (执行命令的次数)
# Number of connections received (连接服务器的次数)
# Number of expired keys (过期key的数量)
# Number of rejected connections (被拒绝的连接数量)
# Latest fork(2) time(最后执行 fork(2) 的时间)
# The aof_delayed_fsync counter(aof_delayed_fsync 计数器的值)
redis-cli -h 192.168.77.100 -p 7000 CONFIG RESETSTAT
```

## 底层命令

``` bash
# 获取全部的命令列表
redis-cli -h 192.168.77.100 -p 7000 COMMAND

# 获取这些命令的总数
redis-cli -h 192.168.77.100 -p 7000 COMMAND COUNT

# 获取给定命令的所有键
redis-cli -h 192.168.77.100 -p 7000 COMMAND GETKEYS SET key value
redis-cli -h 192.168.77.100 -p 7000 COMMAND GETKEYS MSET k1 v1 k2 v2 k3 v3

# 获取Redis相应命令描述的数组
redis-cli -h 192.168.77.100 -p 7000 COMMAND INFO del set
# 不负责任的猜测：
#   是命令实现的简单描述
#   如set需要先调用write写
#   然后调用denyoom拒绝被oom
```
## 清楚库命令

``` bash
# 需要将命令流转化成dos格式，然后使用--pipe参数装入到redis实例
echo -e "select 2\nflushdb"|sed 's/$/\r/'|redis-cli -h 192.168.77.100 -p 7000 --pipe

# 删除所有数据库的所有key，全部库的key都会被清空
redis-cli -h 192.168.77.100 -p 7000 FLUSHALL

# 关闭redis服务
redis-cli -h 192.168.77.100 -p 7000 SHUTDOWN

# 默认使用SAVE命令，阻塞服务请求，生成新的RDB，然后关闭redis服务
redis-cli -h 192.168.77.100 -p 7000 SHUTDOWN SAVE

# 服务崩溃命令
redis-cli -h 192.168.77.100 -p 7000 DEBUG SEGFAULT
```

## 服务状态命令

``` bash
TIME 
# 返回当前服务器时间UNIX时间戳，到'1970-01-01 00:00:00'+时区 的间隔秒和微秒

DBSIZE 
# 返回当前数据库的 key 的数量

LASTSAVE 
# 返回上一次成功保存RDB的时间戳，UNIX时间戳，到'1970-01-01 00:00:00'+时区的间隔秒

DEBUG OBJECT key 
# 获取 key 的调试信息
# Value at:0x7f2167c72528 refcount:1
# encoding:embstr serializedlength:17 lru:13993658 lru_seconds_idle:1424
# 返回的信息包括value内存地址、编码方式、序列化长度、lru空闲秒数 等

# 慢日志相关
redis-cli -c -h 192.168.21.101 -p 7000 SLOWLOG LEN
# 查看当前有多少条慢日志
redis-cli -c -h 192.168.21.101 -p 7000 SLOWLOG GET 1
# 获取最后一条慢日志
# 1) 1) (integer) 95            慢日志的ID
#    2) (integer) 1523857790    慢日志UNIX时间戳
#    3) (integer) 13730         慢日志的耗时/微秒
#    4) 1) "ZADD"               慢日志具体操作内容
#       2) "{OPREATE_RECORD...
#       3) "5.6704707E7"
#       4) "{\"aaCreateTime...
#    5) "192.168.1.117:55165"   会话来源
redis-cli -c -h 192.168.21.101 -p 7000 SLOWLOG GET
# 获取全部慢日志
redis-cli -c -h 192.168.21.101 -p 7000 SLOWLOG RESET
# 清空慢日志

# 监控 MONITOR
redis-cli -c -h 192.168.77.100 -p 7000 MONITOR
# 相当于tailf log，将所有到该redis服务器的会话连接全部监控起来
# 1523933983.091873 [0 192.168.77.100:43468] "dbsize"
# 时间戳 [数据库 IP:PORT] "命令"
date -d@1523933983.091873 +%F_%T_%N
# 2018-04-17_10:59:43_091873000
```