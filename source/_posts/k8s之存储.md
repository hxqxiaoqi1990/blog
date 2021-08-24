---
title: k8s之存储
tags: [k8s]
categories: kubernetes
date: 2020-06-04 09:06:00
---

# 介绍

为了保证数据的持久性，必须保证数据在外部存储在docker容器中，为了实现数据的持久性存储，在宿主机和容器内做映射，可以保证在容器的生命周期结束，数据依旧可以实现持久性存储。但是在k8s中，由于*pod分布在各个不同的节点之上，并不能实现不同节点之间持久性数据的共享，并且，在节点故障时，可能会导致数据的永久性丢失。为此，k8s就引入了外部存储卷的功能。

k8s的存储卷类型：

1. `emptyDir`（临时目录）:Pod删除，数据也会被清除，这种存储成为emptyDir，用于数据的临时存储。
2. `hostPath` (宿主机目录映射):
3. `本地的SAN` (iSCSI,FC)、NAS(nfs,cifs,http)存储
4. `分布式存储`（glusterfs，rbd，cephfs）
5. `云存储`（EBS，Azure Disk）

<font color='32CD32'>k8s整体存储概念为</font>：实际存储空间（nfs，hostpath...）---> pv（抽象的存储卷，从实际的存储空间划分）---> pvc（存储卷创建申请，从pv中划分定义）---> pod（在创建pod中引用pvc进行挂在）

参考链接：https://www.cnblogs.com/linuxk/p/9760363.html

# emptyDir存储卷演示

一个emptyDir 第一次创建是在一个pod被指定到具体node的时候，并且会一直存在在pod的生命周期当中，正如它的名字一样，它初始化是一个空的目录，pod中的容器都可以读写这个目录，这个目录可以被挂在到各个容器相同或者不相同的的路径下。当一个pod因为任何原因被移除的时候，这些数据会被永久删除。注意：一个容器崩溃了不会导致数据的丢失，因为容器的崩溃并不移除pod。

```bash
[root@k8s-master ~]# kubectl explain pods.spec.volumes.emptyDir  #查看emptyDir存储定义
[root@k8s-master ~]# kubectl explain pods.spec.containers.volumeMounts  #查看容器挂载方式
[root@k8s-master ~]# cd mainfests && mkdir volumes && cd volumes
[root@k8s-master volumes]# cp ../pod-demo.yaml ./
[root@k8s-master volumes]# mv pod-demo.yaml pod-vol-demo.yaml

[root@k8s-master volumes]# vim pod-vol-demo.yaml   #创建emptyDir的清单
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: default
  labels:
    app: myapp
    tier: frontend
  annotations:
    magedu.com/create-by:"cluster admin"
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 80
    volumeMounts:    #在容器内定义挂载存储名称和挂载路径
    - name: html
      mountPath: /usr/share/nginx/html/
  - name: busybox
    image: busybox:latest
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: html
      mountPath: /data/    #在容器内定义挂载存储名称和挂载路径
    command: ['/bin/sh','-c','while true;do echo $(date) >> /data/index.html;sleep 2;done']
  volumes:  #定义存储卷
  - name: html    #定义存储卷名称  
    emptyDir: {}  #定义存储卷类型
    
[root@k8s-master volumes]# kubectl apply -f pod-vol-demo.yaml 
pod/pod-vol-demo created 

[root@k8s-master volumes]# kubectl get pods
NAME                                 READY     STATUS    RESTARTS   AGE
pod-vol-demo                         2/2       Running   0          27s

[root@k8s-master volumes]# kubectl get pods -o wide
NAME                      READY     STATUS    RESTARTS   AGE       IP            NODE
......
pod-vol-demo              2/2       Running   0          16s       10.244.2.34   k8s-node02
......

# 在上面，我们定义了2个容器，其中一个容器是输入日期到index.html中，然后验证访问nginx的html是否可以获取日期。以验证两个容器之间挂载的emptyDir实现共享。如下访问验证:
[root@k8s-master volumes]# curl 10.244.2.34  #访问验证
Tue Oct 9 03:56:53 UTC 2018
Tue Oct 9 03:56:55 UTC 2018
Tue Oct 9 03:56:57 UTC 2018
Tue Oct 9 03:56:59 UTC 2018
Tue Oct 9 03:57:01 UTC 2018
Tue Oct 9 03:57:03 UTC 2018
Tue Oct 9 03:57:05 UTC 2018
Tue Oct 9 03:57:07 UTC 2018
Tue Oct 9 03:57:09 UTC 2018
Tue Oct 9 03:57:11 UTC 2018
Tue Oct 9 03:57:13 UTC 2018
Tue Oct 9 03:57:15 UTC 2018
```

