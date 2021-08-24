---
title: elk监控容器日志
tags: [elk]
categories: 监控
date: 2020-01-20 11:00:00
---

# 介绍

本章使用`sebp/elk`镜像直接启动elk环境，使用`rtoma/logspout-redis-logstash`镜像收集改镜像运行主机的所有容器日志。

# 搭建步骤

主机环境：

- elk+redis：192.168.40.100
- eureka+logspout：192.168.40.102

整体架构：`eureka`日志源-->`logspout-redis-logstash`收集日志-->`redis`存储日志-->`elk`处理，存储，展示日志

1. 安装`sebp/elk`镜像，并修改logstash配置
2. 安装redis镜像
3. 安装`rtoma/logspout-redis-logstash`镜像
4. 安装eureka
5. 登录kibana查看es索引，并创建kibana日志索引

## sebp/elk安装

```bash
# 创建持久化目录
mkdir -p /data/elk/{elasticsearch,logstash}

# 配置logstash
cd /data/elk/logstash
```

```bash
vim 02-beats-input.conf
# 添加以下内容
```

```bash
input {
  redis {
    host => "192.168.40.100"
    port => "6379"
    data_type => "list"
    key => "logspout"
    codec => "json"
  }
}
```

```bash
vim 30-output.conf
# 添加以下内容
```

```bash
filter {
  # 丢弃[docker][image]包含内容的日志
  if [docker][image] =~ /acs\// or [docker][image] =~ /logstash/ or [docker][image] =~ /zookeeper/ {
    drop {}
  }
  # 合并多行错误日志
  multiline {
    pattern => "(^\s)|(^Caused by)"
    negate => false
    what => "previous"
  }
  # 丢弃[message]包含内容的日志
  if [message] =~ "Xmemcached is stopped at" or [message] =~ "Unable to read additional data from client sessionid" {
    drop {}
  }
  # 移除标签
  mutate {
    remove_field => [ '[docker][labels]' ]
  }
  # 匹配message包含指定字段的日志，添加标签，用于区分正确和错误日志
  grok {
    match => [ "message", "Exception" ]
    add_tag => ["exception-log"]
    tag_on_failure => []
    add_field => { "Levels" => "Errs" }
  }
  grok {
    match => [ "message", "ERROR" ]
    add_tag => ["exception-log"]
    tag_on_failure => []
    add_field => { "Levels" => "Errs" }
  }
}
# 定义索引，存储到es
output {
if "exception-log" in [tags] {
   elasticsearch {
      hosts => ["localhost"]
      manage_template => false
      index => "err-dockerlogs-%{+YYYY.MM.dd}"
      document_type => "%{[@metadata][type]}"
      codec => rubydebug
   }
}
else {
  elasticsearch {
    hosts => ["localhost"]
    manage_template => false
    index => "dockerlogs-%{+YYYY.MM.dd}"
    document_type => "%{[@metadata][type]}"
    codec => rubydebug
  }
}
}
```

```bash
# 启动elk容器
docker run --restart always -p 5601:5601 -p 9200:9200 -p 5044:5044 -e ES_MIN_MEM=128m -e ES_MAX_MEM=2048m -v /data/elk/logstash/:/etc/logstash/conf.d/ -v /data/elk/elasticsearch/:/var/lib/elasticsearch/ -v /etc/localtime:/etc/localtime -it --name elk -d sebp/elk

# 查看容器日志，会有报错：filter/multiline
docker logs -f elk

# 登录容器并安装：logstash-filter-multiline插件
docker exec -it elk bash
# 容器内执行
/opt/logstash/bin/logstash-plugin install logstash-filter-multiline

# 退出容器并重启
exit
# 重启容器
docker restart elk
```



## redis安装

```bash
docker run --restart always --name redis-elk -p 6379:6379 -d redis
```

## rtoma/logspout-redis-logstash安装

```bash
# 启动容器
docker run --restart -d --name "elk-logspout-redis"  --publish=127.0.0.1:8123:80 -v /var/run/docker.sock:/var/run/docker.sock:ro rtoma/logspout-redis-logstash  'redis://192.168.40.100:6379'

# 查看logspout是否收集日志
curl http://127.0.0.1:8123/logs
```

## eureka安装

```bash
# 启动容器
docker run --name eureka -p 8761:8761 -d springcloud/eureka

# 监控容器日志
docker logs -f eureka
```

## kibana设置索引

1. 浏览器登录kibana：http://192.168.40.100:5601
2. 查看elasticsearch索引
3. 创建kibana索引