---
title: dockerfile几个实例
tags: [docker]
categories: kubernetes
data: 2019-6-20 11:00:00
---


# 实例说明
本章会写两个dockerfile实例和一个shell脚本调用dockerfile实例，重点在于shell和dockerfile中环境变量的调用以及调用系统变量，这里有些坑会重点说明。

# dockerfile封装jdk

以下代码存为名为`Dockerfile`的文件，把下载好的jdk包放在Dockerfile同一目录中，运行dockerfile，即可生成拥有jdk环境的系统镜像，我把镜像上传到阿里云仓库中，方便可以随时使用。

``` dockerfile
#依赖镜像名称和ID
FROM centos

#切换工作目录
WORKDIR /usr
RUN mkdir  /usr/local/java

#ADD 是相对路径jar,把java添加到容器中
ADD jdk-8u191-linux-x64.tar.gz /usr/local/java/

#配置java环境变量
ENV JAVA_HOME /usr/local/java/jdk1.8.0_191
ENV JRE_HOME $JAVA_HOME/jre
ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib:$CLASSPATH
ENV PATH $JAVA_HOME/bin:$PATH
#配置系统语言
ENV LANG en_US.UTF-8

#运行
ENTRYPOINT tail -f /anaconda-post.log
```

# dockerfile封装jar包服务

以下代码存为名为Dockerfile的文件，跟下面shell脚本配合使用，注意：dockerfile中的变量只是容器中的变量，系统变量无法在dockerfile中生效，只有通过ARG向dockerfile中传入系统变量，ARG指令需要在`docker build`时指定其变量。

``` dockerfile
#依赖镜像名称和ID
#阿里云内网
FROM registry-vpc.cn-hangzhou.aliyuncs.com/znknow/page:1.8
#阿里云外网
#FROM registry.cn-hangzhou.aliyuncs.com/znknow/page:1.8

#系统环境变量
ARG pagej
ARG pagep

#dockerfile环境变量
ENV jar $pagej
ENV port $pagep
ENV LANG en_US.UTF-8

COPY ./$jar /usr/local/
EXPOSE $port

ENTRYPOINT java -Xdebug -Xnoagent -Djava.compiler=NONE -Xrunjdwp:transport=dt_socket,address=12000,server=y,suspend=n  -jar /usr/local/$jar -Djava.io.tmpdir=/data/tmp &> /var/log/$jar.log
```

# shell启动dockerfile

该shell脚本为运行dockerfile并向dockerfile中传入相应变量。

``` bash
#!/bin/bash

#定义shell环境变量
export pagedir=`pwd`
export pagejar=$1
export pageport=$2
export pageport3=$3
export pagetag="1.0"
export pageago=`docker ps -a | grep $pagejar | cut -d " " -f1`
export pageagoim=`docker images | grep $pagejar |awk -F " " '{print $3}'`

#删除原镜像和容器，这里不像生成多个相同镜像，也可以使用标签生成多个镜像
/usr/bin/docker rm -f $pageago
/usr/bin/docker rmi -f $pageagoim

#运行dockerfile指令，--build-arg为传入dockerfile中的环境变量
/usr/bin/docker build --build-arg pagej=$pagejar --build-arg pagep=$pageport3 -t $pagejar:$pagetag .

#启动容器
/usr/bin/docker run --restart always -v $pagedir:/var/log/  -v /etc/localtime:/etc/localtime -v /etc/timezone:/etc/timezone -p $pageport:$pageport3 --name $pagejar-$pagetag -d $pagejar:$pagetag
```

# docker-zkui

```bash
FROM registry.cn-hangzhou.aliyuncs.com/mmh/jdk:1.8.0

ENV LANG en_US.UTF-8
ENV JAVA_HOME /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64
ENV CLASSPATH .:/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64/dt.jar:/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64/lib/tools.jar

ADD ./zkui-2.0-SNAPSHOT-jar-with-dependencies.jar /opt/zkui/zkui-2.0-SNAPSHOT-jar-with-dependencies.jar
ADD ./config.cfg /opt/zkui/config.cfg

RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN yum -y install vim net-tools telnet

# $zkcluster变量可以在启动镜像时指定
ENTRYPOINT sed -i 's/zkServer=.*/zkServer='$zkcluster'/' /opt/zkui/config.cfg && cd /opt/zkui/ && java -Xms512m -Xmx512m -XX:PermSize=128m -jar /opt/zkui/zkui-2.0-SNAPSHOT-jar-with-dependencies.jar
```

# docker-zipkin

```bash
FROM registry.cn-hangzhou.aliyuncs.com/mmh/jdk:1.8.0

ENV LANG en_US.UTF-8
ENV JAVA_HOME /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64
ENV CLASSPATH .:/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64/dt.jar:/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.232.b09-0.el7_7.x86_64/lib/tools.jar

ADD ./zipkin.tar.gz /data/

RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN yum -y install vim net-tools

ENTRYPOINT cd /data/zipkin && java -Xms512m -Xmx512m -XX:PermSize=128m -jar /data/zipkin/zipkin.jar
```

# docker-disconf

```bash
FROM centos:7.4.1708

ENV LANG en_US.UTF-8
ENV JAVA_HOME /opt/jdk1.8.0_221
ENV JRE_HOME ${JAVA_HOME}/jre
ENV CLASSPATH .:${JAVA_HOME}/lib:${JRE_HOME}/lib
ENV PATH ${JAVA_HOME}/bin:$PATH
ENV MAVEN_HOME /opt/apache-maven-3.3.9
ENV PATH $PATH:$JAVA_HOME/bin:$MAVEN_HOME/bin
ENV ONLINE_CONFIG_PATH /usr/local/disconf/resource
ENV WAR_ROOT_PATH /usr/local/disconf/war

ADD ./jdk-8u221-linux-x64.tar.gz /opt/
ADD ./apache-maven-3.3.9-bin.tar.gz /opt/
ADD ./apache-tomcat-8.5.47.tar.gz /opt/
ADD ./disconf.tar.gz /usr/local/
ADD ./nginx.sh /opt/
ADD ./redis.sh /opt/
ADD ./redis-stable.tar.gz /opt/
ADD	./nginx-1.14.2.tar.gz /opt/
ADD	./start.sh /opt/
ADD	./nginx.conf /opt/
ADD	./server.xml /opt/
ADD	./nginx.service /lib/systemd/system/

RUN /bin/cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && echo 'Asia/Shanghai' >/etc/timezone
RUN yum -y install gcc gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel wget vim net-tools telnet
RUN sh /opt/nginx.sh
RUN sh /opt/redis.sh
RUN \mv /opt/nginx.conf /opt/nginx/conf/
RUN \mv /opt/server.xml /opt/apache-tomcat-8.5.47/conf/
RUN cd /opt/ && rm -rf nginx-1.14.2  nginx.sh  redis.sh 

ENTRYPOINT cd /opt/redis-stable/ && ./src/redis-server ./redis.conf && bash /opt/apache-tomcat-8.5.47/bin/startup.sh && /opt/nginx/sbin/nginx -g "daemon off;"
```