# hostPath存储卷演示

hostPath宿主机路径，就是把pod所在的宿主机之上的脱离pod中的容器名称空间的之外的宿主机的文件系统的某一目录和pod建立关联关系，在pod删除时，存储数据不会丢失。

```bash
[root@k8s-master volumes]# vim pod-hostpath-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-hostpath
  namespace: default
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
    - name: html
      hostPath:
        path: /data/pod/volume1
        type: DirectoryOrCreate
# DirectoryOrCreate  宿主机上不存在创建此目录  
# Directory 必须存在挂载目录  
# FileOrCreate 宿主机上不存在挂载文件就创建  
# File 必须存在文件  

[root@k8s-node01 ~]# mkdir -p /data/pod/volume1
[root@k8s-node01 ~]# vim /data/pod/volume1/index.html
node01.magedu.com
[root@k8s-node02 ~]# mkdir -p /data/pod/volume1
[root@k8s-node02 ~]# vim /data/pod/volume1/index.html
node02.magedu.com
[root@k8s-master volumes]# kubectl apply -f pod-hostpath-vol.yaml 
pod/pod-vol-hostpath created

[root@k8s-master volumes]# kubectl get pods -o wide
NAME                                 READY     STATUS    RESTARTS   AGE       IP            NODE
......
pod-vol-hostpath                     1/1       Running   0          37s       10.244.2.35   k8s-node02
......
[root@k8s-master volumes]# curl 10.244.2.35
node02.magedu.com
[root@k8s-master volumes]# kubectl delete -f pod-hostpath-vol.yaml  #删除pod，再重建，验证是否依旧可以访问原来的内容
[root@k8s-master volumes]# kubectl apply -f pod-hostpath-vol.yaml 
pod/pod-vol-hostpath created
[root@k8s-master volumes]# curl  10.244.2.37 
node02.magedu.com
```

# nfs共享存储卷演示

nfs使的我们可以挂在已经存在的共享到的我们的Pod中，和emptyDir不同的是，emptyDir会被删除当我们的Pod被删除的时候，但是nfs不会被删除，仅仅是解除挂在状态而已，这就意味着NFS能够允许我们提前对数据进行处理，而且这些数据可以在Pod之间相互传递.并且，nfs可以同时被多个pod挂在并进行读写。

