---
title: ELK之log-pilot部署
tags: [elk]
categories: 监控
date: 2021-08-13 16:00:00
---

# 介绍

本文档介绍一款新的 Docker 日志收集工具：log-pilot。log-pilot 是阿里提供的日志收集镜像。可以在每台机器上部署一个 log-pilot 实例，就可以收集机器上所有 Docker 应用日志。(注意：只支持Linux版本的Docker，不支持Windows/Mac版)

log-pilot 具有如下特性：

- 一个单独的 log 进程收集机器上所有容器的日志。不需要为每个容器启动一个 log 进程。
- 支持文件日志和 stdout。docker log dirver 亦或 logspout 只能处理 stdout，log-pilot 不仅支持收集 stdout 日志，还可以收集文件日志。
- 声明式配置。当您的容器有日志要收集，只要通过 label 声明要收集的日志文件的路径，无需改动其他任何配置，log-pilot 就会自动收集新容器的日志。
- 支持多种日志存储方式。无论是强大的阿里云日志服务，还是比较流行的 elasticsearch 组合，甚至是 graylog，log-pilot 都能把日志投递到正确的地点。
- 开源。log-pilot 完全开源，您可以从 [Git项目地址](https://github.com/AliyunContainerService/log-pilot) 下载代码。如果现有的功能不能满足您的需要，欢迎提 issue。

# log-pilot部署

```yaml
version: '2.2'
services:
  log-pilot:
    container_name: log-pilot
    image: registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.5-filebeat
    restart: always
    environment:
      - LOGGING_OUTPUT=elasticsearch
      - ELASTICSEARCH_HOST=192.168.40.100
      - ELASTICSEARCH_PORT=9200
      - ELASTICSEARCH_USER=elastic
      - ELASTICSEARCH_PASSWORD=123456
    privileged: true
    volumes:
      - /etc/localtime:/etc/localtime
      - /var/run/docker.sock:/var/run/docker.sock
      - /:/host:ro
```

## ES+Kibana部署

```yaml
version: '2.2'
services:
  elasticsearch:
    image: elasticsearch:6.8.2
    container_name: elasticsearch
    hostname: elasticsearch
    restart: always
    environment:
      - node.name=es01
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - xpack.security.enabled=true
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    mem_limit: 2048m
    volumes:
      - /etc/localtime:/etc/localtime
      - /Data/elasticsearch/data:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
  kibana:
    container_name: kibana
    image: kibana:6.8.2
    restart: always
    environment:
      ELASTICSEARCH_HOSTS: http://192.168.40.100:9200
      ELASTICSEARCH_USERNAME: "kibana"
      ELASTICSEARCH_PASSWORD: '"123456"'
      I18N_LOCALE: zh-CN
    mem_limit: 1024m
    volumes:
      - /etc/localtime:/etc/localtime
    ports:
      - 5601:5601
    depends_on:
      - elasticsearch
```

## ES容器设置密码

```bash
# 启动es容器,因为上一步修改了docker-compose.yml文件，所以使用docker-compose来启动
[root@localhost ~]# cd /root/docker-compose/elasticsearch
[root@localhost elasticsearch]# docker-compose up -d 
Recreating es01 ... done

# 进入容器
[root@localhost elasticsearch]# docker exec -it elasticsearch bash
# 以交互方式设置密码，如下第一个提示输入y，其余的密码我都设置成了123456
[root@docker-14 elasticsearch]# ./bin/elasticsearch-setup-passwords interactive   
Initiating the setup of passwords for reserved users elastic,apm_system,kibana,logstash_system,beats_system,remote_monitoring_user.
You will be prompted to enter passwords as the process progresses.
Please confirm that you would like to continue [y/N]y


Enter password for [elastic]: 
Reenter password for [elastic]: 
Enter password for [apm_system]: 
Reenter password for [apm_system]: 
Enter password for [kibana]: 
Reenter password for [kibana]: 
Enter password for [logstash_system]: 
Reenter password for [logstash_system]: 
Enter password for [beats_system]: 
Reenter password for [beats_system]: 
Enter password for [remote_monitoring_user]: 
Reenter password for [remote_monitoring_user]: 
Changed password for user [apm_system]
Changed password for user [kibana]
Changed password for user [logstash_system]
Changed password for user [beats_system]
Changed password for user [remote_monitoring_user]
Changed password for user [elastic]

# 设置完成后退出容器
[root@docker-14 elasticsearch]# exit
exit
```

## 收集nginx日志

```yaml
version: '2.2'
services:
  nginx:
    image: nginx:stable-alpine
    container_name: nginx-alpine
    restart: always
    privileged: true
    environment:
      - TZ=Asia/Shanghai 
    ports:
      - 80:80
    labels:
      - "aliyun.logs.nginx=stdout" #nginx是索引名，stdout采集docker屏幕输出日志
      - "aliyun.logs.access=/var/log/nginx/access.log" #access是索引名，采集容器内文件日志
    volumes:
      - /etc/localtime:/etc/localtime:ro
```

