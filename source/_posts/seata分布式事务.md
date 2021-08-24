---
title: seata分布式事务
tags: [seata]
categories: 中间件
date: 2020-07-13 15:00:00
---

# 介绍

角色介绍：

- XID：一个全局事务的唯一标识，由ip:port:sequence组成
- Transaction Coordinator (TC)： 事务协调器，维护全局事务的运行状态，负责协调并驱动全局事务的提交或回滚。
- Transaction Manager （TM）： 控制全局事务的边界，负责开启一个全局事务，并最终发起全局提交或全局回滚的决议。
- Resource Manager (RM)： 控制分支事务，负责分支注册、状态汇报，并接收事务协调器的指令，驱动分支（本地）事务的提交和回滚。

## k8s高可用部署

```yaml
apiVersion: v1
kind: Service
metadata:
  name: seata-ha-server
  namespace: default
  labels:
    app.kubernetes.io/name: seata-ha-server
spec:
  type: ClusterIP
  ports:
    - port: 8091
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/name: seata-ha-server

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: seata-ha-server
  namespace: default
  labels:
    app.kubernetes.io/name: seata-ha-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: seata-ha-server
  template:
    metadata:
      labels:
        app.kubernetes.io/name: seata-ha-server
    spec:
      containers:
        - name: seata-ha-server
          image: docker.io/seataio/seata-server:latest
          imagePullPolicy: IfNotPresent
          env:
            - name: SEATA_CONFIG_NAME
              value: file:/root/seata-config/registry
          ports:
            - name: http
              containerPort: 8091
              protocol: TCP
          volumeMounts:
            - name: seata-config
              mountPath: /root/seata-config
            - name: file-conf
              mountPath: /root/seata-config1
      volumes:
        - name: seata-config
          configMap:
            name: seata-ha-server-config
        - name: file-conf
          configMap:
            name: file-config

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: file-config
data:
  file.conf: |
    ## transaction log store, only used in seata-server
    store {
      ## store mode: file、db、redis
      mode = "db"

      ## 填写数据库连接信息
      db {
        ## the implement of javax.sql.DataSource, such as DruidDataSource(druid)/BasicDataSource(dbcp)/HikariDataSource(hikari) etc.
        datasource = "druid"
        ## mysql/oracle/postgresql/h2/oceanbase etc.
        dbType = "mysql"
        driverClassName = "com.mysql.jdbc.Driver"
        url = "jdbc:mysql://192.168.10.70:3306/seata"
        user = "root"
        password = "mamahao"
        minConn = 5
        maxConn = 30
        globalTable = "global_table"
        branchTable = "branch_table"
        lockTable = "lock_table"
        queryLimit = 100
        maxWait = 5000
      }
    }
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: seata-ha-server-config
data:
  registry.conf: |
    registry {
      # eureka配置信息
      # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
      type = "eureka"
    
      eureka {
        serviceUrl = "http://10.107.152.214:9901/eureka"
        application = "default"
        weight = "1"
      }
      
    }
    
    config {
      # file、nacos 、apollo、zk、consul、etcd3
      type = "file"
      
      file {
        # 指定file.conf的文件位置
        name = "file:/root/seata-config1/file.conf"
      }
    }

```

