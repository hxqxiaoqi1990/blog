---
title: 局域网DNS搭建与使用
tags: [DNS]
categories: linux基础
date: 2019-08-01 16:57:55
---


# 介绍
1984年，加州大学伯克利分校的几个学生完成了Unix名称服务的实现，起名叫Berkeley Internet Name Domain（BIND）。目前，它是互联网上使用最为广泛的DNS服务软件

# 工作原理
（1）客户机首先查看查找本地hosts文件，如果有则返回，否则进行下一步
（2）客户机查看本地缓存，是否存在本条目的缓存，如果有则直接返回，不再向外发出请求，否则进行下一步，转发。
（3）将请求转发本地DNS服务器。
（4）查看域名是否本地解析，是则本地解析返回，否则进行下一步。
（5）本地DNS服务器首先在缓存中查找，有则返回，无则进行下一步。
（6）向全球13个根域服务器发起DNS请求，根域返回org域的地址列表。
（7）使用某一个org域的IP地址，发起DNS请求，org域返回kernel域服务器地址列表。
（8）使用某一个kernel域IP地址，发起DNS请求，kernel域返回www.kernel.org 主机的IP地址，本地DNS服务收到后，返回给客户机。

注：
	递归查询：压力在服务器端
	迭代查询：压力在客户端(直接返回结果到客户机)

# 搭建DNS
准备环境：
	DNS服务器：192.168.40.100
	客户机：192.168.40.101

搭建步骤：

 1. 安装dns服务bind
 2. 修改主配置文件
 3. 修改区域配置文件
 4. 创建数据配置文件
 5. 重启bind服务
 6. 修改访问域名的主机dns为DNS服务器ip
 7. 测试

## 安装dns服务bind
dns服务包名为bind，安装后的服务名为named
``` bash
yum -y install bind
```

## 修改主配置文件
/etc/named.conf：主配置文件用配置于DNS服务器的访问参数
``` bash
vim /etc/named.conf

options {
        #设置监听的端口和IP
		listen-on port 53 { 192.168.40.100; };
        listen-on-v6 port 53 { ::1; };
		
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
		
		#允许那些IP访问
        allow-query     { any; };
```
## 修改区域配置文件
/etc/named.rfc1912.zones：区域配置文件用于指定域名名称、配置文件名称

``` bash
vim /etc/named.rfc1912.zones

#填写域名
zone "12366xuetang.com" IN {
        type master;
		
		#填写数据配置文件名
        file "12366xuetang.front";
		
        allow-update { none; };
};
```

## 创建数据配置文件
数据配置文件：根据区域配置文件指定的名称创建，并配置域名解析

``` bash
#到dns数据目录下
cd /var/named/

#创建数据文件
cp -a named.localhost 12366xuetang.front

#打开数据配置文件
vim 12366xuetang.front

#配置举例，域名后面的“.”别忘记
$TTL 1D
@       IN SOA  12366xuetang.com. rname.invalid. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum
        NS      js.12366xuetang.com.
js      A       192.168.40.100
zj      A       192.168.40.101
```
## 重启bind服务
如果报错：请仔细查看数据配置文件设置
``` bash
systemctl restart named
```

## 修改访问域名的主机dns为DNS服务器ip
注：所有使用域名访问的客户机都需要配置DNS服务器IP

``` bash
vim /etc/sysconfig/network-scripts/ifcfg-ens33
TYPE=Ethernet
BOOTPROTO=static
NAME=ens33
DEVICE=ens33
ONBOOT=yes
IPADDR=192.168.40.100
NETMASK=255.255.255.0
GATEWAY=192.168.40.2
DNS1=192.168.40.100
```

## 测试

``` bash
#安装nslookup工具
yum -y install bind-utils

#访问域名
nslookup zj.12366xuetang.com

#显示Address为数据配置文件解析的ip则成功
Server:		192.168.40.100
Address:	192.168.40.100#53

Name:	zj.12366xuetang.com
Address: 192.168.40.101
```