---
title: k8s之就绪检查和存活检查
tags: [k8s]
categories: kubernetes
date: 2020-08-01 15:06:00
---

# 介绍

**就绪探针**

就绪探针旨在让Kubernetes知道你的应用**是否准备好为请求提供服务**。Kubernetes只有在就绪探针通过才会把流量转发到Pod。如果就绪探针检测失败，Kubernetes将停止向该容器发送流量，直到它通过。

**存活探针**

Liveness探测器是让Kubernetes知道你的**应用是否活着**。如果你的应用还活着，那么Kubernetes就让它继续存在。如果你的应用程序已经死了，Kubernetes将移除Pod并重新启动一个来替换它。

# 配置

**存活检查 HTTP**

```yaml
      containers:
        - name: xxx
          image: xxx

          # 存活检查
          livenessProbe:
            httpGet:
              scheme: HTTP             # 协议
              path: /actuator/health   # 路径
              port: 8080               # 端口
            initialDelaySeconds: 30    # 延迟探测时间(秒) 【 在k8s第一次探测前等待秒 】
            periodSeconds: 10          # 执行探测频率(秒) 【 每隔秒执行一次 】
            timeoutSeconds: 1          # 超时时间
            successThreshold: 1        # 健康阀值 
            failureThreshold: 3        # 不健康阀值 
```

**就绪检查 HTTP**

```bash
      containers:
        - name: xxx
          image: xxx

          # 就绪检查
          readinessProbe:
            httpGet:
              scheme: HTTP             # 协议
              path: /actuator/health   # 路径
              port: 8080               # 端口
            initialDelaySeconds: 30    # 延迟探测时间(秒)【 在k8s第一次探测前等待秒 】
            periodSeconds: 10          # 执行探测频率(秒) 【 每隔秒执行一次 】
            timeoutSeconds: 1          # 超时时间
            successThreshold: 1        # 健康阀值 
            failureThreshold: 3        # 不健康阀值 
```

