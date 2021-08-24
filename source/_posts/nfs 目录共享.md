---
title: nfs 目录共享
tags: [nfs]
categories: linux基础
date: 2019-06-05 17:40:43
---

# 介绍

NFS 是Network File System的缩写，即网络文件系统。一种使用于分散式文件系统的协定，由Sun公司开发，于1984年向外公布。功能是通过网络让不同的机器、不同的操作系统能够彼此分享个别的数据，让应用程序在客户端通过网络访问位于服务器磁盘中的数据，是在类Unix系统间实现磁盘文件共享的一种方法。

# 安装

<font color='32CD32'>实验环境</font>

|    服务器IP    | 安装服务  |
| :------------: | :-------: |
| 192.168.40.100 | nfs-utils |
| 192.168.40.101 | nfs-utils |

<font color='32CD32'>nfs 服务端安装</font>

在192.168.40.100 操作

```bash
# 安装nfs（实际上需要安装两个包nfs-utils和rpcbind, 不过当使用yum安装nfs-utils时会把rpcbind一起安装上）
yum -y install nfs-utils

# 创建共享目录
mkdir /data

# 创建文件用户，uid为1000
useradd huang
```

```bash
# 修改nfs配置文件
vim /etc/exports
```

```bash
/data 192.168.40.101(rw,all_squash,anonuid=1000,anongid=1000)
```

- /data：为共享目录
- 192.168.40.101：为允许的客户端ip，可以是IP段（192.168.40.0/24），也可以是单个IP，或域名
- sync ：同步模式，内存中数据时时写入磁盘；async ：不同步，把内存中数据定期写入磁盘中
- no_root_squash ：加上这个选项后，root用户就会对共享的目录拥有至高的权限控制，就像是对本机的目录操作一样。不安全，不建议使用；root_squash：和上面的选项对应，root用户对共享目录的权限不高，只有普通用户的权限，即限制了root；all_squash：不管使用NFS的用户是谁，他的身份都会被限定成为一个指定的普通用户身份
- anonuid/anongid ：要和root_squash 以及all_squash一同使用，用于指定使用NFS的用户限定后的uid和gid，前提是本机的/etc/passwd中存在这个uid和gid
- fsid=0：表示将/opt/nfs整个目录包装成根目录
- 如果需要共享多目录，可以在另取一行按以上格式填写

<font color='32CD32'>启动</font>

```bash
systemctl start rpcbind.service
systemctl start nfs.service
```

<font color='32CD32'>查看是否启动</font>

```bash
# 通过查看service列中是否有nfs服务来确认NFS是否启动。
rpcinfo -p    
```

<font color='32CD32'>查看可挂载目录及可连接的IP</font>

```bash
showmount -e 192.168.40.100
```

<font color='32CD32'>重新加载exports配置</font>

```bash
exportfs -arv
```

<font color='32CD32'>nfs客户端</font>

在192.168.40.101 操作

```bash
# 创建挂在目录
mkdir -p /data/test

# 查看nfs服务端可用共享目录
showmount -e 192.168.40.100

# 挂载目录
mount -t nfs 192.168.40.100:/data /data/test

# 查看是否挂在
df -h
```

<font color='32CD32'>开机启动挂载</font>

```
vim /etc/fstab
```

```bash
192.168.40.100:/opt/nfs /data/test nfs nolock 0 0
```

