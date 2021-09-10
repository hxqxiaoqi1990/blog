---
title: elasticstarch集群搭建
tags: [es]
categories: 数据库
date: 2020-01-22 9:00:00
---

# 介绍

针对一个索引，Elasticsearch 中其实有专门的衡量索引健康状况的标志，分为三个等级：

- green，绿色。这代表所有的主分片和副本分片都已分配。你的集群是 100% 可用的。
- yellow，黄色。所有的主分片已经分片了，但至少还有一个副本是缺失的。不会有数据丢失，所以搜索结果依然是完整的。不过，你的高可用性在某种程度上被弱化。如果更多的分片消失，你就会丢数据了。所以可把 yellow 想象成一个需要及时调查的警告。
- red，红色。至少一个主分片以及它的全部副本都在缺失中。这意味着你在缺少数据：搜索只能返回部分数据，而分配到这个分片上的写入请求会返回一个异常。

如果你只有一台主机的话，其实索引的健康状况也是 yellow，因为一台主机，集群没有其他的主机可以防止副本，所以说，这就是一个不健康的状态，因此集群也是十分有必要的。

# 搭建步骤

**主机环境：**

- jdk + es：192.168.40.100
- jdk + es：192.168.40.101
- jdk + es：192.168.40.102

**安装步骤：**

