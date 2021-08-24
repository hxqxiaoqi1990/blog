---
title: Archery-Sql审计平台
tags: [mysql]
categories: 数据库
date: 2021-08-13 15:00:00

---


# 介绍

Archery是[archer](https://github.com/jly8866/archer)的分支开源项目，定位于SQL审核查询平台，旨在提升DBA的工作效率，支持多数据库的SQL上线和查询，同时支持丰富的MySQL运维功能，所有功能都兼容手机端操作。

支持数据库：

- MySQL
- MsSQL
- Redis
- PgSQL
- Oracle
- MongoDB

# 部署

**下载**

```bash
wget https://github.com/hhyo/Archery/archive/refs/tags/v1.8.1.tar.gz
```

**部署**

```bash
# 解压
tar xf Archery-1.8.1.tar.gz
cd cd Archery-1.8.1/src/docker-compose

# 启动
docker-compose -f docker-compose.yml up -d

# 表结构初始化
docker exec -ti archery /bin/bash
cd /opt/archery
source /opt/venv4archery/bin/activate
python3 manage.py makemigrations sql  
python3 manage.py migrate 

# 数据初始化
python3 manage.py dbshell<sql/fixtures/auth_group.sql
python3 manage.py dbshell<src/init_sql/mysql_slow_query_review.sql

# 创建管理用户，输入账号密码
python3 manage.py createsuperuser

# 重启
docker restart archery

# 日志查看和问题排查
docker logs archery -f --tail=50
```

**访问**

```bash
curl http://127.0.0.1:9123
```

