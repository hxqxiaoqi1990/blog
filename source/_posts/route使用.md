---
title: route使用
tags: [route]
categories: linux基础
date: 2020-02-28 15:00:00
---

# linux:

<font color='32CD32'>主机路由</font>

主机路由是路由选择表中指向单个IP地址或主机名的路由记录。主机路由的Flags字段为H。例如，在下面的示例中，本地主机通过IP地址192.168.1.1的路由器到达IP地址为10.0.0.10的主机。

```bash
# 添加
route add -host 192.168.1.2 dev eth0   
route add -host 10.20.30.148 gw 10.20.30.40

# 删除
route del -host 192.168.1.2 dev eth0   
route del -host 10.20.30.148 gw 10.20.30.40
```

<font color='32CD32'>网络路由</font>

网络路由是代表主机可以到达的网络。网络路由的Flags字段为N。例如，在下面的示例中，本地主机将发送到网络192.19.12的数据包转发到IP地址为192.168.1.1的路由器。

```bash
# 添加
route add -net 100.96.174.0/24 gw 192.168.10.126

# 删除
route del -net 100.96.174.0/24 gw 192.168.10.126
```

<font color='32CD32'>默认路由</font>

当主机不能在路由表中查找到目标主机的IP地址或网络路由时，数据包就被发送到默认路由（默认网关）上。默认路由的Flags字段为G。例如，在下面的示例中，默认路由是IP地址为192.168.1.1的路由器。

```bash
# 添加
route add default gw 192.168.1.1  

# 删除
route del default gw 192.168.1.1  
```

# windows:

<font color='32CD32'>网络路由</font>

```bash
# 添加
route add 100.117.144.0/24 192.168.10.124

# 删除
route del 100.117.144.0/24 192.168.10.124
```