```bash
（1）在stor01节点上安装nfs，并配置nfs服务
[root@stor01 ~]# yum install -y nfs-utils  ==》192.168.56.14
[root@stor01 ~]# mkdir /data/volumes -pv
[root@stor01 ~]# vim /etc/exports
/data/volumes 192.168.56.0/24(rw,no_root_squash)
[root@stor01 ~]# systemctl start nfs
[root@stor01 ~]# showmount -e
Export list for stor01:
/data/volumes 192.168.56.0/24

（2）在node01和node02节点上安装nfs-utils，并测试挂载
[root@k8s-node01 ~]# yum install -y nfs-utils
[root@k8s-node02 ~]# yum install -y nfs-utils
[root@k8s-node02 ~]# mount -t nfs stor01:/data/volumes /mnt
[root@k8s-node02 ~]# mount
......
stor01:/data/volumes on /mnt type nfs4 (rw,relatime,vers=4.1,rsize=131072,wsize=131072,namlen=255,hard,proto=tcp,port=0,timeo=600,retrans=2,sec=sys,clientaddr=192.168.56.13,local_lock=none,addr=192.168.56.14)
[root@k8s-node02 ~]# umount /mnt/

（3）创建nfs存储卷的使用清单
[root@k8s-master volumes]# cp pod-hostpath-vol.yaml pod-nfs-vol.yaml
[root@k8s-master volumes]# vim pod-nfs-vol.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-nfs
  namespace: default
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
    - name: html
      nfs:
        path: /data/volumes
        server: stor01
[root@k8s-master volumes]# kubectl apply -f pod-nfs-vol.yaml 
pod/pod-vol-nfs created
[root@k8s-master volumes]# kubectl get pods -o wide
NAME                     READY     STATUS    RESTARTS   AGE       IP            NODE
pod-vol-nfs              1/1       Running   0          21s       10.244.2.38   k8s-node02

（3）在nfs服务器上创建index.html
[root@stor01 ~]# cd /data/volumes
[root@stor01 volumes ~]# vim index.html
<h1> nfs stor01</h1>
[root@k8s-master volumes]# curl 10.244.2.38
<h1> nfs stor01</h1>
[root@k8s-master volumes]# kubectl delete -f pod-nfs-vol.yaml   #删除nfs相关pod，再重新创建，可以得到数据的持久化存储
pod "pod-vol-nfs" deleted
[root@k8s-master volumes]# kubectl apply -f pod-nfs-vol.yaml 
```

# NFS使用PV和PVC（静态创建pv）

## 1、配置nfs存储

```bash
[root@stor01 volumes]# mkdir v{1,2,3,4,5}
[root@stor01 volumes]# vim /etc/exports
/data/volumes/v1 192.168.56.0/24(rw,no_root_squash)
/data/volumes/v2 192.168.56.0/24(rw,no_root_squash)
/data/volumes/v3 192.168.56.0/24(rw,no_root_squash)
/data/volumes/v4 192.168.56.0/24(rw,no_root_squash)
/data/volumes/v5 192.168.56.0/24(rw,no_root_squash)
[root@stor01 volumes]# exportfs -arv
exporting 192.168.56.0/24:/data/volumes/v5
exporting 192.168.56.0/24:/data/volumes/v4
exporting 192.168.56.0/24:/data/volumes/v3
exporting 192.168.56.0/24:/data/volumes/v2
exporting 192.168.56.0/24:/data/volumes/v1
[root@stor01 volumes]# showmount -e
Export list for stor01:
/data/volumes/v5 192.168.56.0/24
/data/volumes/v4 192.168.56.0/24
/data/volumes/v3 192.168.56.0/24
/data/volumes/v2 192.168.56.0/24
/data/volumes/v1 192.168.56.0/24
```

## 2、定义PV

这里定义了pvc的访问模式为多路读写，该访问模式必须在前面pv定义的访问模式之中。定义PVC申请的大小为2Gi，此时PVC会自动去匹配多路读写且大小为2Gi的PV，匹配成功获取PVC的状态即为Bound

