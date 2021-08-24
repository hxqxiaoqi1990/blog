---
title: etcd集群
tags: [etcd]
categories: kubernetes
date: 2021-06-28 16:00:00
---

# 介绍

`ETCD`是用于共享配置和服务发现的分布式，一致性的KV存储系统。作为一个受到ZooKeeper与doozer启发而催生的项目，除了拥有与之类似的功能外，更专注于以下四点。

1. 简单：基于HTTP+JSON的API让你用curl就可以轻松使用。
2. 安全：可选SSL客户认证机制。
3. 快速：每个实例每秒支持一千次写操作。
4. 可信：使用Raft算法充分实现了分布式。

应用场景有如下几类： 

- 服务发现（Service Discovery） 
- 消息发布与订阅 
- 负载均衡 
- 分布式通知与协调 
- 分布式锁、分布式队列 
- 集群监控与Leader竞选

# 部署

## 安装cfssl

cfssl是证书生成和签发工具

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
mv cfssl_linux-amd64 /usr/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssljson_linux-amd64 /usr/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /usr/bin/cfssl-certinfo
```

## 制作etcd证书

搭建环境，三台虚拟机，192.168.40.100，192.168.40.101，192.168.40.102

在所有虚拟机上执行

```bash
mkdir /etc/etcd/ssl -p
```

在192.168.40.100机器执行：

```bash
# 下载
curl -L https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz -o /root/etcd-v3.5.0-linux-amd64.tar.gz

# 解压安装
tar xf /root/etcd-v3.5.0-linux-amd64.tar.gz
cd /root/etcd-v3.5.0-linux-amd64

cp etcd etcdctl etcdutl /usr/bin/
scp etcd etcdctl etcdutl 192.168.40.101:/usr/bin/
scp etcd etcdctl etcdutl 192.168.40.102:/usr/bin/

mkdir /root/kubernetes/resources/cert/etcd -p
cd /root/kubernetes/resources/cert/etcd
```

编辑ca-config.json

```text
vim ca-config.json
```

写入文件内容如下：

```json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "etcd": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
```

编辑ca-csr.json：

```text
vim ca-csr.json
```

写入文件内容如下：

```json
{
    "CN": "etcd ca",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Hunan",
            "ST": "Changsha"
        }
    ]
}
```

生成ca证书和密钥：

```text
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

编辑server-csr.json：

```text
vim server-csr.json
```

写入文件内容如下：

```json
{
    "CN": "etcd",
    "hosts": [
        "192.168.40.100",
        "192.168.40.101",
        "192.168.40.102"
        ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Hunan",
            "ST": "Changsha"
        }
    ]
}
```

> hosts中配置所有Master和Node的IP列表。

生成etcd证书和密钥

```text
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd server-csr.json | cfssljson -bare server
# 此时目录下会生成7个文件
ls
ca-config.json  ca.csr  ca-csr.json  ca-key.pem  ca.pem  server.csr  server-csr.json  server-key.pem  server.pem
```

拷贝证书

```text
cp ca.pem server-key.pem  server.pem /etc/etcd/ssl
scp ca.pem server-key.pem  server.pem 192.168.40.101:/etc/etcd/ssl
scp ca.pem server-key.pem  server.pem 192.168.40.102:/etc/etcd/ssl
```

## 配置etcd

这里开始命令需要分别在Master和Node机器执行，配置etcd.conf

```text
vim /etc/etcd/etcd.conf
```

k8s-master01写入文件内容如下：

```ini
[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.40.100:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.40.100:2379,https://127.0.0.1:2379"

[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.40.100:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.40.100:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.40.100:2380,etcd02=https://192.168.40.101:2380,etcd03=https://192.168.40.102:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

k8s-node01写入文件内容如下：

```ini
[Member]
ETCD_NAME="etcd02"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.40.101:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.40.101:2379,https://127.0.0.1:2379"

