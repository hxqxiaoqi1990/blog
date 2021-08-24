---
title: 定时查询elk错误日志并报警
tags: [shell脚本]
categories: 脚本
date: 2019-07-09 18:27:43
---



```bash
#!/bin/bash
# 安装邮件工具
if [ ! -e "/tmp/sendEmail-v1.56/sendEmail" ];then
     cd /tmp && wget http://caspian.dotconf.net/menu/Software/SendEmail/sendEmail-v1.56.tar.gz && tar xf sendEmail-v1.56.tar.gz && chmod +x /tmp/sendEmail-v1.56/sendEmail
fi

# 获取elasticsearch指定索引，指向条件，指定时间的错误
data=`curl -s -H "Content-Type:application/json" -XPOST http://172.16.190.240:9200/err-dockerlogs-*/_search -d '{"query": {"bool": {"must": [{"bool": {"should": [{"match_phrase": {"message": "Cause: java.sql.SQLSyntaxErrorException:"}},{"match_phrase": {"message": "### Error updating database"}},{"match_phrase": {"message": "### Cause: java.sql"}},{"match_phrase": {"message": "message:### SQL"}},{"match_phrase": {"message": "SQLException"}}]}},{"range": {"@timestamp": {"gte": "now-5m/m","lte": "now/m","format": "epoch_millis"}}}]}},"from": 0,"size": 5}'`

# 转为可读json格式
jqdata=`echo $data|jq .`

# 获取错误总数
num=`echo $data|cut -d ":" -f 10|cut -d "," -f 1`

# 发送邮件报警
if [ "$num" -gt "0" ];then
	echo $num
	/tmp/sendEmail-v1.56/sendEmail -o message-charset="utf-8" -f 111@111.com -t 222@222.com -s smtp.111.com -u "elk-sql-err" -xu 111@111.com -xp 1111.1234 -m "$jqdata" -o tls=no
fi
```

