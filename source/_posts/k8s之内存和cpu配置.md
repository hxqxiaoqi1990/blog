---
title: k8s之内存和cpu配置
tags: [k8s]
categories: kubernetes
date: 2020-05-25 15:06:00
---

# 介绍

资源请求（requests）：即允许运行的最小资源，pod调度规则根据该值判断。

资源限制（limits）：即允许运行的最大资源，limits不会影响pod调度规则，因此limits总和可以超过服务器提供的总资源数，服务运行超过该值会被kill重启。

另外：k8s仅会确保pod能够获得他们请求的cpu时间额度，他们能否获得额外的cpu时间，则取决于其他正在运行的作业对cpu资源的占用情况。例如，对于总数为1000m的cpu来说，容器a请求使用200m，容器b请求使用500m，在不超出它们各自的最大限额的前提下，余下的300m在双方都需要时会以2:5的方式进行配置。（限制cpu是限制其运行占用cpu时间）

# 名称空间资源配置

设置名称空间`namespaces`默认资源

名称空间设置是应用到名称空间下所有未定义资源配置的pod中，如果pod单独定义资源配置，则优先使用pod定义的配置。

```bash
# 查看名称空间
kubectl get ns

# 创建名称空间 default
kubectl create namespace default

# 查看名称空间资源配置信息
kubectl describe namespaces default
```

创建 `limit-default.yaml`：编辑以下内容

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limit-range
spec:
  limits:
  - default:
      cpu: 0.25
      memory: 2048Mi
    defaultRequest:
      cpu: 0.1
      memory: 128Mi
    type: Container
```

更新配置

```bash
# 应用配置
kubectl create -f limit-default.yaml --namespace=default

# 如果配置还有更改，使用以下命令更新
kubectl replace -f limit-default.yaml --namespace=default

# 查看名称空间限制
kubectl describe namespaces default 
```

# pod资源配置

pod资源配置有`deployment` 控制器定义的

```bash
# 查看 deployment 控制器
kubectl get deployment

# 修改 deployment 控制器
kubectl edit deployment nginx-deployment
```

以下为修改内容

```yaml
    spec:
      containers:
      - image: nginx:1.16
        imagePullPolicy: IfNotPresent
        lifecycle: {}
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
        resources:
          limits:
            cpu: 500m
            memory: 500Mi
          requests:
            cpu: 100m
            memory: 128Mi
```

查看pod修改后的配置

```bash
kubectl get pod nginx-deployment-79788f59b6-nfvst --output=yaml --namespace=default
```

查看pod所在node节点资源配置

```bash
kubectl  describe  node node01
```

# 资源配置详解

配置示例

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mylimits
spec:
  limits:
  - max:
      cpu: "4"
      memory: 2Gi
    min:
      cpu: 200m
      memory: 6Mi
    maxLimitRequestRatio:
      cpu: 3
      memory: 2
    type: Pod
  - default:
      cpu: 300m
      memory: 200Mi
    defaultRequest:
      cpu: 200m
      memory: 100Mi
    max:
      cpu: "2"
      memory: 1Gi
    min:
      cpu: 100m
      memory: 3Mi
    maxLimitRequestRatio:
      cpu: 5
      memory: 4
    type: Container
```

`pod`部分：

1. `max`表示`pod`中所有容器资源的`Limit`值和的上限，也就是整个`pod`资源的最大`Limit`，如果`pod`定义中的`Limit`值大于`LimitRange`中的值，则`pod`无法成功创建。
2. `min`表示`pod`中所有容器资源请求总和的下限，也就是所有容器`request`的资源总和不能小于`min`中的值，否则`pod`无法成功创建。
3. `maxLimitRequestRatio`表示`pod`中所有容器资源请求的`Limit`值和`request`值比值的上限，例如该`pod`中`cpu`的`Limit`值为3，而`request`为0.5，此时比值为6，创建`pod`将会失败。

`container`部分

1. 在`container`的部分，`max`、`min`和`maxLimitRequestRatio`的含义和`pod`中的类似，只不过是针对单个的容器而言。下面说明几个情况：

> 如果`container`设置了`max`， `pod`中的容器必须设置`limit`，如果未设置，则使用`defaultlimt`的值，如果`defaultlimit`也没有设置，则无法成功创建
>
> 如果设置了`container`的`min`，创建容器的时候必须设置`request`的值，如果没有设置，则使用`defaultrequest`，如果没有`defaultrequest`，则默认等于容器的`limit`值，如果`limit`也没有，启动就会报错

1. `defaultrequest`和`defaultlimit`则是默认值，注意：`pod`级别没有这两项设置