---
title: k8s之java应用配置说明
tags: [k8s]
categories: kubernetes
date: 2020-07-31 15:06:00
---

# 介绍

部署java应用时，启动jar的一些参数说明和使用，以及内存设置。

# 配置

```bash
# k8s启动jar包示例
["java","-Dserver.port=8080","-Ddisconf.env=pro","-Xms6144m","-Xmx6144m","-XX:PermSize=128m","-XX:-OmitStackTraceInFastThrow","-XX:+HeapDumpOnOutOfMemoryError","-XX:HeapDumpPath=/mnt/api-dump.log","-XX:MetaspaceSize=256m","-XX:MaxMetaspaceSize=256m","-jar","/data/mamahao-app-web.jar","--spring.profiles.active=pro","-Dfile.encoding=utf-8"]
```

`-Dserver.port=8080` ：指定配置端口，需jar包中有该配置。

`-Ddisconf.env=pro` ：指定配置文件环境，需jar包中有该配置。

`-Xms6144m` ：指定最小堆内存。

`-Xmx6144m` ： 指定最大堆内存。

`-XX:PermSize=128m` ：-XX:PermSize分配非堆最小内存，默认为物理内存的1/64。

`-XX:MaxPermSize=512m` ：分配非堆最大内存，默认为物理内存的1/4。 

`-XX:-OmitStackTraceInFastThrow` ：打印栈详细信息。

`-XX:+HeapDumpOnOutOfMemoryError` ：表示当JVM发生OOM时，自动生成DUMP文件。

`-XX:HeapDumpPath=/mnt/api-dump.log` ：指定DUMP文件位置。

`-XX:MetaspaceSize=256m` ：元空间最小值。

`-XX:MaxMetaspaceSize=256m` ：元空间最大值，超过该值，就会发生FGC。

`--spring.profiles.active=pro` ：指定配置环境。

`-Dfile.encoding=utf-8` ：指定字符编码。

```bash
# 停止后命令，主要用于优雅下线eureka
["bash","-c","curl -X POST http://localhost:8080/actuator/service-registry?status=down -H \"Content-Type:application/vnd.spring-boot.actuator.v2+json;charset=UTF-8\";sleep 40"]
```

# 内存配置

1. `-Xmx` 大小：使用`jmap -heap PID`查看堆内存使用量，以老年代使用量乘以2来确定堆的大小。
2. k8s最大内存限制：`-Xmx` 大小，再加1.5G为准。
3. `jstat -gccapacity PID` ：查看元数据使用量，排错。

