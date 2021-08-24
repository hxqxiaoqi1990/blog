---
title: postgre常用命令 
tags: [postgre]
categories: 数据库
date: 2021-05-20 19:49:00
---

# 介绍

常用命令

```bash
# 登录
psql -U postgres
# 查询库
postgres=# \l
# 切换库
postgres=# \c test
# 查询所有表
postgres=# \dt
# 创建库
postgres=# create database test
# 删除库
postgres=# drop database test
# 删除表
postgres=# drop table tablename
```

备份恢复

```bash
# 备份
pg_dump -h localhost -U postgres databasename > databasename.bak
# 恢复
psql -h localhost -U postgres -d databasename < databasename.bak
```



