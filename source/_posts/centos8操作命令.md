---
title: centos8操作命令
tags: [linux]
categories: linux基础
date: 2020-06-09 18:27:43
---

# 网络配置

```bash
# 查看ip（类似于ifconfig、ip addr）
nmcli

# 创建connection，配置静态ip（等同于配置ifcfg，其中BOOTPROTO=none，并ifup启动）
nmcli c add type ethernet con-name ethX ifname ethX ipv4.addr 192.168.1.100/24 ipv4.gateway 192.168.1.1 ipv4.method manual

nmcli c add type ethernet con-name ethX-test ifname ethX ipv4.addresses '192.168.1.100/24,192.168.1.101/32' ipv4.routes '10.0.0.0/8 192.168.1.10,192.168.0.0/16 192.168.1.11' ipv4.gateway 192.168.1.254 ipv4.dns '8.8.8.8,4.4.4.4' ipv4.method manual
# type ethernet：创建连接时候必须指定类型，类型有很多，可以通过nmcli c add type -h看到，这里指定为ethernet。
# con-name ethX ifname ethX：第一个ethX表示连接（connection）的名字，这个名字可以任意定义，无需和网卡名相同；第二个ethX表示网卡名，这个ethX必须是在nmcli d里能看到的。
# ipv4.addresses ‘192.168.1.100/24,192.168.1.101/32’：配置2个ip地址，分别为192.168.1.100/24和192.168.1.101/32
# ipv4.gateway 192.168.1.254：网关为192.168.1.254
# ipv4.dns ‘8.8.8.8,4.4.4.4’：dns为8.8.8.8和4.4.4.4
# ipv4.method manual：配置静态IP


# 创建connection，配置动态ip（等同于配置ifcfg，其中BOOTPROTO=dhcp，并ifup启动）
nmcli c add type ethernet con-name ethX ifname ethX ipv4.method auto

# 修改ip（非交互式）
nmcli c modify ethX ipv4.addr '192.168.1.200/24'
nmcli c up ethX

# 修改ip（交互式）
nmcli c edit ethX
nmcli> goto ipv4.addresses
nmcli ipv4.addresses> change
Edit 'addresses' value: 192.168.1.200/24
Do you also want to set 'ipv4.method' to 'manual'? [yes]: yes
nmcli ipv4> save
nmcli ipv4> activate
nmcli ipv4> quit

# 启用connection（相当于ifup）
nmcli c up ethX

# 停止connection（相当于ifdown）
nmcli c down

# 删除connection（类似于ifdown并删除ifcfg）
nmcli c delete ethX

# 查看connection列表
nmcli c show

# 查看connection详细信息
nmcli c show ethX

# 重载所有ifcfg或route到connection（不会立即生效）
nmcli c reload

# 重载指定ifcfg或route到connection（不会立即生效）
nmcli c load /etc/sysconfig/network-scripts/ifcfg-ethX
nmcli c load /etc/sysconfig/network-scripts/route-ethX

# 立即生效connection，有3种方法
nmcli c up ethX
nmcli d reapply ethX
nmcli d connect ethX

# 查看device列表
nmcli d

# 查看所有device详细信息
nmcli d show

# 查看指定device的详细信息
nmcli d show ethX

# 激活网卡
nmcli d connect ethX

# 关闭无线网络（NM默认启用无线网络）
nmcli r all off

# 查看NM纳管状态
nmcli n

# 开启NM纳管
nmcli n on

# 关闭NM纳管（谨慎执行）
nmcli n off

# 监听事件
nmcli m

# 查看NM本身状态
nmcli

# 检测NM是否在线可用
nm-online

```

# dnf管理

```bash
# 查看 DNF 包管理器版本
dnf –version

# 查看系统中可用的 DNF 软件库
dnf repolist

# 查看系统中可用和不可用的所有的 DNF 软件库
dnf repolist all

# 列出所有 RPM 包
dnf list

# 列出所有安装了的 RPM 包
dnf list installed

# 列出所有可供安装的 RPM 包
dnf list available

# 搜索软件库中的 RPM 包
dnf search nano

# 查找某一文件的提供者
dnf provides /bin/bash

# 查看软件包详情
dnf info nano

# 安装软件包
dnf install nano

# 升级软件包
dnf update systemd

# 检查系统软件包的更新
dnf check-update

# 升级所有系统软件包（都可以）
dnf update 
dnf upgrade

# 删除软件包（都可以）
dnf remove nano 
dnf erase nano

# 删除无用孤立的软件包
dnf autoremove

# 删除缓存的无用软件包
dnf clean all

# 获取有关某条命令的使用帮助
dnf help clean

# 查看所有的 DNF 命令及其用途
dnf help

# 查看 DNF 命令的执行历史
dnf history

# 查看所有的软件包组
dnf grouplist

# 安装一个软件包组
dnf groupinstall 'Educational Software'

# 升级一个软件包组中的软件包
dnf groupupdate 'Educational Software'

# 删除一个软件包组
dnf groupremove 'Educational Software'

# 从特定的软件包库安装特定的软件
dnf –enablerepo=epel install phpmyadmin

# 更新软件包到最新的稳定发行版
dnf distro-sync

# 重新安装特定软件包
dnf reinstall nano

# 回滚某个特定软件的版本
dnf downgrade acpid
```

