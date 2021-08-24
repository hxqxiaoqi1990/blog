---
title: ansible基本配置
tags: [ansible]
categories: linux基础
date: 2019-09-09 20:00:00
---


# 介绍
官方的title是“Ansible is Simple IT Automation”——简单的自动化IT工具。这个工具的目标有这么几项：让我们自动化部署APP；自动化管理配置项；自动化的持续交付；自动化的（AWS）云服务管理。

所有的这几个目标本质上来说都是在一个台或者几台服务器上，执行一系列的命令而已,批量的在远程服务器上执行命令 。

ansible是一个轻量级的运维自动化配置管理和配置工具，基于Python研发。集合了众多老牌运维工具（puppet、cfengine、chef、func、fabric）的优点，实现了批量系统配置、批量程序部署、批量运行命令等功能

# 安装

``` bash
yum -y install ansible
```
目录说明

 1. ansible.cfg：配置文件
 2. hosts：设置被管理服务器IP
 3. roles：ansible-playbook存放位置

ansible.cfg配置文件中配置
``` bash
#跳过第一次连接检测询问是否登陆的提示（YES/NO）
host_key_checking=False       
```

# host配置

``` bash
#远程主机登陆端口
ansible_ssh_port=22

#远程主机登陆用户名
ansible_ssh_user=root

#远程主机登陆用户名的密码
ansible_ssh_pass=

 # sudo密码
ansible_sudo_pass
```
## 设置主机组
使用ansible工具时，指定web组，相当于控制web组下的主机
``` bash
[web]
192.168.1.101
192.168.1.100
```
## 设置主机地址段
相当于一个IP段或域名段的主机
``` bash
[web]
192.168.1.[2:100]
www.test[a:z].com
```
## 指定主机用户、密码、端口
使用ansible工具时，自动使用该账户登录操作
如果想不使用账号密码操作，需要提前设置 [ssh免密设置](https://hxqxiaoqi.gitee.io/2019/06/04/linux%20ssh%E5%85%8D%E5%AF%86%E7%99%BB%E5%BD%95%E8%AE%BE%E7%BD%AE/)
``` bash
[web]
192.168.1.100 ansible_ssh_user=root ansible_ssh_pass=123 ansible_ssh_port=22
```

## 设置主机组变量
等于所有主机都已经指定用户、密码、端口

``` bash
[web]
192.168.1.100
192.168.1.101
192.168.1.102

[web:vars]
ansible_ssh_user=root 
ansible_ssh_pass=123 
ansible_ssh_port=22
```

## 主机组继承
类似于web、web1是weball的子集，使用weball组，包含web、web1组
``` bash
[web]
192.168.182.101
[web1]
192.168.182.100

[weball:children]
web1
web
```

# 模块

``` bash
命令格式：ansible [主机/主机组] [-m 模块] [-a args]

# 列出所有安装模块（q退出）
ansible-doc -l 

# 列出yum模块描述信息和操作动作
ansible-doc -s yum 

```


## command模块
这个模块可以直接在远程主机上执行命令，并将结果返回本主机。注意，该命令不支持 | 管道命令

``` bash
# 查询date
ansible all -m command -a 'date' 

# 如果不加-m模块，默认运行command模块
ansible all -a 'ls /' 
```

## cron模块
该模块适用于管理cron计划任务的。

<font color='98F5FF'>minute参数</font>：此参数用于设置计划任务中分钟设定位的值，比如，上述示例1中分钟设定位的值为5，即 minute=5，当不使用此参数时，分钟设定位的值默认为”*”。

<font color='98F5FF'>hour参数</font>：此参数用于设置计划任务中小时设定位的值，比如，上述示例1中小时设定位的值为1，即 hour=1，当不使用此参数时，小时设定位的值默认为”*”。

<font color='98F5FF'>day参数</font>：此参数用于设置计划任务中日设定位的值，当不使用此参数时，日设定位的值默认为”*”。

<font color='98F5FF'>month参数</font>：此参数用于设置计划任务中月设定位的值，当不使用此参数时，月设定位的值默认为”*”。

<font color='98F5FF'>weekday参数</font>：此参数用于设置计划任务中周几设定位的值，当不使用此参数时，周几设定位的值默认为”*”。

<font color='98F5FF'>job参数</font>：此参数用于指定计划的任务中需要实际执行的命令或者脚本，比如上例中的 “echo test” 命令。

<font color='98F5FF'>name参数</font>：此参数用于设置计划任务的名称，计划任务的名称会在注释中显示

<font color='98F5FF'>state参数</font>：当计划任务有名称时，我们可以根据名称修改或删除对应的任务，当删除计划任务时，需要将 state 的值设置为 absent。

``` bash
#-a：指定添加参数*/1：每分钟执行job：执行内容      
ansible all -m cron -a 'minute="*/1" job="/usr/bin/echo heihei >> /opt/test.txt" name="test cron"'   

# 在all主机上创建计划任务，任务名称为”crontab day test”，任务每3天执行一次，于执行当天的1点1分开始执行，任务内容为输出 test 字符。
ansible all -m cron -a "name='crontab day test' minute=1 hour=1 day=*/3 job='echo test'
```