```bash
[root@k8s-master volumes]# kubectl explain pv
[root@k8s-master volumes]# kubectl explain pv.spec.nfs
[root@k8s-master volumes]# vim pv-demo.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
  labels:
    name: pv001
spec:
  nfs:
    path: /data/volumes/v1
    server: stor01
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
    #accessModes（定义访问模型，有以下三种访问模型，以列表的方式存在，也就是说可以定义多个访问模式）
    #ReadWriteOnce（RWO）  单节点读写
	#ReadOnlyMany（ROX）  多节点只读
	#ReadWriteMany（RWX）  多节点读写
  capacity:
    storage: 1Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv002
  labels:
    name: pv002
spec:
  nfs:
    path: /data/volumes/v2
    server: stor01
  accessModes: ["ReadWriteOnce"]
  capacity:
    storage: 2Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv003
  labels:
    name: pv003
spec:
  nfs:
    path: /data/volumes/v3
    server: stor01
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 2Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv004
  labels:
    name: pv004
spec:
  nfs:
    path: /data/volumes/v4
    server: stor01
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 4Gi
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv005
  labels:
    name: pv005
spec:
  nfs:
    path: /data/volumes/v5
    server: stor01
  accessModes: ["ReadWriteMany","ReadWriteOnce"]
  capacity:
    storage: 5Gi
[root@k8s-master volumes]# kubectl apply -f pv-demo.yaml 
persistentvolume/pv001 created
persistentvolume/pv002 created
persistentvolume/pv003 created
persistentvolume/pv004 created
persistentvolume/pv005 created
[root@k8s-master volumes]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
pv001     1Gi        RWO,RWX        Retain           Available                                      7s
pv002     2Gi        RWO            Retain           Available                                      7s
pv003     2Gi        RWO,RWX        Retain           Available                                      7s
pv004     4Gi        RWO,RWX        Retain           Available                                      7s
pv005     5Gi        RWO,RWX        Retain           Available                                      7s
```

## 3、定义PVC

这里定义了pvc的访问模式为多路读写，该访问模式必须在前面pv定义的访问模式之中。定义PVC申请的大小为2Gi，此时PVC会自动去匹配多路读写且大小为2Gi的PV，匹配成功获取PVC的状态即为Bound

```bash
[root@k8s-master volumes ~]# vim pod-vol-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc
  namespace: default
spec:
  accessModes: ["ReadWriteMany"]
  resources:
    requests:
      storage: 2Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-vol-pvc
  namespace: default
spec:
  containers:
  - name: myapp
    image: ikubernetes/myapp:v1
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
    - name: html
      persistentVolumeClaim:
        claimName: mypvc
[root@k8s-master volumes]# kubectl apply -f pod-vol-pvc.yaml 
persistentvolumeclaim/mypvc created
pod/pod-vol-pvc created
[root@k8s-master volumes]# kubectl get pv
NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM           STORAGECLASS   REASON    AGE
pv001     1Gi        RWO,RWX        Retain           Available                                            19m
pv002     2Gi        RWO            Retain           Available                                            19m
pv003     2Gi        RWO,RWX        Retain           Bound       default/mypvc                            19m
pv004     4Gi        RWO,RWX        Retain           Available                                            19m
pv005     5Gi        RWO,RWX        Retain           Available                                            19m
[root@k8s-master volumes]# kubectl get pvc
NAME      STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mypvc     Bound     pv003     2Gi        RWO,RWX                       22s
```

## 4、测试访问

在存储服务器上创建index.html，并写入数据，通过访问Pod进行查看，可以获取到相应的页面。

```bash
[root@stor01 volumes]# cd v3/
[root@stor01 v3]# echo "welcome to use pv3" > index.html
[root@k8s-master volumes]# kubectl get pods -o wide
pod-vol-pvc             1/1       Running   0          3m        10.244.2.39   k8s-node02
[root@k8s-master volumes]# curl  10.244.2.39
welcome to use pv3
```

# StorageClass（动态创建pv）

在pv和pvc使用过程中存在的问题，在pvc申请存储空间时，未必就有现成的pv符合pvc申请的需求，上面nfs在做pvc可以成功的因素是因为我们做了指定的需求处理。那么当PVC申请的存储空间不一定有满足PVC要求的PV事，又该如何处理呢？？？为此，Kubernetes为管理员提供了描述存储"class（类）"的方法（StorageClass）。举个例子，在存储系统中划分一个1TB的存储空间提供给Kubernetes使用，当用户需要一个10G的PVC时，会立即通过restful发送请求，从而让存储空间创建一个10G的image，之后在我们的集群中定义成10G的PV供给给当前的PVC作为挂载使用。在此之前我们的存储系统必须支持restful接口，比如ceph分布式存储，而glusterfs则需要借助第三方接口完成这样的请求。

