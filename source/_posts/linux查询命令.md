---
title: linux查询命令
tags: [linux]
categories: linux基础
date: 2019-12-27 16:56:00
---

# 介绍
查看系统的相关信息

# 常用查看命令
## <font color='32CD32'>系统硬件信息</font>

#查看 CPU 物理个数
grep 'physical id' /proc/cpuinfo | sort -u | wc -l

#查看 CPU 核心数量
grep 'core id' /proc/cpuinfo | sort -u | wc -l

#查看 CPU 线程数
grep 'processor' /proc/cpuinfo | sort -u | wc -l

#查看 CPU  型号
dmidecode -s processor-version

#查看 CPU 的详细信息：
cat /proc/cpuinfo

## <font color='32CD32'>系统信息</font>
#查看内核/操作系统/CPU信息的linux系统信息命令
uname -a     

#系统版本
cat /etc/redhat-release 

## <font color='32CD32'>系统资源信息</font>
#系统最大打开文件描述符数
cat /proc/sys/fs/file-max

#单个进程可分配最大文件数
cat/proc/sys/fs/nr_open		

#查看当前系统使用的打开文件描述符数
cat /proc/sys/fs/file-nr		

## <font color='32CD32'>yum|rpm查询命令</font>
#列出所有可更新的软件清单命令
yum check-update

#更新所有软件命令
yum update

#仅安装指定的软件命令
yum install <package_name>

#仅更新指定的软件命令
yum update <package_name>

#列出所有可安裝的软件清单命令
yum list

#删除软件包命令
yum remove <package_name>

#查找软件包 命令
yum search <keyword>

#清除缓存命令
yum clean all

#建立缓存
yum makecache

#下载包及依赖

yum install --downloadonly --downloaddir=/root/openssh-ssl 包名

yum reinstall --downloadonly --downloaddir=/root/openssh-ssl 包名



## <font color='32CD32'>systemctl系统命令</font>
#查看服务启动项
systemctl list-unit-files 