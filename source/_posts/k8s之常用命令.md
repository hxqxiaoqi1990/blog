---
title: k8s之常用命令
tags: [k8s]
categories: kubernetes
date: 2019-09-18 15:06:00
---

获取节点相应服务的信息

``` bash
kubectl get pods
```


查看集群信息

``` bash
kubectl cluster-info
```

查看各组件信息

``` bash
kubectl get cs
```

查看pods所在的运行节点

``` bash
kubectl get pods -o wide
```

查看pods定义的详细信息

``` bash
kubectl get pods -o yaml
```

查看运行的pod的环境变量

``` bash
kubectl exec pod名 env
```

查看指定pod的日志

``` bash
kubectl logs -f pods/heapster-xxxxx
```

创建资源

``` bash
kubectl create -f 文件名.yaml
```

重建资源

``` bash
kubectl replace -f 文件名  [--force]
```

删除资源

``` bash
kubectl delete -f 文件名
kubectl delete pod pod名
kubectl delete rc rc名
kubectl delete service service名
kubectl delete pod --all
```

查看所有service

``` bash
kubectl get services kubernetes-dashboard -n kube-system 
```

查看所有发布

``` bash
kubectl get deployment kubernetes-dashboard -n kube-system 
```

查看所有pod

``` bash
kubectl get pods --all-namespaces 
```

查看所有pod的IP及节点

``` bash
kubectl get pods -o wide --all-namespaces 
kubectl get pods -n kube-system | grep dashboard
```

查看指定资源详细描述信息

``` bash
kubectl describe svc --namespace="kube-system"
```

指定类型查看

``` bash
kubectl describe pods/kubernetes-dashboard-349859023-g6q8c --namespace="kube-system" 
```

查看pod详细信息

``` bash
kubectl describe pod nginx-772ai 
```

动态伸缩

``` bash
kubectl scale rc nginx --replicas=5 
```

动态伸缩

``` bash
kubectl scale deployment redis-slave --replicas=5 
```

动态伸缩

``` bash
kubectl scale --replicas=2 -f redis-slave-deployment.yaml 
```

进入pod容器
注：/bin/bash如果不能进入，可以换成sh

``` bash
kubectl exec -it redis-master-1033017107-q47hh /bin/bash 
```


增加lable值 [key]=[value]

``` bash
kubectl label pod redis-master-1033017107-q47hh role=master 
```

删除lable值

``` bash
kubectl label pod redis-master-1033017107-q47hh role- 
```

修改lable值

``` bash
kubectl label pod redis-master-1033017107-q47hh role=backend --overwrite
```


指定pod在哪个节点

``` bash
kubectl label nodes node1 zone=north 
```

配置文件滚动升级

``` bash
kubectl rolling-update redis-master -f redis-master-controller-v2.yaml 
```

命令升级

``` bash
kubectl rolling-update redis-master --image=redis-master:2.0 
```

pod版本回滚

``` bash
kubectl rolling-update redis-master --image=redis-master:1.0 --rollback 
```

查看重启前日志

```bash
kubectl logs -p pod名 -n 名称空间
```

强制删除资源

```
kubectl delete pod [podname] --force --grace-period=0 -n [namespace]
```







