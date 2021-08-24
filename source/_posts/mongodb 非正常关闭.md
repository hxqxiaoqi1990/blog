---
title: mongodb 非正常关闭
tags: [mongodb]
categories: 数据库
date: 2020-01-01 11:00:00
---

# 问题

mongodb 没有一个正常的关闭指令，经常会遇到服务器重启，或磁盘满了等问题，导致mongodb崩溃，无法启动。

<font color='32CD32'>解决方法</font>

1. 删除 mongodb data目录下的`mongod.lock` 文件
2. 启动修复模式：/opt/mongodb/bin/mongod -f  /opt/mongodb/mongodb.conf --repair
3. 修复后再次正常启动：/opt/mongodb/bin/mongod -f  /opt/mongodb/mongodb.conf 

如果以上方法无效，需要删除 `data/journal` 目录下的文件再次执行以上操作启动。

mongodb的`journal`，简单来说就是用于数据故障恢复和持久化数据的，它以日志方式来记录。从1.8版本开始有此功能，2.0开始默认打开此功能，但32位的系统是默认关闭的。journal除了故障恢复的作用之外，还可以提高写入的性能，批量提交（batch-commit），journal一般默认100ms刷新一次，在这个过程中，所有的写入都可以一次提交，是单事务的，全部成功或者全部失败，刷新时间，可以更改，范围是2-300ms。当系统非正常情况下突然挂掉，再次启动时候mongodb就会从journal日志中恢复数据，而确保数据不丢失，最多丢失s级别的数据。
