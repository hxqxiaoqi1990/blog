---
title: mysql8.0 安装
tags: [mysql]
categories: 数据库
date: 2020-04-02 10:49:00
---

# 介绍

mysql8 安装，复制以下脚本执行即可

# 脚本

```bash
#!/bin/bash
# 安装mysql8.0
rpm -qa|grep mysql|awk '{print "yum -y remove\t" $1}'|sh
rpm -qa|grep mariadb|awk '{print "yum -y remove\t" $1}'|sh
rpm -Uvh http://dev.mysql.com/get/mysql57-community-release-el7-9.noarch.rpm 
sed -i "34s/enabled=1/enabled=0/g" /etc/yum.repos.d/mysql-community.repo
sed -i "41s/enabled=0/enabled=1/g" /etc/yum.repos.d/mysql-community.repo
yum repolist enabled | grep mysql
yum install -y mysql-community-server
systemctl start mysqld
systemctl enable mysqld
systemctl daemon-reload

dbpw=`grep 'temporary password' /var/log/mysqld.log|cut -d ' ' -f 13`
newdbpw="123456Aa"
mysql --connect-expired-password -uroot -p$dbpw << EOF
set global validate_password.policy=0;
set global validate_password.length=1;
ALTER USER 'root'@'localhost' IDENTIFIED BY ${newdbpw};
create user 'root'@'%' identified by ${newdbpw};
grant all privileges on *.* to root@'%' with grant option;
flush privileges;
quit
EOF
```

my.cnf 默认配置文件

```bash
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid

#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
```

