---
title: flink之REST API调用
tags: [flink]
categories: 大数据
date: 2020-06-05 13:00:00
---

# 介绍

监视API由作为*Dispatcher*一部分运行的Web服务器支持。默认情况下，此服务器在post侦听`8081`，可以`flink-conf.yaml`通过via 进行配置`rest.port`。请注意，监视API Web服务器和Web仪表板Web服务器当前是相同的，因此可以在同一端口一起运行。但是，它们会响应不同的HTTP URL。

如果有多个Dispatcher（用于高可用性），则每个Dispatcher将运行自己的监视API实例，该实例提供有关在Dispatcher被选为集群负责人时已完成和正在运行的作业的信息。

官网地址：https://ci.apache.org/projects/flink/flink-docs-stable/monitoring/rest_api.html

# 示例

```bash
# 提交任务
# programArgsList: jar包启动的参数，指定允许环境
# entryClass：jar定义的类，可以根据类指定允许不同任务
curl -XPOST -H "Content-Type:application/java-archive" http://192.168.10.111:8081/jars/692d1b44-8f99-4f42-abe0-21579238b57d_mmhsy-dw-flink-1.0-SNAPSHOT.jar/run -d '{"programArgsList":[   "--profiles","test"],"entryClass":"com.mmhsy.flink.member.job.MemberStarMemberInviteJob"}'

# 查看job
curl -XGET http://192.168.10.111:8081/v1/jobs

# 停止job
# a50773a0af8feb5d57a16b6b7191d6b2：是jobid
curl -XPATCH  http://192.168.10.111:8081/jobs/a50773a0af8feb5d57a16b6b7191d6b2

# 查看包
curl -XGET http://192.168.10.111:8081/jars

# 上传包
curl -X POST -F "jarfile=@/root/.jenkins/workspace/mmhsy-dw-flink/target/mmhsy-dw-flink-1.0-SNAPSHOT.jar" http://192.168.10.111:8081/jars/upload

# 删除包
# be820cde-b111-4641-9542-abaef4ca239f_Flink-Demo-1.0-SNAPSHOT.jar：是上传jar时生成的id
curl -X DELETE -H "Content-Type:application/x-java-archive" http://192.168.10.111:8081/jars/be820cde-b111-4641-9542-abaef4ca239f_Flink-Demo-1.0-SNAPSHOT.jar
```



