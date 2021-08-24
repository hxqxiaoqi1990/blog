---
title: k8s之HPA容器水平伸缩
tags: [k8s]
categories: kubernetes
date: 2020-11-11 11:00:00
---

# HPA简介

HPA（Horizontal Pod Autoscaler）是kubernetes（以下简称k8s）的一种资源对象，能够根据某些指标对在statefulSet、replicaController、replicaSet等集合中的pod数量进行动态伸缩，使运行在上面的服务对指标的变化有一定的自适应能力。

HPA目前支持四种类型的指标，分别是Resource、Object、External、Pods。其中在稳定版本autoscaling/v1中只支持对CPU指标的动态伸缩，在测试版本autoscaling/v2beta2中支持memory和自定义指标的动态伸缩，并以annotation的方式工作在autoscaling/v1版本中。

- **HPA动态伸缩的原理**
  HPA在k8s中也由一个controller控制，controller会间隔循环HPA，检查每个HPA中监控的指标是否触发伸缩条件，默认的间隔时间为15s。一旦触发伸缩条件，controller会向k8s发送请求，修改伸缩对象（statefulSet、replicaController、replicaSet）子对象scale中控制pod数量的字段。k8s响应请求，修改scale结构体，然后会刷新一次伸缩对象的pod数量。伸缩对象被修改后，自然会通过`list/watch`机制增加或减少pod数量，达到动态伸缩的目的。
- **HPA伸缩过程叙述**
  HPA的伸缩主要流程如下：

1. 判断当前pod数量是否在HPA设定的pod数量区间中，如果不在，过小返回最小值，过大返回最大值，结束伸缩。
2. 判断指标的类型，并向api server发送对应的请求，拿到设定的监控指标。一般来说指标会根据预先设定的指标从以下三个`aggregated APIs`中获取：`metrics.k8s.io`、`custom.metrics.k8s.io`、 `external.metrics.k8s.io`。其中`metrics.k8s.io`一般由k8s自带的metrics-server来提供，主要是cpu，memory使用率指标，另外两种需要第三方的adapter来提供。`custom.metrics.k8s.io`提供自定义指标数据，一般跟k8s集群有关，比如跟特定的pod相关。`external.metrics.k8s.io`同样提供自定义指标数据，但一般跟k8s集群无关。许多知名的第三方监控平台提供了adapter实现了上述api（如prometheus），可以将监控和adapter一同部署在k8s集群中提供服务，甚至能够替换原来的metrics-server来提供上述三类api指标，达到深度定制监控数据的目的。
3. 根据获得的指标，应用相应的算法算出一个伸缩系数，并乘以目前pod数量获得期望pod数量。系数是指标的期望值与目前值的比值，如果大于1表示扩容，小于1表示缩容。指标数值有平均值（AverageValue）、平均使用率（Utilization）、裸值（Value）三种类型，每种类型的数值都有对应的算法。以下几点值得注意：如果系数有小数点，统一进一；系数如果未达到某个容忍值，HPA认为变化太小，会忽略这次变化，容忍值默认为0.1。
   HPA扩容算法是一个非常保守的算法。如果出现获取不到指标的情况，扩容时算最小值，缩容时算最大值；如果需要计算平均值，出现pod没准备好的情况，平均数的分母不计入该pod。
   一个HPA支持多个指标的监控，HPA会循环获取所有的指标，并计算期望的pod数量，并从期望结果中获得最大的pod数量作为最终的伸缩的pod数量。一个伸缩对象在k8s中允许对应多个HPA，但是只是k8s不会报错而已，事实上HPA彼此不知道自己监控的是同一个伸缩对象，在这个伸缩对象中的pod会被多个HPA无意义地来回修改pod数量，给系统增加消耗，如果想要指定多个监控指标，可以如上述所说，在一个HPA中添加多个监控指标。
4. 检查最终的pod数量是否在HPA设定的pod数量范围的区间，如果超过最大值或不足最小值都会修改为最大值或最小值。然后向k8s发出请求，修改伸缩对象的子对象scale的pod数量，结束一个HPA的检查，获取下一个HPA，完成一个伸缩流程。



