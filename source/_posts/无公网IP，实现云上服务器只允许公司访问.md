---
title: 无公网IP，实现云上服务器只允许公司访问
tags: [shell脚本]
categories: 脚本
date: 2019-06-04 16:40:43
---


# 实验效果
随着云服务的普及，现在小公司大都没有购买公网IP，这对于运维人员来说，无法限制ssh远程登录的IP是一个很不爽的事，这篇博客就是为了实现公司无公网IP，也可现在ssh连接IP方法。

# 实验思路

 1. 注册动态域名
 2. 把动态域名绑定到公司路由器的DDNS服务上
 3. 在服务器上，使用shell脚本定时解析动态域名，获取公司IP，并与服务器白名单匹配
 4. 配置定时任务

## 1. 注册动态域名
登录花生壳官网：https://b.oray.com/ 注册账号，申请动态域名（例：2awfes953.51mypc.cn）
这里就不作详细解释。

## 2. 域名绑定路由器

 1. 登录公司主路由器
 2. 选择：虚拟服务--DDNS服务
 3. 开启DDNS，选择服务商oray.com，输入花生壳注册的用户名和密码，确认。

## 3. shell脚本配置

``` bash
#!/bin/bash
#定时获取公网IP脚本
#如果没有nslookup命令，请安装：yum -y install bind-utils

#获取公司公网IP
ip1=`nslookup 2awfes953.51mypc.cn|grep Address|awk -F ":" 'NR==2{print $2}'|cut -d " " -f2`

#获取白名单下的ssh允许的IP，如果原先没有，请先设置公司当前公网IP
ip2=`grep sshd /etc/hosts.allow|awk -F ":" 'NR==1{print $2}'`

#匹配获取的IP与白名单的IP是否一致，不一致就修改白名单IP为当前获取的公司IP
if [ $ip1 != $ip2 ]:then
	sed -i "s/sshd:$ip2:allow/sshd:$ip1:allow/g" /etc/hosts.allow
fi
```

## 4. 定时任务设置

``` bash
#一小时匹配一次
0 */1 * * * /bin/bash /data/ssh-rule.sh &> /dev/null
```