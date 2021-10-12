---
title: rabbitmq集群部署
date: 2021-08-27 16:23:22
tags: [rabbitmq]
categories: 中间件
---

# 介绍

**RabbitMQ集群分为两种模式：**

- 普通模式：创建好RabbitMQ之后的默认模式。
- 镜像模式：把需要的队列做成镜像队列。

**普通模式**

queue 创建之后，如果没有其它 policy，消息实体只存在于其中一个节点， A、 B 两个 Rabbitmq 节点仅有相同的元数据，即队列结构，但队列的数据仅保存有一份，即创建该队列的 rabbitmq 节点（A 节点），当消息进入 A 节点的 Queue 中后， consumer 从 B 节点拉取时， RabbitMQ 会临时在 A、 B 间进行消息传输，把 A 中的消息实体取出并经过 B 发送给 consumer，所以 consumer 可以连接每一个节点，从中取消息，该模式存在一个问题就是当 A 节点故障后，B节点无法取到 A 节点中还未消费的消息实体。

**镜像模式**

把需要的队列做成镜像队列，存在于多个节点，属于RabbitMQ的HA方案（镜像模式是在普通模式的基础上，增加一些镜像策略）该模式解决了上述问题，其实质和普通模式不同之处在于，消息实体会主动在镜像节点间同步，而不是在consumer取数据时临时拉取。该模式带来的副作用也很明显，除了降低系统性能外，如果镜像队列数量过多，加之大量的消息进入，集群内部的网络带宽将会被这种同步通讯大大消耗掉。所以在对可靠性要求较高的场合中适用，一个队列想做成镜像队列，需要先设置 policy，然后客户端创建队列的时候， rabbitmq 集群根据“队列名称”自动设置是普通集群模式或镜像队列。

**RabbitMQ集群中有两种节点类型**

- 内存节点：只将数据保存到内存。
- 磁盘节点：保存数据到内存和磁盘。
  内存节点虽然不写入磁盘，但是它执行效率比磁盘节点要好，集群中，只需要一个磁盘节点来保存数据就足够了，如果集群中只有内存节点，那么，不能全部停止它们，否则所有数据消息在服务器全部停机之后都会丢失。

# 部署

## 实验环境

| IP             | 主机名           | 角色     | rabbitmq版本 | erlang版本 |
| -------------- | ---------------- | -------- | ------------ | ---------- |
| 192.168.40.100 | rabbitmq-server1 | 内存节点 | 3.7.8        | 19.3       |
| 192.168.40.101 | rabbitmq-server2 | 内存节点 | 3.7.8        | 19.3       |
| 192.168.40.102 | rabbitmq-server3 | 磁盘节点 | 3.7.8        | 19.3       |

1. 禁用防火墙和selinux
2. 时间同步
3. 设置主机名与hosts解析

## 普通模式

**下载**

```bash
# 下载rabbitmq
wget wget https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.7.8/rabbitmq-server-3.7.8-1.el6.noarch.rpm

# 下载erlang
wget https://github.com/rabbitmq/erlang-rpm/releases/download/v19.3.6.13/erlang-19.3.6.13-1.el7.centos.x86_64.rpm
```

**安装，其中一个节点执行**

```bash
yum -y install erlang-19.3.6.13-1.el7.centos.x86_64.rpm rabbitmq-server-3.7.8-1.el6.noarch.rpm

# 启动并开机启动
systemctl enable --now rabbitmq-server
```

**同步集群文件**

Rabbitmq 的集群是依赖于 erlang 的集群来工作的，所以必须先构建起 erlang 的集群环境,而 Erlang 的集群中各节点是通过一个 magic cookie 来实现的，这个cookie 存放在 /var/lib/rabbitmq/.erlang.cookie 中，文件是 400 的权限,用户权限默认是rabbitmq,所以必须保证各节点 cookie 保持一致，否则节点之间就无法通信。

```bash
systemctl stop rabbitmq-server
scp /var/lib/rabbitmq/.erlang.cookie 192.168.40.101:/var/lib/rabbitmq/
scp /var/lib/rabbitmq/.erlang.cookie 192.168.40.102:/var/lib/rabbitmq/
ssh 192.168.40.101 "chown -R rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie"
ssh 192.168.40.102 "chown -R rabbitmq:rabbitmq /var/lib/rabbitmq/.erlang.cookie"
systemctl start rabbitmq-server
```

**启动集群**

所有节点执行

```bash
systemctl start rabbitmq-server
```

**添加集群**

100和101执行

```bash
# 停止app服务
rabbitmqctl stop_app 

# 清空元数据
rabbitmqctl reset

# 将 rabbitmq-server1 添加到集群当中，并成为内存节点，不加--ram 默认是磁盘节点
rabbitmqctl join_cluster rabbit@rabbitmq-server3 --ram

# 启动app服务
rabbitmqctl start_app 
```

**查看集群状态**

```bash
rabbitmqctl cluster_status
```

## 镜像模式

其中一台节点执行

```bash
rabbitmqctl set_policy ha-all "#" '{"ha-mode":"all"}'
```

## 启动web管理

**启动**

```bash
# 启动web插件后会监听15672端口，该端口是web管理端口
rabbitmq-plugins enable rabbitmq_management
```

**登录用户**

3.3以后guest用户无法登录，需创建用户管理

```bash
# 创建用户
rabbitmqctl add_user admin 123456
# 设置权限
rabbitmqctl set_user_tags admin administrator
```

**访问**

```bash
http://192.168.40.100:15672
```

## 常用命令

```bash
# 创建vhost
rabbitmqctl add_vhost hechunping

# 列出所有vhost
rabbitmqctl list_vhosts

# 列出所有队列
rabbitmqctl list_queues 

# 删除指定vhost
rabbitmqctl delete_vhost hechunping
Deleting vhost "hechunping" ...

# 添加账户hechunping密码为123.com
rabbitmqctl add_user hechunping 123.com

# 更改用户hechunping密码
rabbitmqctl change_password hechunping 123456

# 查看集群状态
rabbitmqctl cluster_status

# 查看所有用户
rabbitmqctl list_users

# 删除用户
rabbitmqctl delete_user Username

# 设置 hechunping 用户对 hechunping 的 vhost 有读写权限，三个点分别表示配置正则、读和写
rabbitmqctl set_permissions -p hechunping hechunping ".*" ".*" ".*"

# 查看(指定hostpath)所有用户的权限信息
rabbitmqctl  list_permissions  [-p  VHostPath]

# 查看指定用户的权限信息
rabbitmqctl  list_user_permissions  User

# 清除用户的权限信息
rabbitmqctl  clear_permissions  [-p VHostPath]  User
```
