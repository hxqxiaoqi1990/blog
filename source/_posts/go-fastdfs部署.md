---
title: go-fastdfs部署
tags: [go-fastdfs]
categories: 中间件
date: 2019-07-31 10:27:43
---

# go-fastdfs介绍
go-fastdfs是一个基于http协议的分布式文件系统，它基于大道至简的设计理念，一切从简设计，使得它的运维及扩展变得更加简单，它具有高性能、高可靠、无中心、免维护等优点。

下载地址：https://github.com/sjqzhang/go-fastdfs/releases
# 部署
## 单点部署
linux安装，下载已经编译的文件fileserver，执行以下命令：

``` bash
#赋予执行权限并运行，会在当前目录生成配置文件
chmod +x fileserver
./fileserver &
```
访问：服务器IP:8080
下载：上传时会返回文件地址，使用wget或curl即可下载

## 集群部署
注意，go-fastdfs不能在同一台上部署集群
以单点部署方式部署其它服务器
修改conf/cfg.json，例：

``` json
{
"PeerID": "集群内唯一,请使用0-9的单字符，默认自动生成",
"peer_id": "1",

"本主机地址": "本机http地址,默认自动生成(注意端口必须与addr中的端口一致），必段为内网，自动生成不为内网请自行修改，下同",
"host": "http://192.168.40.100:8080",

"集群": "集群列表,注意为了高可用，IP必须不能是同一个,同一不会自动备份，且不能为127.0.0.1,且必须为内网IP，默认自动生成",
"peers": ["http://192.168.40.100:8080","http://192.168.40.101:8080","http://192.168.40.102:8080"],
}
```
访问任意服务器的go-fastdfs服务，上传文件，其它服务器自动备份
## 备份数据
打包整个目录即可

# web后台管理部署
下载地址：https://github.com/perfree/go-fastdfs-web/releases
1.安装jdk环境
2.上传服务器，解压
3.启动：./goFastDfsWeb.sh start
4.访问：服务器IP:8088
5.注册用户