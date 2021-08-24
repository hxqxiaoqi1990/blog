---
title: registry 私有仓库配置
tags: [k8s,docker]
categories: kubernetes
date: 2020-05-19 17:00:00
---

# 介绍

Registry是一个几种存放image并对外提供上传下载以及一系列API的服务。可以很容易和本地源代码以及远端Git服务的关系相对应。

# 搭建

```bash
# 启动
# -e REGISTRY_STORAGE_DELETE_ENABLED=true 开启允许删除
# -v /data/registry:/var/lib/registry 持久化存储目录
docker run --name docker-registry -d -p 5000:5000 -e REGISTRY_STORAGE_DELETE_ENABLED=true -v /data/registry:/var/lib/registry registry
```

# 基本操作

```bash
# 查看所有镜像
curl -X GET http://<registry_ip>:<registry_port>/v2/_catalog

# 列出指定镜像的全部标签
curl -X GET http://<registry_ip>:<registry_port>/v2/<image_name>/tags/list

# 上传镜像，如果上传失败，需要配置/usr/lib/systemd/system/docker.service中添加私有仓库地址
# 例：
ExecStart=/usr/bin/dockerd --insecure-registry=http://192.168.40.102:5000

# 删除镜像，上面容器启动时已经允许删除
# 获取sha
sha=`ls /data/registry/docker/registry/v2/repositories/$image/_manifests/revisions/sha256`
# 根据sha删除镜像，但空间不会释放
curl -XDELETE http://<registryurl>/v2/$image/manifests/sha256:$sha
# 进入容器
docker exec -it registry sh
# 垃圾回收，释放空间
registry garbage-collect /etc/docker/registry/config.yml
# 推出容器，删除残留目录
rm -rf /opt/registry/docker/registry/v2/repositories/$image
```