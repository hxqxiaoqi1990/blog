---
title: kvm安装与使用 
tags: [kvm]
categories: linux基础
date: 2019-10-26 14:07:00
---

# 介绍

**kvm：** 全称“基于内核的虚拟机”，是一个开源的软件，基于内核的虚拟化技术，实际是嵌入系统的一个虚拟化模块，通过优化内核来使用虚拟技术，该内核模块使得Linux变成一个hypervisor，虚拟机使用Linux自身的调度器进行管理。（就是说Linux要部署一个kvm模块，他才能变成hypervisor层）。

kvm是基于CPU的类型进行管理。

 1. 用户空间：指的是用户得到一个虚拟机。
 2. 内核空间：指的是你的kvm宿主机里面它部署的虚拟化的软件，是通过驱动内核来实现的。

**组件说明：**
1. 虚机：指的是用户的得到一个虚拟机层。
2. Guest：指的我们虚拟机，也称VM。
3. kvm：运行在内核空间，提供CPU和内存的虚拟。
4. QEMU（扩展软件）：帮我们提供了虚拟机的I/O设备（CPU 内存 显示器），其他的硬件虚拟化。
5. kvm有一个内核模块叫kvm.ko，它来提供我们CPU和内存。
6. Libvirt：kvm的管理工具。
7. virt-viewer：轻量级桌面工具。
8. bridge-utils ：网桥工具。
9. virt-install：KVM虚拟机的管理主要是通过virsh命令对虚拟机进行管理。

**环境要求：**

 1. KVM需要硬件⽀持, 所以需要开启虚拟化⽀持。
 2. 硬件设备直接在BIOS设置开启CPU虚拟化。
 3. 个⼈电脑同样进⼊BIOS开启虚拟化⽀持。
 4. VM需要找到对应虚拟机开启对应的VT-EPT虚拟化技术。

# 部署kvm
**安装**

``` bash
yum install -y qemu-kvm libvirt virt-install bridge-utils virt-viewer

# 查看是否开启虚拟化
lsmod |grep kvm

# 启动管理工具
systemctl start  libvirtd
systemctl enable libvirtd
```

**修改网卡**

vim  /etc/sysconfig/network-scripts/ifcfg-ens33
``` bash
NAME=ens33
DEVICE=ens33
BOOTPROTO=none
NM_CONTROLLED=no
ONBOOT=yes
BRIDGE=br0
```
vim ifcfg-br0
设置为本机IP，当作网桥
``` bash
NAME=br0
DEVICE=br0
ONBOOT=yes
NETBOOT=yes
IPV6INIT=no
BOOTPROTO=static
NM_CONTROLLED=no
TYPE=Bridge
IPADDR=192.168.0.127
NETMASK=255.255.255.0
GATEWAY=192.168.0.1
DNS1=8.8.8.8
```
重启网卡

``` bash
systemctl restart network
```
查看网桥

``` bash
brctl show
```
**创建虚拟机**

``` bash
# 创建目录，iso用于存放系统镜像，kvm用于存放创建的虚拟机磁盘文件，保证空间
mkdir -p /data/{iso,kvm}

# 导入镜像到iso目录下

# 创建虚拟磁盘
cd /data/kvm/; qemu-img create -f qcow2 /data/kvm/disk/flink02.qcow2 200G

# 创建虚拟机
# --arch=x86_64 为虚拟机请求一个非本地CPU架构，这个选项当前只对qemu客户机有效，但是不能够使用加速机制。如果忽略，在虚拟机中将使用主机CPU架构。
# --vcpus=1 cpu数
# --disk 指定磁盘类型，bus：磁盘总结类型，其值可以为ide、scsi、usb、virtio或xen；cache：缓存模型，其值有none、writethrouth（缓存读）及writeback（缓存读写）。
# --os-type 针对一类操作系统优化虚拟机配置
# --vnc vnc监听端口
virt-install -n flink02 --arch=x86_64 --vcpus=4 -r 16384 --disk path=/data/kvm/disk/flink02.qcow2,io=native,bus=virtio,cache=none --network bridge=br0,model=virtio --os-type=linux --os-variant=rhel7 --cdrom /data/ISO/CentOS-7-x86_64-DVD-1611.iso --vnc --vncport=7001 --vnclisten=0.0.0.0 --video=vga

# 另开一个终端，查看虚拟机列表
virsh list
```

# 安装vnc

