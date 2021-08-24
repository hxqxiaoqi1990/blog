---
title: linux系统监控工具
tags: [linux]
categories: 监控
date: 2019-12-27 16:56:00
---

**iftop**

监控网络流向

```bash
# 下载
wget http://www.ex-parrot.com/pdw/iftop/download/iftop-0.17.tar.gz
tar xf iftop-0.17.tar.gz
cd iftop-0.17/

# 编译
yum -y install gcc flex byacc  libpcap ncurses ncurses-devel libpcap-devel
./configure
make && make install
```

**iotop**

监控进程磁盘IO

```bash
yum -y install iotop
```

**vmstat**

命令用来显示Linux系统虚拟内存状态，也可以报告关于进程、内存、I/O等系统整体运行状态

```bash
# 显示3次，间隔2秒
vmstat 2 3
```



