---
title: elasticstarch执行语句
tags: [es]
categories: 数据库
date: 2020-01-22 11:00:00
---

# 介绍

索引是ElasticSearch存放数据的地方，可以理解为关系型数据库中的一个数据库。

<font color='32CD32'>es与数据库对应关系</font> 

| 数据库类型    | 库名      | 表名   | 记录      | 字段    |
| :------------ | --------- | ------ | --------- | ------- |
| Relational DB | Databases | Tables | Rows      | Columns |
| Elasticsearch | Indices   | Types  | Documents | Fields  |

<font color='32CD32'>文档元数据</font>

一个文档不只有数据。它还包含了元数据(metadata)——关于文档的信息。三个必须的元数据节点是（其实就是数据库字段）：

| **节点** | 说明               |
| -------- | ------------------ |
| _index   | 文档存储的地方     |
| _type    | 文档代表的对象的类 |
| _id      | 文档的唯一标识     |

<font color='32CD32'>索引创建原则</font> 

1. 类似的数据放在一个索引，非类似的数据放不同索引：product index（包含了所有的商品），sales index（包含了所有的商品销售数据），inventory index（包含了所有库存相关的数据）。如果你把比如product，sales，human resource（employee），全都放在一个大的index里面，比如说company index，不合适的。
2. index中包含了很多类似的document：类似是什么意思，其实指的就是说，这些document的fields很大一部分是相同的，你说你放了3个document，每个document的fields都完全不一样，这就不是类似了，就不太适合放到一个index里面去了。
3. 索引名称必须是小写的，不能用下划线开头，不能包含逗号：product，website，blog

# 语法示例

## 运维相关

<font color='32CD32'>查看ES集群中所有节点信息，以及各个节点内存和CPU相关的指标</font>

```bash
curl -X GET 'http://192.168.40.100:9200/_cat/nodes?v'
```

<font color='32CD32'>查看整个ES集群的状态，以及节点、分片等信息</font>

```bash
curl -XGET 'http://192.168.40.100:9200/_cluster/health?pretty'  
```

<font color='32CD32'>如果集群状态不正常了（status是yellow或者red），可以通过以下命令查看具体是哪个index中的哪些shard出问题了：</font>

```bash
curl -XGET 'http://192.168.40.100:9200/_cluster/health?pretty&level=indices' 
curl -XGET 'http://192.168.40.100:9200/_cluster/health?pretty&level=shards'
```

<font color='32CD32'>列出ES集群中所有的index信息</font>

```bash
curl -XGET 'http://192.168.40.100:9200/_cat/indices?v'
```

<font color='32CD32'>显示索引的别名信息、过滤器和路由信息</font>

```bash
GET _cat/aliases?v
```

<font color='32CD32'>查看每个节点的分片数量以及每个节点的磁盘空间使用情况</font>

```bash
GET _cat/allocation?v
```

<font color='32CD32'>查看索引或集群的文档数量</font>

```bash
# 查询全部
GET _cat/count?v

# 查询单个索引
GET _cat/count/books?v
```

<font color='32CD32'>查看每个数据节点上被fielddata所使用的堆内存大小</font>

```
GET _cat/fielddata?v
```

<font color='32CD32'>显示master节点的id、ip和节点名</font>

```
GET _cat/master?v
```

<font color='32CD32'>返回集群中各节点信息</font>

```
GET _cat/nodes?v
```

<font color='32CD32'>查看节点所运行插件信息</font>

```
GET /_cat/plugins?v
```

<font color='32CD32'>查看索引分片恢复进度</font>

```
GET /_cat/recovery?v
```

<font color='32CD32'>查看集群中的快照库</font>

```
GET /_cat/repositories?v
```

<font color='32CD32'>查看集群每个节点的线程池统计信息</font>

```
GET /_cat/thread_pool?v
```

<font color='32CD32'>查看集群中每个节点的分片信息，包括分片名称、编号、是否是主分片、状态、文档数据、空间大小、所有节点ip、节点名称</font>

```
GET /_cat/shards?v
```

<font color='32CD32'>查看索引的segment信息，注意，索引数据实际上是以一个个segment的方式进行存储的</font>

```
GET /_cat/segments?v
```

