---
title: prometheus+grafana监控部署
tags: [prometheus]
categories: 监控
date: 2019-08-04 11:00:00
---


# 介绍
Prometheus是一个开源监控系统，它前身是SoundCloud的警告工具包。从2012年开始，许多公司和组织开始使用Prometheus。该项目的开发人员和用户社区非常活跃，越来越多的开发人员和用户参与到该项目中。目前它是一个独立的开源项目，且不依赖与任何公司。为了强调这点和明确该项目治理结构，Prometheus在2016年继Kurberntes之后，加入了Cloud Native Computing Foundation。
# 安装步骤
监控机安装：192.168.1.24
 1. 安装prometheus，监控服务
 2. 安装node-exporter，节点数据收集服务
 3. 安装alertmanager，报警服务
 4. 安装grafana，图形化展示服务

## 一、监控机安装prometheus
### 1、安装命令

``` bash
# 下载
wget https://github.com/prometheus/prometheus/releases/download/v2.12.0/prometheus-2.12.0.linux-amd64.tar.gz

tar xf prometheus-2.12.0.linux-amd64.tar.gz
cd prometheus-2.12.0.linux-amd64

# 根据下面yml配置文件配置需要信息
vim prometheus.yml

# 启动
nohup ./prometheus --config.file=./prometheus.yml &

# 访问prometheus
http://192.168.1.24:9090
```

### 2、配置修改：vim prometheus.yml

``` yml
global:
  # 每隔15秒向pushgateway采集一次指标数据
  scrape_interval:    15s 
  # 每隔15秒根据所配置的规则集，进行规则计算
  evaluation_interval: 15s 

alerting:
  alertmanagers:
  - static_configs:
  # 设置altermanager的地址，altermanager报警规则服务
    - targets: ['192.168.1.24:9093']

  # 指定所配置报价模板文件，下面给出模板配置
rule_files:
  - "rules.yml"

scrape_configs:
  # 用于获取被控主机数据，可以添加多条
  - job_name: '测试-25'
    static_configs:
    - targets: ['192.168.1.25:9100']
 
```
### 3、rules.yml报警模板配置

``` yml
groups:
- name: down
  rules:
  - alert: "down报警"
    expr: up == 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "down报警"
      description: "报警时间:"
      value: "已使用：{{ $value }}"
- name: memory
  rules:
  - alert: "内存报警"
    expr: ((node_memory_MemTotal_bytes -(node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes) )/node_memory_MemTotal_bytes ) * 100 > 90
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "内存报警"
      description: "报警时间:"
      value: "已使用：{{ $value }}%"
- name: cpu
  rules:
  - alert: "cpu报警"
    expr: 100 - ((avg by (instance,job,env)(irate(node_cpu_seconds_total{mode="idle"}[30s]))) *100) > 90
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "cpu报警"
      description: "报警时间:"
      value: "已使用：{{ $value }}%"
- name: disk
  rules:
  - alert: "disk报警"
    expr: 100 - (node_filesystem_avail_bytes{fstype !~ "nfs|rpc_pipefs|rootfs|tmpfs",device!~"/etc/auto.misc|/dev/mapper/centos-home",mountpoint !~ "/boot|/net|/selinux"} /node_filesystem_size_bytes{fstype !~ "nfs|rpc_pipefs|rootfs|tmpfs",device!~"/etc/auto.misc|/dev/mapper/centos-home",mountpoint !~ "/boot|/net|/selinux"} ) * 100  > 80
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "disk报警"
      description: "报警时间:"
      value: "已使用：{{ $value }}%"
- name: net
  rules:
  - alert: "net报警"
    expr: (irate(node_network_transmit_bytes_total{device!~"lo"}[1m]) / 1000) > 80000
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "net报警"
      description: "报警时间:"
      value: "已使用：{{ $value }}KB"
```

## 二、监控机安装node-exporter

``` bash
# 下载
wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz

tar xf node_exporter-0.18.1.linux-amd64.tar.gz
cd node_exporter-0.18.1.linux-amd64

# 启动
nohup ./node_exporter &

# 访问
http://192.168.1.24:9100
```

## 三、监控机安装alertmanager
### 1、安装命令
``` bash
# 下载
wget https://github.com/prometheus/alertmanager/releases/download/v0.19.0/alertmanager-0.19.0.linux-amd64.tar.gz

tar xf alertmanager-0.19.0.linux-amd64.tar.gz
cd alertmanager-0.19.0.linux-amd64

# 修改配置，下面给出配置信息
vim alertmanager.yml

# 启动
nohup ./alertmanager --config.file=./alertmanager.yml &

# 访问
http://192.168.1.24:9093
```
### 2、alertmanager.yml报警规则配置

以下是邮箱报警配置：
``` yml
global:
  #处理超时时间，默认为5min
  resolve_timeout: 5m   
  # 邮箱smtp服务器代理
  smtp_smarthost: 'smtp.163.com:25'   
  # 发送邮箱名称
  smtp_from: 'hxqxiaoqi1990@163.com'    
  # 邮箱名称
  smtp_auth_username: 'hxqxiaoqi1990@163.com'   
  # 邮箱密码或授权码
  smtp_auth_password: 'Hxq7996026'      
  smtp_require_tls: false

route:
  # 报警分组依据
  group_by: ['alertname']       
  # 最初即第一次等待多久时间发送一组警报的通知
  group_wait: 10s       
  # 在发送新警报前的等待时间
  group_interval: 1h    
  # 发送重复警报的周期 对于email配置中，此项不可以设置过低，否则将会由于邮件发送太多频繁，被smtp服务器拒绝
  repeat_interval: 1h   
  # 发送警报的接收者的名称，以下receivers name的名称
  receiver: 'mail'      
receivers:
- name: 'mail'
  email_configs:
  - to: 'hxqxiaoqi1990@163.com'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

## 四、监控机安装grafana
### 1、安装命令
``` bash
wget https://dl.grafana.com/oss/release/grafana-6.3.5.linux-amd64.tar.gz 

tar xf grafana-6.3.5.linux-amd64.tar.gz 
cd grafana-6.3.5.linux-amd64

# 启动
nohup ./alertmanager --config.file=./alertmanager.yml &

# 访问
http://192.168.1.24:3000
```
### 2、添加数据源

1.在设置中：Configuration-->Date Sources
2.配置prometheus服务信息：http://192.168.1.24:9090

### 3、添加仪表盘
1.在创建中：Create-->Import
2.去下载：https://grafana.com/grafana/dashboards 或复制模板ID号
3.填写ID号，或上传仪表盘模板