---
title: nginx日志备份脚本
tags: [shell脚本]
categories: 脚本
date: 2019-06-04 19:40:43
---

# 定时任务配置

``` bash
#定时任务
#30 0 * * * /usr/bin/bash /usr/local/nginx/logs/nginx-log.sh &> /dev/null
```

# 备份脚本配置
``` bash
#!/bin/bash
#此脚本用于自动分割Nginx的日志，包括access.log和error.log
#将前一天的access.log重命名为access-xxxx-xx-xx.log格式，并重新打开日志文件

#Nginx日志文件所在目录
LOG_PATH=/usr/local/nginx/logs/

#获取昨天的日期
YESTERDAY=$(date -d "yesterday" +%Y-%m-%d)
WEEK=$(date -d -7day +%Y-%m-%d)

#获取pid文件路径
PID=/usr/local/nginx/logs/nginx.pid

#创建备份目录
mkdir -p /usr/local/nginx/logs/bak

#分割日志
mv ${LOG_PATH}access.log ${LOG_PATH}bak/access-${YESTERDAY}.log
mv ${LOG_PATH}error.log ${LOG_PATH}bak/error-${YESTERDAY}.log

#删除一周前日志
rm -rf ${LOG_PATH}bak/access-${WEEK}.log
rm -rf ${LOG_PATH}bak/error-${WEEK}.log

#向Nginx主进程发送USR1信号，重新打开日志文件
kill -USR1 `cat ${PID}`
```
