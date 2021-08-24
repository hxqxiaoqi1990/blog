---
title: docker修改存储目录和连接私库
tags: [docker]
categories: kubernetes
date: 2020-05-14 18:27:43
---

两种方式：

方式一：

vim /etc/docker/daemon.json

```json
{
  "registry-mirrors": [
     "https://bxsfpjcb.mirror.aliyuncs.com"
  ],
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
    },
# 修改私库地址
  "insecure-registries":
        ["127.0.0.1","192.168.10.121:30000","192.168.10.122:30000","192.168.10.123:30000"],
# 修改存储目录
  "data-root":"/data/docker"
}
```

注：

- 修改存储目录后，cp原docker目录到指定的目录中，重启daemon-reload和docker

方式二：

vim /usr/lib/systemd/system/docker.service

```bash
ExecStart=/usr/bin/dockerd --graph /data/docker --insecure-registry=http://192.168.10.121:30000 -H fd:// --containerd=/run/containerd/containerd.sock
```