1. [参考elasticsearch安装](https://hxqxiaoqi.gitee.io/2019/08/08/elasticsearch-6.6.1安装/) 在两台主机上安装
2. 修改配置文件

**192.168.40.100配置文件**

```yaml
# ======================== Elasticsearch Configuration =========================
  # 集群名称
  cluster.name: mmh-es
  # 节点名称，不同节点命名不能一样
  node.name: node-1
# ----------------------------------- Memory -----------------------------------
  #因为centos6.x操作系统不支持SecComp，而elasticsearch5.5.2默认bootstrap.system_call_filter为true进行检测，所以导致检测失败，失败后直接导致ES不能启动。
  bootstrap.memory_lock: false
  bootstrap.system_call_filter: false
  
  # 当前节点Ip，不同节点配置不一样
  network.host: 192.168.40.100
  
  # 设置对外服务的http端口,默认为9200 
  http.port: 9200
  
  # 设置节点间交互的tcp端口,默认是9300 
  transport.tcp.port: 9300
  
  # 这是一个集群中的主节点的初始列表,当节点(主节点或者数据节点)启动时使用这个列表进行探测 
  discovery.zen.ping.unicast.hosts: ["192.168.40.100","192.168.40.101","192.168.40.102"]
  
  # 设置这个参数来保证集群中的节点可以知道其它N个有master资格的节点.默认为1,对于大的集群来说,可以设置大一点的值(2-4) 
  discovery.zen.minimum_master_nodes: 1
  
  #使用head等插件监控集群信息，需要打开以下配置项
  http.cors.enabled: true
  http.cors.allow-origin: "*"
```

**192.168.40.101配置文件**

其它与192.168.40.100配置一致

```yaml
node.name: node-2
network.host: 192.168.40.101
```

**192.168.40.102配置文件**

其它与192.168.40.100配置一致

```yaml
node.name: node-3
network.host: 192.168.40.102
```



# 安装head插件

由于head插件本质上还是一个nodejs的工程，因此需要安装node，使用npm来安装依赖的包。（npm可以理解为maven）

在192.168.40.100操作

**node安装**

```bash
# 下载nodejs最新的bin包
wget https://nodejs.org/dist/v9.3.0/node-v9.3.0-linux-x64.tar.xz　
# 解压
xz -d node-v9.3.0-linux-x64.tar.xz
tar -xf node-v9.3.0-linux-x64.tar

# 部署bin文件，先确定nodejs的bin路径
ln -s ~/node-v9.3.0-linux-x64/bin/node /usr/bin/node  
ln -s ~/node-v9.3.0-linux-x64/bin/npm /usr/bin/npm

# npm加速 全局安装cnpm 指定来源淘宝镜像
npm install -g cnpm --registry=https://registry.npm.taobao.org
```

**elasticsearch-head安装**

```bash
cd /usr/local/
git clone git://github.com/mobz/elasticsearch-head.git
cd elasticsearch-head
npm install
```

<font color='32CD32'>注：</font> 

1. 5.0以上，elasticsearch-head 不能放在elasticsearch的 plugins、modules 目录下，否则elasticsearch启动会报错。
2. 这里如果grunt没有安装成功也无所谓，可以通过其他方式启动elasticsearch-head插件（npm run start）。

**grunt安装**

```bash
# grunt是一个很方便的构建工具，可以进行打包压缩、测试、执行等等的工作，5.0里的head插件就是通过grunt启动的。
cd  /usr/local/elasticsearch-head
npm install -g grunt-cli  //执行后会生成node_modules文件夹
npm install
```

**修改配置**

```bash
cd  /usr/local/elasticsearch-head
vim _site/app.js
# 修改以下内容
```

```bash
# 这里的 localhost 是指进入elasticsearch-head页面时默认访问的ES集群地址，把他修改为其中一台ES节点的地址即可
this.base_uri = this.config.base_uri || this.prefs.get("app-base_uri") || "http://192.168.40.100:9200";
```

```bash
cd  /usr/local/elasticsearch-head
vim Gruntfile.js
# 修改以下内容
```

```js
        connect: {
            server: {
                options: {
                    port: 9100,
                    base: '.',
                    keepalive: true,
                    hostname: '*'
                }
            }
        }
```

**启动**

```bash
cd /usr/local/elasticsearch-head 
# 若想在后台运行，结尾追加“&”,也可以使用 npm run start启动
grunt server
```

**访问**

```bash
# 到浏览器访问
curl http://192.168.40.100:9100
```

**验证**

- 访问，有两个节点即成功
- head可以操作和查看es的所有索引

# 集群备份和恢复

备份es集群是使用`snapshot` API快照功能，需要一个共同的数据保存仓库，可以是：

- 共享文件系统，比如 NAS，NFS
- Amazon S3
- HDFS (Hadoop 分布式文件系统)
- Azure Cloud

这里，我们使用NFS做共享仓库，服务地址为：192.168.40.103，注意，需要集群外的`NFS`服务端提供目录共享，集群内搭建的`NFS`服务端会连接报错。

NFS 搭建

[NFS 搭建教程跳转](https://hxqxiaoqi.gitee.io/2019/06/05/nfs%20目录共享/)

创建es系统仓库

```yaml
PUT _snapshot/my_backup 
{
    "type": "fs", 
    "settings": {
        "location": "/mount/backups/my_backup" 
    }
}
```

- `_snapshot` es快照接口
- `my_backup` 快照仓库，自定义
- `/mount/backups/my_backup` NFS共享目录挂在地址，自定义，注意：共享文件系统路径必须确保集群所有节点都可以访问到。

快照备份

```yaml
# 备份索引打开索引
PUT _snapshot/my_backup/snapshot_1

# 备份单个索引
PUT _snapshot/my_backup/snapshot_2
{
    "indices": "index_1,index_2"
}
```

- `snapshot_1` 快照名称，自定义

查看快照

```yaml
GET _snapshot/my_backup/snapshot_2
```

删除快照

```
DELETE _snapshot/my_backup/snapshot_2
```

恢复快照

```yaml
# 恢复所有索引
POST _snapshot/my_backup/snapshot_1/_restore

# 恢复单个索引
POST /_snapshot/my_backup/snapshot_1/_restore
{
    "indices": "index_1", 
    "rename_pattern": "index_(.+)", 
    "rename_replacement": "restored_index_$1" 
}
```

1. 只恢复 `index_1` 索引，忽略快照中存在的其余索引。
2. 查找所提供的模式能匹配上的正在恢复的索引。
3. 然后把它们重命名成替代的模式。

这个会恢复 `index_1` 到你及群里，但是重命名成了 `restored_index_1` 。

# 配置集群密码

生成证书，之后把证书复制到其它节点的config目录下

```bash
bin/elasticsearch-certutil cert -out config/elastic-certificates.p12 -pass ""
mv elastic-certificates.p12 config/
```

添加配置`elasticsearch.yml` 

```yaml
http.cors.enabled: true
http.cors.allow-origin: "*"
http.cors.allow-headers: Authorization
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.verification_mode: certificate
xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
```

重启es，配置密码

```bash
# 手动配置密码
bin/elasticsearch-setup-passwords interactive

# 自动生成密码
bin/elasticsearch-setup-passwords auto
```

测试连接

```bash
curl --user elastic:123123 -XGET https://192.168.40.100:9200/_cat/indices?v
```