[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.40.101:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.40.101:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.40.100:2380,etcd02=https://192.168.40.101:2380,etcd03=https://192.168.40.102:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

k8s-node02写入文件内容如下：

```ini
[Member]
ETCD_NAME="etcd03"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://192.168.40.102:2380"
ETCD_LISTEN_CLIENT_URLS="https://192.168.40.102:2379,https://127.0.0.1:2379"

[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://192.168.40.102:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://192.168.40.102:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://192.168.40.100:2380,etcd02=https://192.168.40.101:2380,etcd03=https://192.168.40.102:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

这里开始在所有机器执行，设置etcd服务配置文件

```bash
mkdir -p /var/lib/etcd
vim /usr/lib/systemd/system/etcd.service
```

执行上行命令，写入文件内容如下：

```ini
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
        --cert-file=/etc/etcd/ssl/server.pem \
        --key-file=/etc/etcd/ssl/server-key.pem \
        --peer-cert-file=/etc/etcd/ssl/server.pem \
        --peer-key-file=/etc/etcd/ssl/server-key.pem \
        --trusted-ca-file=/etc/etcd/ssl/ca.pem \
        --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

> etcd3.4版本会自动EnvironmentFile文件中的环境变量，不需要再ExecStart的命令参数重复设置，否则会报："xxx" is shadowed by corresponding command-line flag的错误信息。

启动etcd，并且设置开机自动运行etcd

```bash
systemctl daemon-reload
systemctl start etcd.service
systemctl enable etcd.service
```

检查etcd集群的健康状态

```bash
etcdctl endpoint health --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379,https://192.168.40.101:2379,https://192.168.40.102:2379"
```

输出如下，说明etcd集群已经部署成功。

```bash
https://192.168.40.102:2379 is healthy: successfully committed proposal: took = 15.805605ms
https://192.168.40.101:2379 is healthy: successfully committed proposal: took = 22.127986ms
https://192.168.40.100:2379 is healthy: successfully committed proposal: took = 24.829669ms
```

etcd集群搭建完毕，如果不需要集群ssl证书认证，只需修改`etcd.conf`文件，把所有地址改为http，再修改`etcd.service`，去除证书相关参数即可。

## 集群备份

etcd3版本，备份只需要单节点执行，恢复需要在每个节点上执行恢复操作，恢复时需要停止集群，最好把原来的etcd目录备份后删除，否则报错目录已存在。

```bash
# 备份集群
etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379" snapshot save snapshot.db

# 恢复集群

# 192.168.40.100 机器上操作

ETCDCTL_API=3 etcdctl snapshot restore /mnt/snapshot.db \
  --name etcd01 \
  --initial-cluster "etcd01=https://192.168.40.100:2380,etcd02=https://192.168.40.101:2380,etcd03=https://192.168.40.102:2380" \
  --initial-cluster-token etcd-cluster \
  --initial-advertise-peer-urls https://192.168.40.100:2380 \
  --data-dir=/var/lib/etcd/default.etcd

# 192.168.40.101 机器上操作

ETCDCTL_API=3 etcdctl snapshot restore /mnt/snapshot.db \
  --name etcd02 \
  --initial-cluster "etcd01=https://192.168.40.100:2380,etcd02=https://192.168.40.101:2380,etcd03=https://192.168.40.102:2380"  \
  --initial-cluster-token etcd-cluster \
  --initial-advertise-peer-urls https://192.168.40.101:2380 \
  --data-dir=/var/lib/etcd/default.etcd

# 192.168.40.102 机器上操作

ETCDCTL_API=3 etcdctl snapshot restore /mnt/snapshot.db \
  --name etcd03 \
  --initial-cluster "etcd01=https://192.168.40.100:2380,etcd02=https://192.168.40.101:2380,etcd03=https://192.168.40.102:2380"  \
  --initial-cluster-token etcd-cluster \
  --initial-advertise-peer-urls https://192.168.40.102:2380 \
  --data-dir=/var/lib/etcd/default.etcd
```

## 添加节点

```bash
# 查看节点列表，输出ID
etcdctl member list --write-out=table
# 根据ID删除节点
etcdctl member remove 297faf509a025173
# 添加节点，状态为unstarted，如果新节点有数据，需删除数据目录，配置文件从其它节点拷贝，添加新节点配置，ETCD_INITIAL_CLUSTER_STATE配置为existing，之后重启新节点etcd即可
etcdctl member add etcd03 --peer-urls=http://192.168.40.102:2380
```

## 常用命令

```bash
# 得到所有的key
etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379,https://192.168.40.101:2379,https://192.168.40.102:2379" --prefix --keys-only=true get /

# 添加key
etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379,https://192.168.40.101:2379,https://192.168.40.102:2379" put /school/class/name helios

# 删除key
etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379,https://192.168.40.101:2379,https://192.168.40.102:2379" del /foo/bar

# 查看集群健康状态
etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379,https://192.168.40.101:2379,https://192.168.40.102:2379" endpoint health 

# 查看集群状态
etcdctl --write-out=table --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379,https://192.168.40.101:2379,https://192.168.40.102:2379" endpoint status

# 切换leader
# 先看状态
etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379,https://192.168.40.101:2379,https://192.168.40.102:2379" endpoint --cluster=true status  -w table
# move-leader
etcdctl --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/server.pem --key=/etc/etcd/ssl/server-key.pem --endpoints="https://192.168.40.100:2379,https://192.168.40.101:2379,https://192.168.40.102:2379" move-leader d6414a7c7c550d29
```

