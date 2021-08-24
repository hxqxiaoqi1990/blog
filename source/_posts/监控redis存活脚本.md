---
title: 监控redis存活脚本
tags: [shell脚本]
categories: 脚本
date: 2020-01-23 17:58:55
---



# 介绍

redis监控状态脚本

# 脚本

```bash
#!/bin/bash
#redis主从切换报警脚本
#sendEmail命令下载链接：http://caspian.dotconf.net/menu/Software/SendEmail/sendEmail-v1.56.tar.gz

#获取redis主从状态值
role=`/data/redis32/bin/redis-cli -p 6379 info replication|grep role|cut -d ":" -f 2|tr -d -c 'a-zA-z'`

#获取本机IP
localIP=`/sbin/ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"`

#改文件用于判断redis状态
if [ ! -e "/tmp/redis_err" ];then
	echo 0 > /tmp/redis_err
fi
dir="/tmp/redis_err"

#拥有判断是否执行过邮件发送，如果存在，则不发送邮件，需要手动删除
err=`cat /tmp/redis_err`

if [ "$role" == "slave" ];then
    if [ ! "$err" == 1 ];then
        /usr/local/bin/sendEmail -o message-charset=utf8 -f 123.com -t 123.com  -s smtp.mamahao.com -u 'redis主从切换' -xu '123.com' -xp '123.1234' -m "IP：$localIP date：`date` redis主切换为从"
        echo 1 > $dir
    fi
elif [ "$role" == "master" ];then
    if [ "$err" == 0 ];then
        echo 0 > $dir
    elif [ ! "$err" == 0 ];then
        /usr/local/bin/sendEmail -o message-charset=utf8 -f 123.com -t 123.com  -s smtp.mamahao.com -u 'redis主从切换' -xu '123.com' -xp '123.1234' -m "IP：$localIP date：`date` redis从切换为主"
        echo 0 > $dir
    fi
else
    if [ ! "$err" == 2 ];then
        /usr/local/bin/sendEmail -o message-charset=utf8 -f 123.com -t 123.com  -s smtp.mamahao.com -u 'redis故障' -xu '123.com' -xp '123.1234' -m "IP：$localIP date：`date` 故障：无法获取主从状态"
        echo 2 > $dir
    fi
fi
```

