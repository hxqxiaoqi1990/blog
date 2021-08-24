---
title: SkyWalking单点安装
tags: [SkyWalking]
categories: 监控
date: 2020-02-28 15:00:00
---

# 介绍

随着微服务架构的流行，一些微服务架构下的问题也会越来越突出，比如一个请求会涉及多个服务，而服务本身可能也会依赖其他服务，整个请求路径就构成了一个网状的调用链，而在整个调用链中一旦某个节点发生异常，整个调用链的稳定性就会受到影响。

面对以上情况， 我们就需要一些可以帮助理解系统行为、用于分析性能问题的工具，以便发生故障的时候，能够快速定位和解决问题。这时候分布式追踪系统就该闪亮登场了。

<font color='32CD32'>SkyWalking</font>  是针对分布式系统的 APM 系统，也被称为分布式追踪系统

* 全自动探针监控，不需要修改应用程序代码。查看支持的中间件和组件库列表：https://github.com/apache/incubator-skywalking
* 支持手动探针监控, 提供了支持 OpenTracing 标准的SDK。覆盖范围扩大到 OpenTracing-Java 支持的组件。查看OpenTracing组件支持列表：https://github.com/opentracing-contrib/meta
* 自动监控和手动监控可以同时使用，使用手动监控弥补自动监控不支持的组件，甚至私有化组件。
* 纯 Java 后端分析程序，提供 RESTful 服务，可为其他语言探针提供分析能力。
* 高性能纯流式分析

# 安装步骤

## 安装环境

1. 安装：jdk1.8
2. 安装：elasticsearch-6.x
3. 安装：SkyWalking

## 安装jdk1.8

[点击：安装jdk1.8](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/)

## 安装elasticsearch-6.x

