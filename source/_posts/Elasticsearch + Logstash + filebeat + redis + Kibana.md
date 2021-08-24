---
title: ELK部署
tags: [elk]
categories: 监控
date: 2019-09-07 21:44:00
---


# 介绍
ELK是Elasticsearch + Logstash + Kibana 这种架构的简写。这是一种日志分平台析的架构。从前我们用shell三剑客(grep, sed, awk)来分析日志, 虽然也能对付大多数场景，但当日志量大，分析频繁，并且使用者可能不会shell三剑客的情况下， 配置方便，使用简单，并且分析结果更加直观的工具(平台)就诞生了，它就是ELK。 ELK是开源的，并且社区活跃，用户众多。 
# 架构说明
1 Elasticsearch + Logstash + Kibana
这是一种最简单的架构。这种架构，通过logstash收集日志，Elasticsearch分析日志，然后在Kibana(web界面)中展示。这种架构虽然是官网介绍里的方式，但是往往在生产中很少使用。

2 Elasticsearch + Logstash + filebeat + Kibana
与上一种架构相比，这种架构增加了一个filebeat模块。filebeat是一个轻量的日志收集代理，用来部署在客户端，优势是消耗非常少的资源(较logstash)， 所以生产中，往往会采取这种架构方式，但是这种架构有一个缺点，当logstash出现故障， 会造成日志的丢失。

3 Elasticsearch + Logstash + filebeat + redis(也可以是其他中间件，比如kafka) + Kibana
这种架构是上面那个架构的完善版，通过增加中间件，来避免数据的丢失。当Logstash出现故障，日志还是存在中间件中，当Logstash再次启动，则会读取中间件中积压的日志。目前我司使用的就是这种架构，我个人也比较推荐这种方式。

# 安装步骤
## 安装jdk1.8
elasticsearch、logstash服务都需要jdk环境

[跳转 jdk安装教程](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85-linux/)
## 安装redis
[跳转 redis安装教程](https://hxqxiaoqi.gitee.io/2019/09/04/redis-5.0.2%E5%AE%89%E8%A3%85/)

## 安装elasticsearch
[跳转 elasticsearch安装教程](https://hxqxiaoqi.gitee.io/2019/08/08/elasticsearch-6.6.1%E5%AE%89%E8%A3%85/)

## 安装kibana

``` bash
cd /data

# 下载
wget https://artifacts.elastic.co/downloads/kibana/kibana-6.6.1-linux-x86_64.tar.gz

# 解压
tar xf kibana-6.6.1-linux-x86_64.tar.gz
cd kibana-6.6.1-linux-x86_64

vim config/kibana.yml
# 取消以下注释，并修改对应IP
```

``` yml
# 服务端口
server.port: 5601
# 指明服务运行的地址
server.host: "192.168.182.100"
# 指明elasticsearch运行的地址和端口
elasticsearch.hosts: ["http://192.168.182.100:9200"]
# 指明kibana使用的索引，这个是自定义的
kibana.index: ".kibana"
```

``` bash
# 启动，如果elasticsearch有kibana数据，则配置成功
/data/kibana-6.6.1-linux-x86_64/bin/kibana
```

## 安装filebeat
该组件为日志收集服务，安装在需要收集日志的服务器上

``` bash
# 下载
wget https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-6.6.1-linux-x86_64.tar.gz

# 解压
tar xf filebeat-6.6.1-linux-x86_64.tar.gz
cd filebeat-6.6.1-linux-x86_64

vim filebeat.yml
# 配置以下设置
```
该配置在input中定义多个日志路径，用于之后的logstash过滤，匹配不同索引

``` yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/*.log
  fields:
    log_source: sys

- type: log
  enabled: true
  paths:
    - /var/log/httpd/*_log
  fields:
    log_source: nginx

output.redis:  
  hosts: ["192.168.182.100:6379"]
  password: ""
  db: 0
  key: "syslog"
```
日志输入：
filebeat.inputs: 模块用来指定日志文件的来源
type:  指定日志类型，在这里是log， 应该也可以是json。
paths指定日志文件路径。
fields: 是打标记，主要为了后面日志分析查找的方便，存储的时候也会根据fields分类存储，相同fields的数据存在同一个redis key中

日志输出
output.redis：指定输出到redis
hosts：指定redis主机，可以指定多台。
password：redis密码，redis默认没有密码，在这里设为空就行
key：指定存入redis中的key
db: 指定存在redis中的db编号(redis默认安装有16个databases，0~15， 默认是存储在db0中的)


``` bash  
# 启动
./filebeat -e -c filebeat.yml
```


## 安装logstash

``` bash
# 下载
wget https://artifacts.elastic.co/downloads/logstash/logstash-6.6.1.tar.gz
tar xf logstash-6.6.1.tar.gz
cd logstash-6.6.1

# 创建配置文件,配置以下设置
vim config/syslog.conf
```
该配置在output中使用fields标签匹配不同索引，从而创建俩个不同的索引给kibana使用

``` yml
input {
  redis {
    host => "192.168.182.100"
    port => 6379
    data_type => "list"
    key => "syslog"
    db => 0
  }
}

output {
  if [fields][log_source] == 'sys' {
    elasticsearch {
      hosts => ["http://192.168.182.100:9200"]
      index => "syslog-%{+YYYY.MM.dd}"
      id => "syslog_id"
    }
  }

  if [fields][log_source] == 'nginx' {
    elasticsearch {
      hosts => ["http://192.168.182.100:9200"]
      index => "nginxlog-%{+YYYY.MM.dd}"
      id => "nginx_id"
    }
  } 
}
```

1.key => "syslog" <font color='90EE90'>对应filebeat.yml配置中，redis设置的key</font>
2.if [fields][log_source] == 'sys' <font color='90EE90'>对应filebeat.yml配置中，input设置fields标签</font>
3.index => "nginxlog-%{+YYYY.MM.dd}" <font color='90EE90'>创建elasticsearch索引</font>


``` bash
# 启动
./bin/logstash -f config/syslog.conf
```

## 访问kibana，并添加索引

访问：IP:5601，没有密码，密码认证需要收费，可以使用nginx代理认证

创建索引：
 1. 点击 management
 2. 点击 Index Patterns
 3. 点击 Create index pattern
 4. 在当前配置页中会显示能添加的索引，如果没有显示，请查看logstash和filebeat配置
 5. 添加索引：nginxlog-* ， 该索引为elasticsearch中的索引
 6. 下一步：选择@timestamp
 7. 点击：Create index pattern 创建成功

## 查看日志

 1. 点击：Discover
 2. 当前页就是搜索日志页面，有切换索引，添加搜索标签，选择时间范围等功能

## kibana汉化
Kibana在6.7版本以上，支持了多种语言。并且自带在安装包里，之前版本需要下载中文包安装。

Kibana在6.7版本以下

``` bash
wget https://mirrors.yangxingzhen.com/kibana/Kibana_Hanization.tar.gz

tar xf Kibana_Hanization.tar.gz
cd Kibana_Hanization/old/

python main.py /data/kibana-6.6.1-linux-x86_64
```

Kibana在6.7版本以上

``` bash
vim /data/kibana-6.6.1-linux-x86_64/config/kibana.yml
# 取消注释
i18n.locale: "zh-CN"
```