StorageClass 中包含 provisioner、parameters 和 reclaimPolicy 字段，当 class 需要动态分配 PersistentVolume 时会使用到。由于StorageClass需要一个独立的存储系统。

 之前我们部署了PV 和 PVC 的使用方法，但是前面的 PV 都是静态的，什么意思？就是我要使用的一个 PVC 的话就必须手动去创建一个 PV，我们也说过这种方式在很大程度上并不能满足我们的需求，比如我们有一个应用需要对存储的并发度要求比较高，而另外一个应用对读写速度又要求比较高，特别是对于 StatefulSet 类型的应用简单的来使用静态的 PV 就很不合适了，这种情况下我们就需要用到动态 PV，也就是我们今天要讲解的 StorageClass。

## 创建

要使用 StorageClass，我们就得安装对应的自动配置程序，比如我们这里存储后端使用的是 nfs，那么我们就需要使用到一个 nfs-client 的自动配置程序，我们也叫它 Provisioner，这个程序使用我们已经配置好的 nfs 服务器，来自动创建持久卷，也就是自动帮我们创建 PV。

- 自动创建的 PV 以`${namespace}-${pvcName}-${pvName}`这样的命名格式创建在 NFS 服务器上的共享数据目录中

**回收策略**

通过persistentVolumeReclaimPolicy字段设置，默认是Delete

◎ `Retain` 保留：保留数据，需要手工处理，解绑PVC后，PV保留，状态为`Available` 