**linux下载vnc服务端**
``` bash
yum install -y tigervnc-server tigervnc-server-module

# 启动vnc，进入交互式界面，设置vnc连接密码
vncserver

```
**在windows：** [下载vncviewer客户端](https://d11.baidupcs.com/file/19d5f22dd5351b10c7a82795ae827a35?bkt=en-40ebf341379bd9a045bd167e394aeff34204e2a7e4b35305754d10c11832e851a73cb4e9880a6a61235817c8f4333c966853e26d73ff6081d3426a1254004497&xcode=b706998e826bf38845c23a653cfd0e778ba2f1aa68560750fa8414341c215772bd8a81a5f5880760f5ba5cbb0ec279470b2977702d3e6764&fid=2102443798-250528-571824942489132&time=1571651441&sign=FDTAXGERLQBHSKf-DCb740ccc5511e5e8fedcff06b081203-FFA8cEc5B5k%2BrjpzUNnP3YFmVUc%3D&to=d11&size=257415&sta_dx=257415&sta_cs=536&sta_ft=rar&sta_ct=7&sta_mt=7&fm2=MH%2CYangquan%2CAnywhere%2C%2Czhejiang%2Ccnc&ctime=1480492285&mtime=1480492285&resv0=cdnback&resv1=0&resv2=&resv3=&resv4=257415&vuk=2119336240&iv=0&htype=&randtype=&newver=1&newfm=1&secfm=1&flow_ver=3&pkey=en-5bb5273a92a673518b9592e9aacfc8b85a43abf896bfd998c53c12f594777c6d230f09398f426c034cca91fa8f01245e72cd9c07e86e37c1305a5e1275657320&sl=68616270&expires=8h&rt=sh&r=515599588&vbdid=3495373818&fin=vnc_82537_82537.rar&fn=vnc_82537_82537.rar&rtype=1&dp-logid=6812741667613920645&dp-callid=0.1&hps=1&tsl=200&csl=200&csign=6zDaKLjJFnzw3jAXKwtvq08JXtQ%3D&so=0&ut=6&uter=4&serv=0&uc=370421133&ti=54c943154d8629031741981d4efd601089cd4022ba64c95a&reqlabel=250528_f&by=themis)

打开vncviewer，连接虚拟机，端口为创建系统时指定的7000，可以看到图形化安装显示，安装系统。
注：配置ip，需要同一局域网，如果不同，需要设置路由记录，之后就可以使用ssh工具连接虚拟机了。

**查看vnc密码**

```bash
cd /etc/libvirt/qemu
vim 虚拟机配置文件
```

# 修改虚拟机root密码

```bash
# 安装编辑工具
yum -y install libguestfs-tools

# 在你的另外的机器上面，查看root的shadow文件，复制root的密码文件

# 关掉虚拟机
virsh xxxx shutdown  

# 编辑虚拟机shadow文件
virt-edit xxx /etc/shadow

# 启动虚拟机 
virsh xxxx start 
```

# 扩大虚拟机内存和cpu

**方法一**：

```bash
# 查看虚拟机信息
virsh dominfo etc01

# 关闭虚拟机
virsh shutdown etc01

# 方法一：临时修改
# 设置虚拟机最大内存
virsh setmaxmem etc01 8388608
# 开启虚拟机
virsh start etc01
# 设置虚拟机可以内存
virsh setmem etc01 8388608

# 方法二：永久修改，编辑配置文件
virsh edit etc01
```

**方法二**：

```bash
# 永久修改，编辑配置文件
virsh edit etc01
```

# 磁盘扩容

```bash
# 关闭虚拟机
virsh destroy test1
# 添加容量
qemu-img resize /data/kvm/disk/test1.qcow2 +100G
# 查看容量
qemu-img info /data/kvm/disk/test1.qcow2
# 启动虚拟机
virsh start test1
# 使用vnc登录
# 查看磁盘分区
fdisk -l
# 给多余容量分区，按n添加分区，按p添加主分区，按w保存推出，按t修改分区格式为8e
fdisk /dev/vda3
# 重新加载分区
partprobe
# 创建pv
pvcreate /dev/vda3
# 查看并添加pv搭配vg中
vgdisplay
vgextend centos /dev/vda3
# 查看并扩容已有分区
lvdisplay
lvresize -L +50G /dev/mapper/centos-root
# 扩充文件系统
resize2fs /dev/mapper/centos-root
```



# kvm常用命令

**命令帮助**

``` bash
virsh --help
```

**查看虚拟机状态**

``` bash
virsh list --all
```

**关机**

``` bash
virsh shutdown win2k8r2
```

**强制关闭电源**

``` bash
virsh destroy win2k8r2
```

**通过配置文件创建虚拟机**

``` bash
virsh create /etc/libvirt/qemu/win2k8r2.xml
```

**设置虚拟机开机自启**

``` bash
# 开机启动
virsh autostart win2k8r2

# 取消开机启动
virsh autostart --disable win2k8r2

# 查看虚拟机是否自启
ll /etc/libvirt/qemu/autostart/
```
**删除虚拟机**

``` bash
# 该命令只删除配置文件，并不删除磁盘文件
virsh undefine win2k8r2
```

**导出虚拟机配置文件**

``` bash
virsh dumpxml win2k8r2 > /etc/libvirt/qemu/win2k8r2_bak.xml
```

**通过配置文件恢复原KVM虚拟机**

``` bash
mv /etc/libvirt/qemu/win2k8r2_bak.xml /etc/libvirt/qemu/win2k8r2.xml
virsh define /etc/libvirt/qemu/win2k8r2.xml
```

**编辑配置文件**

``` bash
virsh edit win2k8r2
```

**挂起**

``` bash
virsh suspend win2k8r2
```

**恢复**

``` bash
virsh resume win2k8r2
```

