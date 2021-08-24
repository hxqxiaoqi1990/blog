---
title: oss安装
tags: [阿里云]
categories: linux基础
date: 2020-01-23 17:00:00
---

# 介绍

oss是阿里云存储服务，可以使用客户端或命令行工具上传下载文件。

# 使用

下载

```bash
wget http://gosspublic.alicdn.com/ossutil/1.6.10/ossutil64?spm=a2c4g.11186623.2.12.3467448aqUzNnH
mv ossutil64?spm=a2c4g.11186623.2.12.3467448aqUzNnH ossutil64
chmod 755 ossutil64
```

注册

```bash
# 配置信息
ossutil64 config

# 创建的bucket上可以看到endpoint信息
endpoint=oss-cn-hangzhou-fawefwae.aliyuncs.com
# 登录账号设置
accessKeyID=efwaefaw
accessKeySecret=efawefawefwae
```

上传

```bash
ossutil64 cp delete_promoter_memcache.py oss://aliyun-mysql-storage
```

下载

```bash
/usr/local/bin/ossutil64 cp oss://aliyun-mysql-storage/test.sql.gz .
```

