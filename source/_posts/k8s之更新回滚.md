---
title: k8s之更新回滚
tags: [k8s]
categories: kubernetes
date: 2020-05-24 14:06:00
---

# 介绍

记录k8s的更新操作

更新的三种方式：

1. kubectl edit 修改配置
2. kubectl set image 修改镜像版本（只介绍该方法）
3. kubectl patch 修改配置

# 更新回滚

以一个简单的deployment为例来进行说明：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: nginx
  strategy:  # 更新配置声明
    rollingUpdate:
      maxSurge: 25%  # 滚动更新时最多可以多启动多少个pod，更新时 最大的pod数是 replicas+ maxSurge = 5+1 =6，最大的个数是6，可以设置整数或百分比。
      maxUnavailable: 25%  # 滚动更新时最大可以删除多少个pod，最小pod数是 replicas - maxUnavailable = 5-0 = 5,最小pod数是5，所以只能先启动一个pod，再删除一个pod，可以设置整数或百分比。
    type: RollingUpdate  # 更新类型：滚动跟新
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.12
        ports:
        - containerPort: 80
```

创建deployment

```shell
# --record：记录版本，没有该参数无法回滚
kubectl apply -f nginx-deployment.yaml --record
```

更新与回滚

只有spec.template.spec.containers.image或者spec.template.metadata.labels发生变化时才会出发更新rolout操作

```bash
# 更新镜像版本
kubectl set image deployment nginx-deployment nginx=nginx:1.14 --record

# 查看已有版本，数字最大的为当前使用镜像
kubectl rollout history deployment nginx-deployment

# 回滚上一个版本 
kubectl rollout undo deployment nginx-deployment

# 回滚到指定版本
kubectl rollout undo deployment nginx-deployment --to-revision=2

#查看升级状态
kubectl rollout status deploy nginx 

#升级暂停
kubectl rollout pause deployment nginx 

#恢复升级
kubectl rollout resume deployment nginx 
```



