---
title: linux优化 
tags: [linux]
categories: linux基础
date: 2019-11-05 14:29:00
---


# 介绍
linux相关优化

# 虚拟机优化
用于第一次安装完linux后，执行，关闭防火墙和安装常用工具。

``` bash
# CentOS-7.4.1708
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
```
```bash
# CentOS-8.2
rm -rf /etc/yum.repos.d/*
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-8.repo
yum clean all && yum makecache
echo "export PS1='\[\e[1;32m\][\u@\h \W]\\$ \[\e[0m\]'" >> /etc/bashrc
echo "export TIME_STYLE='+%Y-%m-%d %H:%M:%S'" >> /etc/bashrc
echo "export HISTTIMEFORMAT='%F %T '" >> /etc/profile
sed -i "s/^#UseDNS.*/UseDNS no/g" /etc/ssh/sshd_config 
sed -i "s/SELINUX=.*/SELINUX=disabled/g" /etc/selinux/config
sed -i "s/pool 2.centos.pool.ntp.org iburst/#pool 2.centos.pool.ntp.org iburst/g" /etc/chrony.conf
sed -i '4a server ntp.aliyun.com iburst' /etc/chrony.conf
systemctl restart chronyd.service
systemctl enable chronyd.service
chronyc sources -v
timedatectl set-timezone Asia/Shanghai
systemctl stop firewalld && systemctl disable firewalld && rpm -e --nodeps firewalld
yum -y install iptables-services
systemctl start iptables && iptables -F && service iptables save
yum -y install lrzsz net-tools vim psmisc bash-completion kernel-tools tree wget dos2unix unzip tcpdump 
cat >> ~/.vimrc <<EOF
set ts=4
set expandtab
set paste
EOF
```



# 内核优化
Linux服务器内核参数优化,主要是指在Linux系统中针对业务服务应用而进行的系统内核参数调整,优化并无一定的标准.下面是生产环境下Linux常见的内核优化:

vim /etc/sysctl.conf
``` bash
# 关闭ipv6
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# 避免放大攻击
net.ipv4.icmp_echo_ignore_broadcasts = 1

# 开启恶意icmp错误消息保护
net.ipv4.icmp_ignore_bogus_error_responses = 1

# 关闭路由转发
net.ipv4.ip_forward = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# 开启反向路径过滤
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# 处理无源路由的包
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# 关闭sysrq功能
kernel.sysrq = 0

# core文件名中添加pid作为扩展名
kernel.core_uses_pid = 1

# 开启SYN洪水攻击保护
net.ipv4.tcp_syncookies = 1

# 修改消息队列长度
kernel.msgmnb = 65536
kernel.msgmax = 65536

# 设置最大内存共享段大小bytes
kernel.shmmax = 68719476736
kernel.shmall = 4294967296

# timewait的数量，默认180000
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096        87380   4194304
net.ipv4.tcp_wmem = 4096        16384   4194304
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

# 每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目
net.core.netdev_max_backlog = 262144

# 限制仅仅是为了防止简单的DoS 攻击
net.ipv4.tcp_max_orphans = 3276800

# 未收到客户端确认信息的连接请求的最大值
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 0

# 内核放弃建立连接之前发送SYNACK 包的数量
net.ipv4.tcp_synack_retries = 1

# 内核放弃建立连接之前发送SYN 包的数量
net.ipv4.tcp_syn_retries = 1

# 启用timewait 快速回收
net.ipv4.tcp_tw_recycle = 1

# 开启重用。允许将TIME-WAIT sockets 重新用于新的TCP 连接
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_fin_timeout = 1

# 当keepalive 起用的时候，TCP 发送keepalive 消息的频度。缺省是2 小时
net.ipv4.tcp_keepalive_time = 30

# 允许系统打开的端口范围
net.ipv4.ip_local_port_range = 1024    65000

# 修改防火墙表大小，默认65536
net.netfilter.nf_conntrack_max=655350
net.netfilter.nf_conntrack_tcp_timeout_established=1200

# 确保无人能修改路由表
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# 系统最大进程设置
kernel.pid_max = 32768

# 系统最大文件数设置
fs.file-max = 6553560
```

# ulimit（进程占用资源优化）

配置文件位置：vim /etc/security/limits.conf
centos6：/etc/security/limits.d/90-nproc.conf
centos7：/etc/security/limits.d/20-nproc.conf

ulimit -a

``` bash
core file size  		//限制内核文件的大小限制
data seg size   		//最大数据大小限制
scheduling priority   	//调度优先级，一般根据nice设置
file size        		//最大文件大小限制
pending signals  		//信号可以被挂起的最大数，这个值针对所有用户,表示可以被挂起/阻塞的最大信号数量。
max locked memory  		//内存锁定值的限制
max memory size    		//最大可以使用内存限制
open files     			//进程打开文件数的限制
pipe size        		//管道文件大小限制
POSIX message queues   	//可以创建使用POSIX消息队列的最大值,单位为bytes
real-time priority  	//限制程序实时优先级的范围,只针对普通用户
stack size           	//限制进程使用堆栈段的大小
cpu time            	//程序占用CPU的时间,单位是秒
max user processes   	//限制程序可以fork的进程数,只对普通用户有效
virtual memory       	//限制进程使用虚拟内存的大小
file locks           	//锁定文件大小限制
```

同时还有个要注意的值 file-max 是设置 系统所有进程一共可以打开的文件数量 ,可以通过如下方法进行修改echo 6553560 > /proc/sys/fs/file-max
或修改 /etc/sysctl.conf, 加入fs.file-max = 6553560 重启生效
另外还有一个，/proc/sys/fs/file-nr，可以看到整个系统目前使用的文件句柄数量
