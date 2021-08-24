---
title: k8s之pod控制器
tags: [k8s]
categories: kubernetes
date: 2020-05-26 15:06:00
---

# 介绍

Pod控制器是用于实现管理pod的中间层，确保pod资源符合预期的状态，pod的资源出现故障时，会尝试 进行重启，当根据重启策略无效，则会重新新建pod的资源。

pod控制器有多种类型：

1. **ReplicaSet**：代用户创建指定数量的pod副本数量，确保pod副本数量符合预期状态，并且支持滚动式自动扩容和缩容功能。主要用于配合`Deployment`控制器使用。
2. **Deployment**：工作在ReplicaSet之上，用于管理无状态应用，目前来说最好的控制器。支持滚动更新和回滚功能，还提供声明式配置。
3. **DaemonSet**：用于确保集群中的每一个节点只运行特定的pod副本，通常用于实现系统级后台任务。比如ELK服务
4. **StatefulSet**：管理有状态应用

# Deployment

功能：

- （1）使用Deployment来创建ReplicaSet。ReplicaSet在后台创建pod。检查启动状态，看它是成功还是失败。
- （2）然后，通过更新Deployment的PodTemplateSpec字段来声明Pod的新状态。这会创建一个新的ReplicaSet，Deployment会按照控制的速率将pod从旧的ReplicaSet移动到新的ReplicaSet中。
- （3）如果当前状态不稳定，回滚到之前的Deployment revision。每次回滚都会更新Deployment的revision。
- （4）扩容Deployment以满足更高的负载。
- （5）暂停Deployment来应用PodTemplateSpec的多个修复，然后恢复上线。
- （6）根据Deployment 的状态判断上线是否hang住了。
- （7）清除旧的不必要的 ReplicaSet。

```yaml
apiVersion: apps/v1　　#api版本定义
kind: Deployment　　#定义资源类型为Deployment
metadata:　　#元数据定义
    name: myapp #Deployment名称
    namespace: default #应用的名称空间
spec:　　#Deployment的规格定义
    replicas: 2　　#定义副本数量为2个
    selector:　　　　#标签选择器，定义匹配pod的标签
        matchLabels:
            app: myapp
    template:　　#pod的模板定义
        metadata:　　#pod的元数据定义
            labels: 　　#定义pod的标签，需要和上面定义的标签一致，也可以多出其他标签
                app: myapp
        spec:　　#pod的规格定义
            containers:　　#容器定义
            - name: myapp-container　　#容器名称
              image: ikubernetes/myapp:v1　　#容器镜像
              ports:　　#暴露端口
              - name: http
                containerPort: 80
```

# DaemonSet

DaemonSet 确保全部（或者一些）Node 上运行一个 Pod 的副本。当有 Node 加入集群时，也会为他们新增一个 Pod 。当有 Node 从集群移除时，这些 Pod 也会被回收。删除 DaemonSet 将会删除它创建的所有 Pod。

```yaml
kind: DaemonSet
apiVersion: apps/v1
metadata: 
  labels:
    app: node-exporter
  name: node-exporter
  namespace: ns-monitor
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: node-exporter
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:v0.16.0
          ports:
            - containerPort: 9100
              protocol: TCP
              name:	http
      hostNetwork: true
      hostPID: true
```

# StatefulSet

一个完整的StatefulSet控制器由一个Headless Service、一个StatefulSet和一个volumeClaimTemplate组成。如下资源清单中的定义：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # 集群设置None就是headless service 无头服务
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # 必须设置项
  serviceName: "nginx"  #声明它属于哪个Headless Service.
  replicas: 3 # 默认1
  template:
    metadata:
      labels:
        app: nginx # 必须设置项
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:   #可看作pvc的模板
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "gluster-heketi"  #存储类名，改为集群中已存在的
      resources:
        requests:
          storage: 1Gi
```

- Headless Service：名为nginx，用来定义Pod网络标识( DNS domain)。
- StatefulSet：定义具体应用，名为Nginx，有三个Pod副本，并为每个Pod定义了一个域名。
- volumeClaimTemplates： 存储卷申请模板，创建PVC，指定pvc名称大小，将自动创建pvc，且pvc必须由存储类供应。

**为什么需要 headless service 无头服务？**
在用Deployment时，每一个Pod名称是没有顺序的，是随机字符串，因此是Pod名称是无序的，但是在statefulset中要求必须是有序 ，每一个pod不能被随意取代，pod重建后pod名称还是一样的。而pod IP是变化的，所以是以Pod名称来识别。pod名称是pod唯一性的标识符，必须持久稳定有效。这时候要用到无头服务，它可以给每个Pod一个唯一的名称 。

**为什么需要volumeClaimTemplate？**
对于有状态的副本集都会用到持久存储，对于分布式系统来讲，它的最大特点是数据是不一样的，所以各个节点不能使用同一存储卷，每个节点有自已的专用存储，但是如果在Deployment中的Pod template里定义的存储卷，是所有副本集共用一个存储卷，数据是相同的，因为是基于模板来的 ，而statefulset中每个Pod都要自已的专有存储卷，所以statefulset的存储卷就不能再用Pod模板来创建了，于是statefulSet使用volumeClaimTemplate，称为卷申请模板，它会为每个Pod生成不同的pvc，并绑定pv， 从而实现各pod有专用存储。这就是为什么要用volumeClaimTemplate的原因。