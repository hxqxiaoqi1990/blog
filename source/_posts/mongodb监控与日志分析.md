---
title: mongodb监控与日志分析
tags: [mongodb]
categories: 数据库
date: 2019-11-13 14:22:00
---


# 介绍
mongodb自带监控命令
# 监控命令
## mongostat
mongostat是mongodb自带的状态检测工具，在命令行下使用，会间隔固定时间获取mongodb的当前运行状态，并输出。

**常用命令格式：**

mongostat --host 192.168.1.100:27017 -uroot -p123456 --authenticationDatabase admin
参数说明：
host:指定IP地址和端口，也可以只写IP，然后使用--port参数指定端口号
-u： 如果开启了认证，则需要在其后填写用户名
-p: 不用多少，肯定是密码
--authenticationDatabase：若开启了认证，则需要在此参数后填写认证库（注意是认证上述账号的数据库）

**命令输出格式解释**

**insert/s :** 官方解释是每秒插入数据库的对象数量，如果是slave，则数值前有*,则表示复制集操作
**query/s :** 每秒的查询操作次数
**update/s :** 每秒的更新操作次数
**delete/s :** 每秒的删除操作次数
**getmore/s:** 每秒查询cursor(游标)时的getmore操作数
**command:** 每秒执行的命令数，在主从系统中会显示两个值(例如 3|0),分表代表 本地|复制 命令
注： 一秒内执行的命令数比如批量插入，只认为是一条命令（所以意义应该不大）
**dirty:** 仅仅针对WiredTiger引擎，官网解释是脏数据字节的缓存百分比
**used:** 仅仅针对WiredTiger引擎，官网解释是正在使用中的缓存百分比
**flushes:**
For WiredTiger引擎：指checkpoint的触发次数在一个轮询间隔期间
For MMAPv1 引擎：每秒执行fsync将数据写入硬盘的次数
注：一般都是0，间断性会是1， 通过计算两个1之间的间隔时间，可以大致了解多长时间flush一次。flush开销是很大的，如果频繁的flush，可能就要找找原因了
**vsize:** 虚拟内存使用量，单位MB （这是 在mongostat 最后一次调用的总数据）
**res:**  物理内存使用量，单位MB （这是 在mongostat 最后一次调用的总数据）
注：这个和你用top看到的一样, vsize一般不会有大的变动， res会慢慢的上升，如果res经常突然下降，去查查是否有别的程序狂吃内存。

**qr:** 客户端等待从MongoDB实例读数据的队列长度
**qw：** 客户端等待从MongoDB实例写入数据的队列长度
**ar:** 执行读操作的活跃客户端数量
**aw:** 执行写操作的活客户端数量
注：如果这两个数值很大，那么就是DB被堵住了，DB的处理速度不及请求速度。看看是否有开销很大的慢查询。如果查询一切正常，确实是负载很大，就需要加机器了
**netIn:** MongoDB实例的网络进流量
**netOut:** MongoDB实例的网络出流量
注：此两项字段表名网络带宽压力，一般情况下，不会成为瓶颈
**conn:** 打开连接的总数，是qr,qw,ar,aw的总和
注：MongoDB为每一个连接创建一个线程，线程的创建与释放也会有开销，所以尽量要适当配置连接数的启动参数，maxIncomingConnections，阿里工程师建议在5000以下，基本满足多数场景

## mongotop
mongotop用来跟踪MongoDB的实例， 提供每个集合的统计数据。默认情况下，mongotop每一秒刷新一次。

**输出字段说明**

ns：数据库命名空间，后者结合了数据库名称和集合。
db：数据库的名称。名为 . 的数据库针对全局锁定，而非特定数据库。
total：mongod在这个命令空间上花费的总时间。
read：在这个命令空间上mongod执行读操作花费的时间。
write：在这个命名空间上mongod进行写操作花费的时间。

# 日志分析
**日志信息的格式**
<日志时间> <严重级别> <信息所属分类> [<内容>] <消息>

**日志信息严重级别**

级别	级别描述
F	Fatal
E	Error
W	Warning
I	Informational, for Verbosity Level of 0
D	Debug, for All Verbosity Levels > 0

**信息所属分类**

登入信息
ACCESS：登入访问相关的信息，例如登录验证情况。

命令信息	
COMMAND：数据库执行命令相关信息，例如，查询。

控制管理信息
CONTROL：记录控制管理相关的信息，例如数据库初始化。

FTDC信息
FTDC（full-time diagnostic data ）：全程检测数据信息，例如Server的状态统计信息。

索引信息
INDEX：索引相关信息，例如索引的创建过程信息。

网络信息
NETWORK：网络相关信息，例如网络连接信息。

查询信息
QUERY：查询相关信息，例如查询计划信息。

副本集信息
REPL：副本集相关信息，例如副本集初始过程、心跳、回滚等信息

分片信息
SHARDING：分片相关信息，例如mongos的启动信息

存储信息
STORAGE：存储相关信息，例如将 storage 层的数据刷入磁盘的信息。

还原信息
RECOVERY：还原活动相关的信息

日志信息
JOURNAL：日志相关的信息

写操作信息
WRITE：写操作相关的信息，例如更新（update）的命令。