---
title: docker基本使用
tags: [docker]
categories: kubernetes
date: 2019-06-012 15:27:43
---


# docker是什么
<font color=DarkTurquoise>**docker**</font>是一个开源的应用容器引擎，开发者可以打包自己的应用到容器里面，然后迁移到其他机器的docker应用中，可以实现快速部署。如果出现的故障，可以通过镜像，快速恢复服务。

docker是利用Linux内核虚拟机化技术（LXC），提供轻量级的虚拟化，以便隔离进程和资源。LXC不是硬件的虚拟化，而是Linux内核的级别的虚拟机化，相对于传统的虚拟机，节省了很多硬件资源。

<font color=DarkTurquoise>**NameSpace**</font>

LXC是利用内核namespace技术，进行进程隔离。其中pid, net, ipc, mnt, uts 等namespace将container的进程, 网络, 消息, 文件系统和hostname 隔离开。

<font color=DarkTurquoise>**Control Group**</font>

LXC利用的宿主机共享的资源，虽然用namespace进行隔离，但是资源使用没有收到限制，这里就需要用到Control Group技术，对资源使用进行限制，设定优先级，资源控制等。
# docker安装

## 安装
软件源：阿里云镜像
``` bash
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
yum makecache fast
yum -y install docker-ce
```


## 镜像加速

``` bash
mkdir -p /etc/docker

tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://pthx0mbz.mirror.aliyuncs.com"]
}
EOF
```

## 启动
``` bash
systemctl start docker
systemctl enable docker
```

## 删除

``` bash
yum remove docker-ce
rm -rf /var/lib/docker
```

# docker常用命令

``` bash
docker info
#显示守护进程的系统资源

docker search imageID
#Docker仓库的查询,用户名/镜像名:版本，最好选官方的

docker pull imageID
#Docker仓库的下载

docker images
#Docker镜像的查询

docker rmi imageID/镜像名:版本
#Docker	镜像的删除，-f强制删除

docker ps -a
#正在运行容器的查询，-a查询所有容器

docker run --restart always -v /data/:/var/log/ -v /etc/localtime:/etc/localtime -v /etc/timezone:/etc/timezone -p 服务器端口:容器端口 --name 定义容器名 -d 指定images:tag
#--restart always在容器自启动
#-v /data/:/var/log/是持久化目录
#-v /etc/localtime:/etc/localtime -v /etc/timezone:/etc/timezone  持久化容器时间，如果没有timezone文件，执行echo 'Asia/Shanghai' >/etc/timezone

#更新启动命令，在容器创建时没使用的参数
docker update --restart=always xxx

docker exec -it 容器名 bash
#进入容器中

docker start/stop 容器别名/ID
#容器启动停止

docker rm -f 容器名
#强制删除容器

docker rm -f `docker ps -a -q`
#删除所有容器

docker rmi -f
#强制删除镜像

docker save -o rocketmq.tar rocketmq 
#-o：指定保存的镜像的名字；rocketmq.tar：保存到本地的镜像名称；rocketmq：镜像名字

docker load --input rocketmq.tar
#导入镜像
```

# docker网络模式
docker network ls  #查看当前可用的网络类型
--net=网络类型	#创建容器时指定

<font color=DarkTurquoise>**--net=bridge**</font> 这个是默认值，连接到默认的网桥，容器IP自动生成，相互可以访问，共用一个docker0网桥。

<font color=DarkTurquoise>**--net=host**</font> 告诉 Docker 不要将容器网络放到隔离的名字空间中，即不要容器化容器内的网络。此时容器使用本地主机的网络，它拥有完全的本地主机接口访问权限。容器进程可以跟主机其 它 root 进程一样可以打开低范围的端口，可以访问本地网络服务比如 D-bus，还可以让容器做一些影响整个主机系统的事情，比如重启主机。因此使用这个选项的时候要非常小心。如果进一步的使用 --privileged=true，容器会被允许直接配置主机的网络堆栈。

<font color=DarkTurquoise>**--net=container:NAME_or_ID**</font> 让 Docker 将新建容器的进程放到一个已存在容器的网络栈中，新容器进程有自己的文件系统、进程列表和资源限制，但会和已存在的容器共享 IP 地址和端口等网络资源，两者进程可以直接通过 lo 环回接口通信。

<font color=DarkTurquoise>**--net=none**</font> 让 Docker 将新容器放到隔离的网络栈中，但是不进行网络配置。之后，用户可以自己进行配置。