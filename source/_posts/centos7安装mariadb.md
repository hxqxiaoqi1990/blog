---
title: centos7安装mariadb
tags: [mysql]
categories: 数据库
date: 2019-10-23 11:11:11
---


# 介绍
使用yum安装mariadb，方便快捷。

# 安装
**安装并启动**

``` bash
yum install mariadb-server

systemctl start mariadb 

systemctl enable mariadb
```
**初始化**

``` bash
# 初始化命令，进入交互式界面
mysql_secure_installation

# 输入数据库超级管理员root的密码(注意不是系统root的密码)，第一次进入还没有设置密码则直接回车
Enter current password for root (enter for none):  

# 设置密码，y
Set root password? [Y/n]  

# 新密码
New password:  
# 再次输入密码
Re-enter new password:  

# 移除匿名用户， y
Remove anonymous users? [Y/n]  

# 拒绝root远程登录，n，不管y/n，都会拒绝root远程登录
Disallow root login remotely? [Y/n]  

# 删除test数据库，y：删除。n：不删除，数据库中会有一个test数据库，一般不需要
Remove test database and access to it? [Y/n]  

# 重新加载权限表，y。或者重启服务也许
Reload privilege tables now? [Y/n]  
```

**登录**

``` bash
mysql -u root -p
```

**修改字符集**
vim /etc/my.cnf
在  [mysqld]  标签下添加
``` bash
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
```
vim /etc/my.cnf.d/client.cnf
在  [client]  标签下添加
``` bash
default-character-set=utf8
```
vim /etc/my.cnf.d/mysql-clients.cnf
在  [mysql]  标签下添加
``` bash
default-character-set=utf8
```
**重启**
``` bash
systemctl restart mariadb
```
**修改远程登录权限**

``` bash
MariaDB [(none)]> grant all privileges on *.* to 'root'@'%' identified by '123456Aa';

MariaDB [(none)]> flush privileges;
```

