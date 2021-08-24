---
title: maven-3.6.1安装脚本 
tags: [maven]
categories: linux基础
date: 2019-09-01 11:00:00
---


# 介绍
Maven 翻译为"专家"、"内行"，是 Apache 下的一个纯 Java 开发的开源项目。基于项目对象模型（缩写：POM）概念，Maven利用一个中央信息片断能管理一个项目的构建、报告和文档等步骤。

Maven 是一个项目管理工具，可以对 Java 项目进行构建、依赖管理。

Maven 也可被用于构建和管理各种项目，例如 C#，Ruby，Scala 和其他语言编写的项目。Maven 曾是 Jakarta 项目的子项目，现为由 Apache 软件基金会主持的独立 Apache 项目。

# 安装脚本

[jdk1.8环境安装](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85-linux/)

``` bash
tar xf apache-maven-3.6.1-bin.tar.gz -C /opt

#maven
echo "
export MAVEN_HOME=/opt/apache-maven-3.6.1
export PATH=\${PATH}:\${MAVEN_HOME}/bin
" >> /etc/profile
	
source /etc/profile
```

# 更换仓库镜像站

```bash
# 配置文件
vim /usr/local/apache-maven-3.6.1/conf/settings.xml
```

```xml
<!-- 阿里云站点 -->
<mirror>
      <id>alimaven</id>
      <name>aliyun maven</name>
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url>
      <mirrorOf>central</mirrorOf>        
</mirror>
```



# java源码编译测试

```bash
# 下载zk源码
git clone https://gitee.com/Outsrkem/zkweb.git

# 编译
cd zkweb
mvn clean package

# 安装包位置
zkweb/target/zkWeb-v1.2.1.war
```

