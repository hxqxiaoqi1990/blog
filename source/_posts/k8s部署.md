---
title: k8s之kubeadm部署
tags: [k8s]
categories: kubernetes
date: 2019-09-17 15:06:00
---

# 介绍
 Kubernetes是当今最流行的开源容器管理平台，它就是大名鼎鼎的Google Borg的开源版本。Google在2014年推出了Kubernetes，本文发布时最新的版本是1.11。

Kubernetes源于希腊语，意为舵手，K8S是一个简称，因为首尾字母中间正好有8个字母。基于容器技术，Kubernetes可以方便的进行集群应用的部署、扩容、缩容、自愈机制、服务发现、负载均衡、日志、监控等功能，大大减少日常运维的工作量。

Kubernetes所有的操作都可以通过Kubernetes API来进行，通过API来操作Kubernetes中的对象，包括Pod、Service、Volume、Namespace等等。
# 组件说明
<font color='98F5FF'>API Server：</font>整个集群管理的 API 接口，所有对集群进行的查询和管理都要通过 API 来进行集群内部各个模块之间通信的枢纽，所有模块之前并不会之间互相调用，而是通过和 API Server 打交道来完成自己那部分的工作集群安全控制，API Server 提供的验证和授权保证了整个集群的安全。

<font color='98F5FF'>Kube Proxy：</font>负责为pod提供代理。它会定期从etcd获取所有的service，并根据service信息创建代理。当某个客户pod要访问其他pod时，访问请求会经过本机proxy做转发。

<font color='98F5FF'>Scheduler：</font>负责集群的资源调度,跟踪集群中所有Node的资源利用情况,对新创建的pod，采取合适的调度策略，进行均衡的调度到合适的节点上。

<font color='98F5FF'>etcd：</font>Kubernetes所有的配置数据存储在etcd中；所有组件通过API Server和etcd打交道。

<font color='98F5FF'>Controller Manager：</font>主要负责集群的故障检测和恢复的自动化，它内部的组件如下:
1.endpointController:定期关联service和pod，关联信息由endpoint负责创建和更新。
2.ReplicationController:完成pod的复制或移除，以确保ReplicationController定义的复本数量与实际运行pod的数量一致性

<font color='98F5FF'>CoreDNS：</font>Kubernetes包括用于服务发现的DNS服务器Kube-DNS。 该DNS服务器利用SkyDNS的库来为Kubernetes pod和服务提供DNS请求。在这种灵活的模型中添加对Kubernetes的支持，相当于创建了一个Kubernetes中间件。该中间件使用Kubernetes API来满足针对特定Kubernetes pod或服务的DNS请求。而且由于Kube-DNS作为Kubernetes的另一项服务，kubelet和Kube-DNS之间没有紧密的绑定。您只需要将DNS服务的IP地址和域名传递给kubelet，而Kubernetes并不关心谁在实际处理该IP请求。

<font color='98F5FF'>pause：</font>我们看下在node节点上都会起很多pause容器，和pod是一一对应的。每个Pod里运行着一个特殊的被称之为Pause的容器，其他容器则为业务容器，这些业务容器共享Pause容器的网络栈和Volume挂载卷，因此他们之间通信和数据交换更为高效，在设计时我们可以充分利用这一特性将一组密切相关的服务进程放入同一个Pod中。同一个Pod里的容器之间仅需通过localhost就能互相通信。

<font color='98F5FF'>Flannel：</font>通过给每台宿主机分配一个子网的方式为容器提供虚拟网络，它基于Linux TUN/TAP，使用UDP封装IP包来创建overlay网络，并借助etcd维护网络的分配情况。简单说就是给pod分配ip的。

<font color='98F5FF'>kubelet：</font>每个node上都会启动一个kubelet服务进程，相当于agent进程，该进程用于处理master节点下发到本节点的任务，管理Pod及Pod中的容器，同时Kubelet进程会在API Server上注册节点自身的信息，定期向Master节点汇报节点的资源情况，并通过cAdvise监控容器和节点资源；它负责创建和销毁Pod，通过探针监测Pod的状态，并通过cAdvise监控Pod的状态。

<font color='98F5FF'>kubeadm：</font>k8s的一键安装工具。

<font color='98F5FF'>kubectl：</font>通过apiserver组件向整个集群內发送控制命令的管理工具。

<font color='98F5FF'>service：</font>是pod的路由代理抽象，用于解决pod之间的服务发现问题，即上下游pod之间使用的问题。传统部署方式中，实例所在的主机ip（或者dns名字）一般是不会改变的，但是pod的运行状态可动态变化(比如容器重启、切换机器了、缩容过程中被终止了等)，所以访问端不能以写死IP的方式去访问该pod提供的服务。service的引入旨在保证pod的动态变化对访问端透明，访问端只需要知道service的地址，由service来提供代理。Service分为三类：ClusterIP，NodePort，LoadBalancer。

<font color='98F5FF'>Pod：</font>Pod是kubernetes集群中的最小单元，可以运行一个或多个容器，Pod对外共享一个ip，通过livenessProbe探针监测容器是否健康，另外一类是readinessProbe探针来判断容器是否启动完成。