<font color='32CD32'>查看集群的健康状态</font>

```
GET /_cluster/health
```

<font color='32CD32'>返回集群的完整状态信息。</font>

```
GET /_cluster/state
```

<font color='32CD32'>返回集群的完整状态信息。</font>

```
GET /_cluster/state/version
```

<font color='32CD32'>获取各种统计数据。包括两部分数据：</font>

- 索引层面：分片数、存储大小、内存使用等；
- 节点层面：节点数量、节点角色、操作系统、jvm信息、内存、CPU、插件等；

```
GET /_cluster/stats
```

<font color='32CD32'>明确地执行集群重新路由分配命令。</font>

```
POST /_cluster/reroute
```

<font color='32CD32'>更新集群中的配置，如果是永久配置，需要重启集群；临时配置的訞 不不需要重启集群</font>

```yaml
PUT /_cluster/settings
{
  "persistent": {
    "discovery.zen.minimum_master_nodes":1
  }
}
```

<font color='32CD32'>统计集群中一个或多个节点的统计信息。</font>

```yaml
GET /_nodes
```

或：

```yaml
GET /_nodes/es01,es02
```

<font color='32CD32'>获取集群中正在节点中执行的任务信息。</font>

```
GET /_tasks
```

<font color='32CD32'>查看分片没有被分配的原因，比如通过`GET /_cat/shards?v`看到某个索引没有被分配，就可以使用下面的命令来查看没有被分配的原因。</font>

```yaml
GET /_cluster/allocation/explain
{
  "index":"twitter",
  "shard":0,
  "primary":true
}
```

<font color='32CD32'>创建索引</font>

```bash
curl -X PUT "http://192.168.40.100:9200/索引名?pretty"
```

<font color='32CD32'>删除索引</font>

```bash
curl -XDELETE 'http://192.168.40.100:9200/索引名'
```

<font color='32CD32'>关闭索引</font>

```bash
curl -XPOST 'http://192.168.40.100:9200/索引名/_close'
```

<font color='32CD32'>开启索引</font>

```bash
curl -XPOST 'http://192.168.40.100:9200/索引名/_open'
```

<font color='32CD32'>动态修改ES相关配置</font>

```bash
curl -XPUT  'http://192.168.40.100:9200/索引名/_settings' -d '{"index":{"refresh_interval":"60s"}}'
```

<font color='32CD32'>设置最大查询条数</font>

```bash
curl -XPUT 'http://192.168.40.100:9200/索引名/_settings' -d'{"index":{"max_result_window":1000000}}'
```

<font color='32CD32'>设置默认副本和分片</font>

```bash
curl -XPOST http://192.168.40.100:9200/_template/template_http_request_record -H 'Content-Type: application/json' -d '
{
  "order": 0,
  "index_patterns": [
    "*"	#匹配的索引
  ],
  "settings": {
    "index": {
      "number_of_shards": "5",
      "number_of_replicas": "0"
    }
  },
  "mappings": {},
  "aliases": {}
}'
```

<font color='32CD32'>变更之前的副本数</font>

```bash
curl -XPUT http://192.168.40.100:9200/_settings -H 'Content-Type: application/json' -d '{"index":{"number_of_replicas":0}}'
```



## 基本语法

[查询语法参考该文档](https://www.cnblogs.com/heqiuyong/p/10351176.html)

<font color='32CD32'>查询文档</font>

```bash
curl -XGET 'http://192.168.40.100:9200/索引名/类型/ID'
```

<font color='32CD32'>新增文档</font>

```bash
curl -XPOST 'http://192.168.40.100:9200/索引名/类型/ID?pretty' -d '
{
"zbbm": "10010103Y",
"zzid": "0600",
"tjqj": "201712",
"gwxd": 3124543444.71,
"nbys":2073344433.12
}'
```

<font color='32CD32'>删除文档</font>

```bash
curl -XDELETE 'http://192.168.40.100:9200/索引名/类型/ID'
```

<font color='32CD32'>修改文档</font>

```bash
curl -XPOST 'http://192.168.40.100:9200/索引名/类型/ID/_update' -d '
{
"doc":{
"gwxd":1237674.23,
"nbys": 123233221212
}
}'
```