[点击：安装elasticsearch-6.x](https://hxqxiaoqi.gitee.io/2019/08/08/elasticsearch-6.6.1安装/)

## 安装SkyWalking

```bash
# 下载
wget https://archive.apache.org/dist/skywalking/6.4.0/apache-skywalking-apm-6.4.0.tar.gz

# 解压
tar xf apache-skywalking-apm-6.4.0.tar.gz -C /opt/

# 修改配置文件
vim /opt/apache-skywalking-apm-bin/config/application.yml

```

```yaml
# 取消es配置的注释，使用es作为存储库
storage:
  elasticsearch:
    nameSpace: ${SW_NAMESPACE:"my-application"}
    clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:192.168.10.131:9200}
    protocol: ${SW_STORAGE_ES_HTTP_PROTOCOL:"http"}
    
# 注释以下配置，h2为SkyWalking自用的存储库，存储在内存中
#  h2:
#    driver: ${SW_STORAGE_H2_DRIVER:org.h2.jdbcx.JdbcDataSource}
#    url: ${SW_STORAGE_H2_URL:jdbc:h2:mem:skywalking-oap-db}
#    user: ${SW_STORAGE_H2_USER:sa}
#    metadataQueryMaxSize: ${SW_STORAGE_H2_QUERY_MAX_SIZE:5000}
```

```bash
# 启动
/opt/apache-skywalking-apm-bin/bin/startup.sh

# 访问
curl http://localhost:8080
```

# 客户端连接测试

<font color='32CD32'>客户端说明：</font>

1. /opt/apache-skywalking-apm-bin/agent/目录是数据收集的服务客户端
2. 要监控哪台服务器上的jar服务，就需要把该目录复制到被监控的服务器上
3. 通过以下方式指定配置启动后，需要再访问被监控的服务，skywalking上才可接收到该请求信息。

<font color='32CD32'>方式一：手动指定配置启动</font>

```bash
java -javaagent:/opt/apache-skywalking-apm-bin/agent/skywalking-agent.jar \
-Dskywalking.agent.service_name=eureka \
-Dskywalking.collector.backend_service=localhost:11800 \
-jar eureka-1.0.0-SNAPSHOT.jar
```

- -javaagent：指定agent地址
- -Dskywalking.agent.service_name：指定要运行的服务名称，可自由定义
- -Dskywalking.collector.backend_service：skywalking服务端连接地址
- eureka-1.0.0-SNAPSHOT.jar：为测试用的jar包

<font color='32CD32'>方式二：修改配置文件启动</font>

```bash
vim /opt/apache-skywalking-apm-bin/agent/config/agent.config
# 修改以下配置
```

```bash
# 定义服务名
agent.service_name=${SW_AGENT_NAME:mmhsy-eureka-1.0.0-SNAPSHOT}

# 修改连接skywalking地址
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:localhost:11800}
```

```bash
# 启动
java -javaagent:/opt/apache-skywalking-apm-bin/agent/skywalking-agent.jar -jar eureka-1.0.0-SNAPSHOT.jar
```

# 配置文件说明

## application.yml

OAP(Collector)链路数据归集器，主要用于数据落地，大部分都会选择 Elasticsearch 6，OAP配置文件为 /opt/apache-skywalking-apm-6.2.0/config/application.yml，配置单点的 OAP(Collector)配置如下：

```yaml
cluster:
   # 单节点模式
   standalone:
   # zk用于管理collector集群协作.
   # zookeeper:
      # 多个zk连接地址用逗号分隔.
      # hostPort: localhost:2181
      # sessionTimeout: 100000
   # 分布式 kv 存储设施，类似于zk，但没有zk重型（除了etcd，consul、Nacos等都是类似功能）
   # etcd:
      # serviceName: ${SW_SERVICE_NAME:"SkyWalking_OAP_Cluster"}
      # 多个节点用逗号分隔, 如: 10.0.0.1:2379,10.0.0.2:2379,10.0.0.3:2379
      # hostPort: ${SW_CLUSTER_ETCD_HOST_PORT:localhost:2379}
core:
   default:
      # 混合角色：接收代理数据，1级聚合、2级聚合
      # 接收者：接收代理数据，1级聚合点
      # 聚合器：2级聚合点
      role: ${SW_CORE_ROLE:Mixed} # Mixed/Receiver/Aggregator
 
       # rest 服务地址和端口
      restHost: ${SW_CORE_REST_HOST:localhost}
      restPort: ${SW_CORE_REST_PORT:12800}
      restContextPath: ${SW_CORE_REST_CONTEXT_PATH:/}
 
      # gRPC 服务地址和端口
      gRPCHost: ${SW_CORE_GRPC_HOST:localhost}
      gRPCPort: ${SW_CORE_GRPC_PORT:11800}
 
      downsampling:
      - Hour
      - Day
      - Month
 
      # 设置度量数据的超时。超时过期后，度量数据将自动删除.
      # 单位分钟
      recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:90}
 
      # 单位分钟
      minuteMetricsDataTTL: ${SW_CORE_MINUTE_METRIC_DATA_TTL:90}
 
      # 单位小时
      hourMetricsDataTTL: ${SW_CORE_HOUR_METRIC_DATA_TTL:36}
 
      # 单位天
      dayMetricsDataTTL: ${SW_CORE_DAY_METRIC_DATA_TTL:45}
 
      # 单位月
      monthMetricsDataTTL: ${SW_CORE_MONTH_METRIC_DATA_TTL:18}
 
storage:
 
   elasticsearch:
 
      # elasticsearch 的集群名称
      nameSpace: ${SW_NAMESPACE:"local-ES"}
 
      # elasticsearch 集群节点的地址及端口
      clusterNodes: ${SW_STORAGE_ES_CLUSTER_NODES:192.168.2.10:9200}
 
      # elasticsearch 的用户名和密码
      user: ${SW_ES_USER:""}
      password: ${SW_ES_PASSWORD:""}
 
      # 设置 elasticsearch 索引分片数量
      indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:2}
 
      # 设置 elasticsearch 索引副本数
      indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:0}
 
      # 批量处理配置
      # 每2000个请求执行一次批量
      bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:2000}
 
      # 每 20mb 刷新一次内存块
      bulkSize: ${SW_STORAGE_ES_BULK_SIZE:20}
 
      # 无论请求的数量如何，每10秒刷新一次堆
      flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10}
 
      # 并发请求的数量
      concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2}
 
      # elasticsearch 查询的最大数量
      metadataQueryMaxSize: ${SW_STORAGE_ES_QUERY_MAX_SIZE:5000}
 
      # elasticsearch 查询段最大数量
      segmentQueryMaxSize: ${SW_STORAGE_ES_QUERY_SEGMENT_SIZE:200}
```

## webapp.yml

 Skywalking 的 WebApp 主要是用来展示落地的数据，因此只需要配置 Web 的端口及获取数据的 OAP(Collector)的IP和端口，webApp 配置文件地址为  /opt/apache-skywalking-apm-6.2.0/webapp/webapp.yml 配置如下：

```yaml
server:
  port: 9000
collector:
  path: /graphql
  ribbon:
    ReadTimeout: 10000
    # 指向所有后端collector 的 restHost:restPort 配置，多个使用, 分隔
    listOfServers: localhost:12800
 
security:
  user:
    # username
    admin:
    # password
    password: admin
```

## agent.config

Skywalking 的 Agent 主要用于收集和发送数据到 OAP(Collector)，因此需要进行配置 Skywalking OAP(Collector)的地址，Agent 的配置文件地址为  /opt/apache-skywalking-apm-6.2.0/agent/config/agent.config，配置如下：

```bash
# 设置Agent命名空间，它用来隔离追踪和监控数据，当两个应用使用不同的名称空间时，跨进程传播链会中断。
agent.namespace=${SW_AGENT_NAMESPACE:default-namespace}
 
# 设置服务名称，会在 Skywalking UI 上显示的名称
agent.service_name=${SW_AGENT_NAME:Your_ApplicationName}
 
# 每 3秒采集的样本跟踪比例，如果是负数则表示 100%采集
agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:-1}
 
# 启用 Debug ，如果为 true 则将把所有检测到的类文件保存在"/debug"文件夹中
# agent.is_open_debugging_class = ${SW_AGENT_OPEN_DEBUG:true}
 
# 后端的 collector 端口及地址
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:192.168.2.215:11800}
 
# 日志级别
logging.level=${SW_LOGGING_LEVEL:DEBUG}
```