<font color='98F5FF'>Deployment：</font>为Pod和Replica Set提供声明式更新。你只需要在 Deployment 中描述您想要的目标状态是什么，Deployment controller 就会帮您将 Pod 和ReplicaSet 的实际状态改变到您的目标状态。您可以定义一个全新的

# kubeadm部署
## 环境准备（所有节点都要操作）
实验虚拟机：
k8s-master：192.168.40.100 
k8s-node1：192.168.40.101
k8s-node2：192.168.40.102 

**设置主机名：**

``` bash
hostnamectl set-hostname k8s-master
hostnamectl set-hostname k8s-node1
hostnamectl set-hostname k8s-node2
```

**配置hosts文件解析：**

``` bash
cat >> /etc/hosts << EOF
192.168.40.100 k8s-master
192.168.40.101 k8s-node1
192.168.40.102 k8s-node2
EOF
```

**关闭防火墙和selinux：**

``` bash
systemctl stop firewalld && systemctl disable firewalld
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config && setenforce 0
```

**关闭swap：**

``` bash
swapoff -a 
sed -i 's/.*swap.*/#&/' /etc/fstab
```
**设置网桥包经过iptalbes：**
RHEL / CentOS 7上的一些用户报告了由于iptables被绕过而导致流量路由不正确的问题。
``` bash
# 创建/etc/sysctl.d/k8s.conf文件，添加如下内容
cat <<EOF >  /etc/sysctl.d/k8s.conf
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# 使配置生效
modprobe br_netfilter
sysctl -p /etc/sysctl.d/k8s.conf
```
**kube-proxy开启ipvs的前提条件：**
由于ipvs已经加入到了内核的主干，所以为kube-proxy开启ipvs的前提需要加载以下的内核模块：

``` bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

# 执行脚本
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4

# 查看是否已经正确加载所需的内核模块
lsmod | grep -e ip_vs -e nf_conntrack_ipv4   
```

**安装ipvsadm：**
接下来还需要确保各个节点上已经安装了ipset软件包，为了便于查看ipvs的代理规则。

``` bash
yum -y install ipset ipvsadm 
```

**安装Docker：**

``` bash
yum -y install yum-utils

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum install -y docker-ce-18.06.1.ce

systemctl start docker && systemctl enable docker
```
**配置kubernetes源：**

``` bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

**安装kubeadm、kubelet、kubectl：**

``` bash
yum install -y kubelet-1.13.1 
yum install -y kubeadm-1.13.1 
yum install -y kubectl-1.13.1

systemctl enable kubelet && systemctl start kubelet
```

## 设置时间同步
**配置k8s-master节点：**

``` bash
# 安装chrony：
yum install -y chrony

# 注释默认ntp服务器
sed -i 's/^server/#&/' /etc/chrony.conf

# 指定上游公共 ntp 服务器，并允许其他节点同步时间
cat >> /etc/chrony.conf << EOF
server 0.asia.pool.ntp.org iburst
server 1.asia.pool.ntp.org iburst
server 2.asia.pool.ntp.org iburst
server 3.asia.pool.ntp.org iburst
allow all
EOF

# 重启chronyd服务并设为开机启动：
systemctl enable chronyd && systemctl restart chronyd

# 开启网络时间同步功能
timedatectl set-ntp true
```
**配置k8s-node节点：**

``` bash
# 安装chrony：
yum install -y chrony

# 注释默认服务器
sed -i 's/^server/#&/' /etc/chrony.conf

# 指定内网 master节点为上游NTP服务器
echo server 192.168.40.100 iburst >> /etc/chrony.conf

# 重启服务并设为开机启动：
systemctl enable chronyd && systemctl restart chronyd

# 所有节点执行，查看是否同步成功
chronyc sources
```

## master节点操作
**初始化k8s-master：**
初始化过程需要下载镜像，要等待几分钟，注意更改主机IP。
注意：如果cpu需要2核，否则会报错。

``` bash
# 初始化，成功后会生成节点加入集群的命令，保存下来
kubeadm init --apiserver-advertise-address=192.168.40.100 --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.13.1 --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# 查看集群状态
kubectl get cs
```
**下载flannel网络插件：**
如果网络不好，需要耐心等待flannel进行下载完成
``` bash
# 安装flannel插件
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# 查看coredns是否运行
kubectl get pod -n kube-system -o wide

# 如果节点Ready状态，即表示运行正常
kubectl get nodes 
```

## node操作加入集群

``` bash
# 执行，这是master节点出生成功后生成的
kubeadm join 192.168.40.100:6443 --token 0jxnxc.5pn1l3z931kcdqzv --discovery-token-ca-cert-hash sha256:d3de7befd0aa46d999c8f3a4880e45b7f4ab87dafa54c39232078b6cec5b1833

#如果执行kubeadm init时没有记录下加入集群的命令，可以通过以下命令重新创建

# 查看节点，如果node显示，则成功
kubectl get nodes
```

