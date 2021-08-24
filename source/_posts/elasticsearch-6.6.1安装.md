---
title: elasticsearch安装
tags: [es]
categories: 数据库
date: 2019-08-08 11:00:00
---

# 介绍
Elasticsearch 是一个分布式可扩展的实时搜索和分析引擎,一个建立在全文搜索引擎 Apache Lucene(TM) 基础上的搜索引擎.当然 Elasticsearch 并不仅仅是 Lucene 那么简单，它不仅包括了全文搜索功能，还可以进行以下工作:

1.分布式实时文件存储，并将每一个字段都编入索引，使其可以被搜索。
2.实时分析的分布式搜索引擎。
3.可以扩展到上百台服务器，处理PB级别的结构化或非结构化数据。

# 安装步骤

[先安装：jdk1.8]([[https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85-linux/](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/))

1.下载，解压
2.修改linux内核，设置资源参数
3.修改配置
4.启动elasticsearch、自启动脚本
5.安装中文分词

## 下载，解压

``` bash
# 下载
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-6.6.1.tar.gz

cd /data
tar xf elasticsearch-6.6.1.tar.gz

# 创建用户，需要以普通用户启动
useradd  elk

# 创建目录，用于存放数据和日志
cd /data/elasticsearch-6.6.1
mkdir data logs

```
## 修改linux内核，设置资源参数
``` bash
# 修改内核参数 
cat >> /etc/sysctl.conf << EOF
vm.max_map_count=655360 
EOF
sysctl -p

# 修改服务器资源参数
cat >> /etc/security/limits.conf << EOF 
* soft nofile 655350 
* hard nofile 655350 
EOF

cat >> /etc/security/limits.d/20-nproc.conf << EOF 
elk soft nproc 65536
EOF	
```
## 修改配置

``` bash
cd /data/elasticsearch-6.6.1

vim config/elasticsearch.yml	
# 修改，取消以下配置注释
```

```bash
#数据目录
path.data: /data/elasticsearch-6.6.1/data	
path.logs: /data/elasticsearch-6.6.1/logs
#允许哪个IP访问,0代表所有
network.host: 0.0.0.0		
http.port: 9200
```

```bash
# 附权限
chown -R elk:elk elasticsearch-6.6.1
```



## 启动elasticsearch

``` bash
su elk
cd /data/elasticsearch-6.6.1

# 加-d：后台启动
./bin/elasticsearch
```
启动脚本

``` bash
#!/bin/bash
su -  elk<<!
/data/elasticsearch-6.6.1/bin/elasticsearch -d
exit
!
```

root启动命令

```bash
# 内存最好指定为服务器的一半
su - elk -l -c "cd /data/elasticsearch-6.6.1/bin/ && export JAVA_HOME=/opt/jdk1.8.0_221 && export ES_JAVA_OPTS='-Xms10g -Xmx10g' && ./elasticsearch -d"
```



## 安装中文分词

``` bash
# 下载
wget https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.6.1/elasticsearch-analysis-ik-6.6.1.zip

# 创建ik目录
mkdir /data/elasticsearch-6.6.1/plugins/ik

mv elasticsearch-analysis-ik-6.6.1.zip elasticsearch-6.6.1/plugins/ik

cd /data/elasticsearch-6.6.1/plugins/ik

unzip elasticsearch-analysis-ik-6.6.1.zip
```

# docker部署

elasticsearch部署

```bash
docker run -d --name elasticsearch  -p 9200:9200 -p 9300:9300 -e "discovery.type=single-node" elasticsearch:6.4.0
```

kibana部署

```bash
docker run -p 5601:5601 --name kibana  -e "elasticsearch.hosts=http://localhost:9200" -d kibana:6.4.0
```

