---
title: canal adapter（同步到es）安装使用
tags: [canal]
categories: 中间件
date: 2020-03-18 20:00:00
---

# 介绍

canal 1.1.1版本之后, 增加客户端数据落地的适配及启动功能, 目前支持功能:

- 客户端启动器
- 同步管理REST接口
- 日志适配器, 作为DEMO
- 关系型数据库的数据同步(表对表同步), ETL功能
- HBase的数据同步(表对表同步), ETL功能
- ElasticSearch多表数据同步,ETL功能

# 安装

请先安装以下服务：

1. [安装高可用教程](https://hxqxiaoqi.gitee.io/2020/01/20/canal高可用安装/)
2. [安装elasticsearch教程](https://hxqxiaoqi.gitee.io/2019/08/08/elasticsearch-6.6.1安装/)

## 安装canal adapter适配器

```bash
# 下载地址
wget https://github.com/alibaba/canal/releases/download/canal-1.1.4/canal.deployer-1.1.4.tar.gz

tar xf canal.deployer-1.1.4.tar.gz -C /data/canal.deployer
vim /data/canal.deployer/conf/application.yml
# 修改以下配置
```

```yaml
server:
  port: 8081
spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: GMT+8
    default-property-inclusion: non_null

canal.conf:
  mode: tcp # kafka rocketMQ
#  canalServerHost为单机模式，zookeeperHosts为集群模式
#  canalServerHost: 127.0.0.1:11111
  zookeeperHosts: 192.168.40.101:2181
#  mqServers: 127.0.0.1:9092 #or rocketmq
#  flatMessage: true
  batchSize: 500
  syncBatchSize: 1000
  retries: 0
  timeout:
  accessKey:
  secretKey:
# 源数据库信息
  srcDataSources:
    defaultDS:
      url: jdbc:mysql://192.168.40.100:3306/test?useUnicode=true
      username: root
      password: 123123
  canalAdapters:
  # instance的名称
  - instance: elastic
    groups:
    - groupId: g1
      outerAdapters:
      - name: logger
      # elasticsearch集群信息，9300需要es配置相应端口
      - name: es
        hosts: 192.168.40.101:9300
        properties:
          # mode: transport # or rest
          # security.auth: test:123456
          cluster.name: canal-es
```

创建mysql--elasticsearch映射表文件

```bash
vim /data/canal.deployer/conf/es/test.yml
```

```yaml
dataSourceKey: defaultDS
destination: elastic
groupId:
esMapping:
  _index: test
  _type: _doc
  _id: _id
  upsert: true
  sql: "select a.id as _id,a.name,a.address from test a"
  commitBatch: 3000
```

mysql表结构

```sql
CREATE TABLE `test` (
  `id` int(11) NOT NULL,
  `name` varchar(200) NOT NULL,
  `address` varchar(1000) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

elasticsearch索引创建

```json
curl -XPUT http://127.0.0.1:9200/test -d 
'{
    "mappings":{
        "_doc":{
            "properties":{
                "name":{
                    "type":"text"
                },
                "address":{
                    "type":"text"
                }
            }
        }
    }
}'
```

elastic配置文件

```yaml
cluster.name: canal-es
node.name: node-1
network.host: 192.168.40.101
http.port: 9200
discovery.zen.ping.unicast.hosts: ["192.168.40.101"]
transport.tcp.port: 9300
transport.tcp.compress: true
http.cors.enabled: true
http.cors.allow-origin: "*"
```

启动

```bash
/data/canal.deployer/bin/startup.sh
/data/canal.deployer/bin/stop.sh
```

测试

1. 在mysql插入记录
2. 查看es文档数据