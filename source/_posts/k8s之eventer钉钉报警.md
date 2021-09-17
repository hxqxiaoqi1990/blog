---
title: k8s之eventer钉钉报警
date: 2021-09-16 10:58:49
tags: [k8s]
categories: kubernetes
---

# 介绍

- 在Kubernetes中，事件分为两种，一种是Warning事件，表示产生这个事件的状态转换是在非预期的状态之间产生的；另外一种是Normal事件，表示期望到达的状态，和目前达到的状态是一致的。我们用一个Pod的生命周期进行举例，当创建一个Pod的时候，首先Pod会进入Pending的状态，等待镜像的拉取，当镜像录取完毕并通过健康检查的时候，Pod的状态就变为Running。此时会生成Normal的事件。而如果在运行中，由于OOM或者其他原因造成Pod宕掉，进入Failed的状态，而这种状态是非预期的，那么此时会在Kubernetes中产生Warning的事件。那么针对这种场景而言，如果我们能够通过监控事件的产生就可以非常及时的查看到一些容易被资源监控忽略的问题。

- 针对Kubernetes的事件监控场景，Kuernetes社区在Heapter中提供了简单的事件离线能力，后来随着Heapster的废弃，相关的能力也一起被归档了。为了弥补事件监控场景的缺失，阿里云容器服务发布并开源了kubernetes事件离线工具kube-eventer。支持离线kubernetes事件到钉钉机器人、SLS日志服务、Kafka开源消息队列、InfluxDB时序数据库等等。
- 官方仓库地址 ：https://github.com/AliyunContainerService/kube-eventer
- 支持通知方式：
  - 钉钉机器人
  - 微信
  - SLS日志服务
  - Kafka开源消息队列
  - InfluxDB时序数据库
  - elasticsearch
  - mongodb
  - .......

# 部署

**创建yaml文件**

```bash
vim eventer.yaml
```

**添加以下内容**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    name: kube-eventer
  name: kube-eventer
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kube-eventer
  template:
    metadata:
      labels:
        app: kube-eventer
      annotations:	
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      dnsPolicy: ClusterFirstWithHostNet
      serviceAccount: kube-eventer
      containers:
        - image: registry.aliyuncs.com/acs/kube-eventer-amd64:v1.2.0-484d9cd-aliyun
          name: kube-eventer
          command:
            - "/kube-eventer"
            - "--source=kubernetes:https://kubernetes.default"
            ## .e.g,dingtalk sink demo
            - --sink=dingtalk:https://oapi.dingtalk.com/robot/send?access_token=da26636c287511a04ea3e57fdfd860439b2644ae50d0dee18311ab9c1c5a669b&label=huang&level=Warning
          env:
          # If TZ is assigned, set the TZ value as the time zone
          - name: TZ
            value: "Asia/Shanghai" 
          volumeMounts:
            - name: localtime
              mountPath: /etc/localtime
              readOnly: true
            - name: zoneinfo
              mountPath: /usr/share/zoneinfo
              readOnly: true
          resources:
            requests:
              cpu: 100m
              memory: 100Mi
            limits:
              cpu: 500m
              memory: 250Mi
      volumes:
        - name: localtime
          hostPath:
            path: /etc/localtime
        - name: zoneinfo
          hostPath:
            path: /usr/share/zoneinfo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: kube-eventer
rules:
  - apiGroups:
      - ""
    resources:
      - configmaps
      - events
    verbs:
      - get
      - list
      - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kube-eventer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: kube-eventer
subjects:
  - kind: ServiceAccount
    name: kube-eventer
    namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube-eventer
  namespace: kube-system
```

**部署**

```bash
kubectl apply -f eventer.yaml
```

**字段说明**

- Webhook Sink支持根据事件的Kind、Reason、Level、Namespace进行过滤，支持通过泛化的逻辑将数据离线给自定义Webhook系统、钉钉、微信、贝洽（bear chat）、slack等等。那么如何使用泛化的Webhook来实现上述的逻辑呢？首先我们先来看下Webhook Sink的参数与定义。

```
# --sink=webhook:<WEBHOOK_URL>&level=<Normal or Warning, Warning default>
--sink=webhook:<WEBHOOK_URL>?level=Warning&namespaces=ns1,ns2&kinds=Node,Pod&method=POST&header=customHeaderKey=customHeaderValue 
```

- level -事件级别（可选。默认：警告。选项：警告和正常）
- namespaces -要过滤的名称空间（可选。默认值：所有名称空间，使用逗号分隔多个名称空间，Regexp模式支持）
- kinds -要过滤的种类（可选。默认：所有种类，使用逗号分隔多个种类。选项：Node，Pod等。）
- reason-进行过滤的原因（可选。默认值：空，支持Regexp模式）。您可以在查询中使用多原因字段。
- method -发送请求的方法（可选。默认：GET）
- header-请求中的标头（可选。默认值：空）。您可以在查询中使用多标题字段。
- custom_body_configmap-请求主体模板的configmap名称。您可以使用模板来自定义请求主体。（可选的。）
- custom_body_configmap_namespace-请求主体模板的configmap命名空间。（可选的。）