- **HPA的应用场景**
  HPA的特性结合第三方的监控应用，使得部署在HPA伸缩对象（statefulSet、replicaController、replicaSet）上的服务有了非常灵活的自适应能力，能够在一定限度内复制多个副本来应对某个指标的急剧飙升，也可以在某个指标较小的情况下删除副本来让出计算资源给其他更需要资源的应用使用，维持整个系统的稳定。非常适合于一些流量波动大，机器资源吃紧，服务数量多的业务场景，如：电商服务、抢票服务、金融服务等。
- **k8s-prometheus-adapter**
  前文说到，许多监控系统通过adapter实现了接口，给HPA提供指标数据。在这里我们具体介绍一下prometheus监控系统的adapter。
  prometheus是一个知名开源监控系统，具有数据维度多，存储高效，使用便捷等特点。用户可以通过丰富的表达式和内置函数，定制自己所需要的监控数据。
  prometheus-adapter在prometheus和api-server中起到了适配者的作用。prometheus-adapter接受从HPA中发来，通过apiserver aggregator中转的指标查询请求，然后根据内容发送相应的请求给prometheus拿到指标数据，经过处理后返回给HPA使用。prometheus可以同时实现`metrics.k8s.io`、`custom.metrics.k8s.io`、 `external.metrics.k8s.io`三种api接口，代替k8s自己的matrics-server，提供指标数据服务。
  prometheus-adapter部署能否成功的关键在于配置文件是否正确。配置文件中可以设定需要的指标以及对指标的处理方式，以下是一份简单的配置文件，加上注释以稍做解释。

# 配置结构

```yaml
apiVersion: autoscaling/v2beta2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  # HPA的伸缩对象描述，HPA会动态修改该对象的pod数量
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  # HPA的最小pod数量和最大pod数量
  minReplicas: 1
  maxReplicas: 10
  # 监控的指标数组，支持多种类型的指标共存
  metrics:
  # Object类型的指标
  - type: Object
    object:
      metric:
        # 指标名称
        name: requests-per-second
      # 监控指标的对象描述，指标数据来源于该对象
      describedObject:
        apiVersion: networking.k8s.io/v1beta1
        kind: Ingress
        name: main-route
      # Value类型的目标值，Object类型的指标只支持Value和AverageValue类型的目标值
      target:
        type: Value
        value: 10k
  # Resource类型的指标
  - type: Resource
    resource:
      name: cpu
      # Utilization类型的目标值，Resource类型的指标只支持Utilization和AverageValue类型的目标值
      target:
        type: Utilization
        averageUtilization: 50
  # Pods类型的指标
  - type: Pods
    pods:
      metric:
        name: packets-per-second
      # AverageValue类型的目标值，Pods指标类型下只支持AverageValue类型的目标值
      target:
        type: AverageValue
        averageValue: 1k
  # External类型的指标
  - type: External
    external:
      metric:
        name: queue_messages_ready
        # 该字段与第三方的指标标签相关联，（此处官方文档有问题，正确的写法如下）
        selector:
          matchLabels:
            env: "stage"
            app: "myapp"
      # External指标类型下只支持Value和AverageValue类型的目标值
      target:
        type: AverageValue
        averageValue: 30
```

`autoscaling/v1`版本将metrics字段放在了annotation中进行处理。

target共有3种类型：Utilization、Value、AverageValue。Utilization表示平均使用率；Value表示裸值；AverageValue表示平均值。

metrics中的type字段有四种类型的值：Object、Pods、Resource、External。

Resource指的是当前伸缩对象下的pod的cpu和memory指标，只支持Utilization和AverageValue类型的目标值。

Object指的是指定k8s内部对象的指标，数据需要第三方adapter提供，只支持Value和AverageValue类型的目标值。

Pods指的是伸缩对象（statefulSet、replicaController、replicaSet）底下的Pods的指标，数据需要第三方的adapter提供，并且只允许

AverageValue类型的目标值。

External指的是k8s外部的指标，数据同样需要第三方的adapter提供，只支持Value和AverageValue类型的目标值。

# 操作命令

```bash
# 查看指定名称空间下，所有pod的cpu和内存使用量
kubectl top pods -n mamahao

# 查看指定名称空间下，的 hpa 配置项
kubectl get hpa -n mamahao

# 查看指定名称空间下，指定pod的详细内容
kubectl describe hpa web-test -n mamahao
```