◎ `Recycle` 回收空间：简单清除文件的操作（例如执行rm -rf /thevolume/* 命令），当这个 PV 被回收后会会清空实际存储数据

◎ `Delete` 删除：与PV相连的后端存储完成Volume的删除操作，而当这个 PV 被回收后会以`archieved-${namespace}-${pvcName}-${pvName}`这样的命名格式存在 NFS 服务器上。

**第一步**：配置 Deployment，将里面的对应的参数替换成我们自己的 nfs 配置（nfs-client.yaml），该pod控制自动创建pv。

```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app: nfs-client-provisioner
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.40.100
            - name: NFS_PATH
              value: /media/pv01
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.40.100
            path: /media/pv01
```

**第二步**：将环境变量 NFS_SERVER 和 NFS_PATH 替换，当然也包括下面的 nfs 配置，我们可以看到我们这里使用了一个名为 nfs-client-provisioner 的`serviceAccount`，所以我们也需要创建一个 sa，然后绑定上对应的权限：（nfs-client-sa.yaml）

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["create", "delete", "get", "list", "watch", "patch", "update"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
```

我们这里新建的一个名为 nfs-client-provisioner 的`ServiceAccount`，然后绑定了一个名为 nfs-client-provisioner-runner 的`ClusterRole`，而该`ClusterRole`声明了一些权限，其中就包括对`persistentvolumes`的增、删、改、查等权限，所以我们可以利用该`ServiceAccount`来自动创建 PV。

**第三步**：nfs-client 的 Deployment 声明完成后，我们就可以来创建一个`StorageClass`对象了：（nfs-client-class.yaml）

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: course-nfs-storage
provisioner: fuseim.pri/ifs # or choose another name, must match deployment's env PROVISIONER_NAME'
```

我们声明了一个名为 course-nfs-storage 的`StorageClass`对象，注意下面的`provisioner`对应的值一定要和上面的`Deployment`下面的 PROVISIONER_NAME 这个环境变量的值一样。

现在我们来创建这些资源对象吧：

```shell
$ kubectl create -f nfs-client.yaml
$ kubectl create -f nfs-client-sa.yaml
$ kubectl create -f nfs-client-class.yaml
```

创建完成后查看下资源状态：

```bash
$ kubectl get pods
NAME                                             READY     STATUS             RESTARTS   AGE
...
nfs-client-provisioner-7648b664bc-7f9pk          1/1       Running            0          7h
...
$ kubectl get storageclass
NAME                 PROVISIONER      AGE
course-nfs-storage   fuseim.pri/ifs   11s
```

## 新建

上面把`StorageClass`资源对象创建成功了，接下来我们来通过一个示例测试下动态 PV，首先创建一个 PVC 对象：(test-pvc.yaml)

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

我们这里声明了一个 PVC 对象，采用 ReadWriteMany 的访问模式，请求 1Mi 的空间，但是我们可以看到上面的 PVC 文件我们没有标识出任何和 StorageClass 相关联的信息，那么如果我们现在直接创建这个 PVC 对象能够自动绑定上合适的 PV 对象吗？显然是不能的(前提是没有合适的 PV)，我们这里有两种方法可以来利用上面我们创建的 StorageClass 对象来自动帮我们创建一个合适的 PV:

- 第一种方法：在这个 PVC 对象中添加一个声明 StorageClass 对象的标识，这里我们可以利用一个 annotations 属性来标识，如下：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: "course-nfs-storage"
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
```

- 第二种方法：我们可以设置这个 course-nfs-storage 的 StorageClass 为 Kubernetes 的默认存储后端，我们可以用 kubectl patch 命令来更新：

```shell
$ kubectl patch storageclass course-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

上面这两种方法都是可以的，当然为了不影响系统的默认行为，我们这里还是采用第一种方法，直接创建即可：

```shell
$ kubectl create -f test-pvc.yaml
persistentvolumeclaim "test-pvc" created
$ kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
...
test-pvc     Bound     pvc-73b5ffd2-8b4b-11e8-b585-525400db4df7   1Mi        RWX            course-nfs-storage    2m
...
```

我们可以看到一个名为 test-pvc 的 PVC 对象创建成功了，状态已经是 Bound 了，是不是也产生了一个对应的 VOLUME 对象，最重要的一栏是 STORAGECLASS，现在是不是也有值了，就是我们刚刚创建的 StorageClass 对象 course-nfs-storage。

然后查看下 PV 对象呢：

```shell
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                STORAGECLASS          REASON    AGE
...
pvc-73b5ffd2-8b4b-11e8-b585-525400db4df7   1Mi        RWX            Delete           Bound       default/test-pvc     course-nfs-storage              8m
...
```

可以看到是不是自动生成了一个关联的 PV 对象，访问模式是 RWX，回收策略是 Delete，这个 PV 对象并不是我们手动创建的吧，这是通过我们上面的 StorageClass 对象自动创建的。这就是 StorageClass 的创建方法。

## 测试

接下来我们还是用一个简单的示例来测试下我们上面用 StorageClass 方式声明的 PVC 对象吧：(test-pod.yaml)

```yaml
kind: Pod
apiVersion: v1
metadata:
  name: test-pod
spec:
  containers:
  - name: test-pod
    image: busybox
    imagePullPolicy: IfNotPresent
    command:
    - "/bin/sh"
    args:
    - "-c"
    - "touch /mnt/SUCCESS && exit 0 || exit 1"
    volumeMounts:
    - name: nfs-pvc
      mountPath: "/mnt"
  restartPolicy: "Never"
  volumes:
  - name: nfs-pvc
    persistentVolumeClaim:
      claimName: test-pvc
```

上面这个 Pod 非常简单，就是用一个 busybox 容器，在 /mnt 目录下面新建一个 SUCCESS 的文件，然后把 /mnt 目录挂载到上面我们新建的 test-pvc 这个资源对象上面了，要验证很简单，只需要去查看下我们 nfs 服务器上面的共享数据目录下面是否有 SUCCESS 这个文件即可：

```shell
$ kubectl create -f test-pod.yaml
pod "test-pod" created
```

然后我们可以在 nfs 服务器的共享数据目录下面查看下数据：

```shell
$ ls /media/pv01/
default-test-pvc-pvc-73b5ffd2-8b4b-11e8-b585-525400db4df7
```

我们可以看到下面有名字很长的文件夹，这个文件夹的命名方式是不是和我们上面的规则：**${namespace}-${pvcName}-${pvName}**是一样的，再看下这个文件夹下面是否有其他文件：

```shell
$ ls /data/k8s/default-test-pvc-pvc-73b5ffd2-8b4b-11e8-b585-525400db4df7
SUCCESS
```

我们看到下面有一个 SUCCESS 的文件，是不是就证明我们上面的验证是成功的啊。

另外我们可以看到我们这里是手动创建的一个 PVC 对象，在实际工作中，使用 StorageClass 更多的是 StatefulSet 类型的服务，StatefulSet 类型的服务我们也可以通过一个 volumeClaimTemplates 属性来直接使用 StorageClass，如下：(test-statefulset-nfs.yaml)

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nfs-web
spec:
  serviceName: "nginx"
  selector:
    matchLabels:
      app: nfs-web
  template:
    metadata:
      labels:
        app: nfs-web
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
      annotations:
        volume.beta.kubernetes.io/storage-class: course-nfs-storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Mi
```

实际上 volumeClaimTemplates 下面就是一个 PVC 对象的模板，就类似于我们这里 StatefulSet 下面的 template，实际上就是一个 Pod 的模板，我们不单独创建成 PVC 对象，而用这种模板就可以动态的去创建了对象了，这种方式在 StatefulSet 类型的服务下面使用得非常多。

直接创建上面的对象：

```shell
$ kubectl create -f test-statefulset-nfs.yaml
statefulset.apps "nfs-web" created
$ kubectl get pods
NAME                                             READY     STATUS              RESTARTS   AGE
...
nfs-web-0                                        1/1       Running             0          1m
nfs-web-1                                        1/1       Running             0          1m
nfs-web-2                                        1/1       Running             0          33s
...
```

创建完成后可以看到上面的3个 Pod 已经运行成功，然后查看下 PVC 对象：

```shell
$ kubectl get pvc
NAME            STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
...
www-nfs-web-0   Bound     pvc-cc36b3ce-8b50-11e8-b585-525400db4df7   1Gi        RWO            course-nfs-storage    2m
www-nfs-web-1   Bound     pvc-d38285f9-8b50-11e8-b585-525400db4df7   1Gi        RWO            course-nfs-storage    2m
www-nfs-web-2   Bound     pvc-e348250b-8b50-11e8-b585-525400db4df7   1Gi        RWO            course-nfs-storage    1m
...
```

我们可以看到是不是也生成了3个 PVC 对象，名称由模板名称 name 加上 Pod 的名称组合而成，这3个 PVC 对象也都是 绑定状态了，很显然我们查看 PV 也可以看到对应的3个 PV 对象：

```shell
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                   STORAGECLASS          REASON    AGE
...                                                        1d
pvc-cc36b3ce-8b50-11e8-b585-525400db4df7   1Gi        RWO            Delete           Bound       default/www-nfs-web-0   course-nfs-storage              4m
pvc-d38285f9-8b50-11e8-b585-525400db4df7   1Gi        RWO            Delete           Bound       default/www-nfs-web-1   course-nfs-storage              4m
pvc-e348250b-8b50-11e8-b585-525400db4df7   1Gi        RWO            Delete           Bound       default/www-nfs-web-2   course-nfs-storage              4m
...
```

查看 nfs 服务器上面的共享数据目录：

```shell
$ ls /media/pv01/
...
default-www-nfs-web-0-pvc-cc36b3ce-8b50-11e8-b585-525400db4df7
default-www-nfs-web-1-pvc-d38285f9-8b50-11e8-b585-525400db4df7
default-www-nfs-web-2-pvc-e348250b-8b50-11e8-b585-525400db4df7
...
```

是不是也有对应的3个数据目录，这就是我们的 StorageClass 的使用方法，对于 StorageClass 多用于 StatefulSet 类型的服务，在后面的课程中我们还学不断的接触到。
