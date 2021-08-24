---
title: k8s安装之sealos
tags: [k8s]
categories: kubernetes
date: 2020-05-22 16:00:00
---

# 介绍

极其方便的k8s安装方式

参考资料：

github地址：https://github.com/fanux/sealos

Kuboard：https://kuboard.cn/install/install-dashboard.html

# 安装

<font color='32CD32'>虚拟机环境</font>

| 主机名 |     IP地址     |   系统    |
| :----: | :------------: | :-------: |
| etc01  | 192.168.10.121 | centos7.6 |
| etc02  | 192.168.10.122 | centos7.6 |
| etc03  | 192.168.10.123 | centos7.6 |
| node01 | 192.168.10.124 | centos7.6 |
| node02 | 192.168.10.125 | centos7.6 |

<font color='32CD32'>安装环境</font>

每台虚拟机执行以下脚本，简单的系统优化，也可以不需要执行

```bash
# CentOS-7.6
rm -rf /etc/yum.repos.d/*
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
yum clean all && yum makecache
echo "export PS1='\[\e[1;32m\][\u@\h \W]\\$ \[\e[0m\]'" >> /etc/bashrc
echo "export TIME_STYLE='+%Y-%m-%d %H:%M:%S'" >> /etc/bashrc
echo "export HISTTIMEFORMAT='%F %T '" >> /etc/profile
sed -i "s/^#UseDNS.*/UseDNS no/g" /etc/ssh/sshd_config
sed -i "s/SELINUX=.*/SELINUX=disabled/g" /etc/selinux/config
systemctl stop firewalld && systemctl disable firewalld && rpm -e --nodeps firewalld
systemctl disable chronyd
systemctl stop chronyd
yum -y install iptables-services
systemctl start iptables && iptables -F && service iptables save
yum -y install lrzsz net-tools vim psmisc bash-completion kernel-tools tree wget dos2unix ntpdate unzip tcpdump
cat >> ~/.vimrc <<EOF
set ts=4
set expandtab
set paste
EOF
# 根据虚拟机设置的主机名更改
hostnamectl set-hostname node03
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
yum --enablerepo=elrepo-kernel install kernel-ml -y
grub2-set-default 0
grub2-mkconfig -o /boot/grub2/grub.cfg
yum install ntp -y
ntpdate -u ntp1.aliyun.com
reboot
```

<font color='32CD32'>安装命令</font>

```bash
# 下载并安装sealos, sealos是个golang的二进制工具，直接下载拷贝到bin目录即可, release页面也可下载
$ wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/latest/sealos && \
    chmod +x sealos && mv sealos /usr/bin 

# 下载离线资源包
$ wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/d551b0b9e67e0416d0f9dce870a16665-1.18.0/kube1.18.0.tar.gz 

# 安装一个三master的kubernetes集群，密码需要一致
$ sealos init --passwd 123456 \
	--master 192.168.10.121  --master 192.168.10.122  --master 192.168.10.123  \
	--node 192.168.10.124 \
	--pkg-url /root/kube1.18.0.tar.gz \
	--version v1.18.0
```

<font color='32CD32'>集群操作</font>

增加master

```bash
sealos join --master 192.168.0.6 --master 192.168.0.7
sealos join --master 192.168.0.6-192.168.0.9  # 或者多个连续IP
```

增加node

```bash
sealos join --node 192.168.0.6 --node 192.168.0.7
sealos join --node 192.168.0.6-192.168.0.9  # 或者多个连续IP
```

删除指定master节点

```bash
sealos clean --master 192.168.0.6 --master 192.168.0.7
sealos clean --master 192.168.0.6-192.168.0.9  # 或者多个连续IP
```

删除指定node节点

```bash
sealos clean --node 192.168.0.6 --node 192.168.0.7
sealos clean --node 192.168.0.6-192.168.0.9  # 或者多个连续IP
```

清理集群

```bash
sealos clean
```