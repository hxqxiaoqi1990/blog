---
title: JDK1.8环境安装-linux
tags: [jdk]
categories: linux基础
date: 2019-06-04 17:40:43
---


# 什么是JDK
<font color=DarkTurquoise>**JDK**</font>  (Java Development Kit) 是 Java 语言的软件开发工具包(SDK)。在JDK的安装目录下有一个jre目录，里面有两个文件夹bin和lib，在这里可以认为bin里的就是jvm，lib中则是jvm工作所需要的类库，而jvm和 lib合起来就称为jre。
<font color=DarkTurquoise>**JRE**</font>（Java Runtime Environment，Java运行环境），包含JVM标准实现及Java核心类库。JRE是Java运行环境，并不是一个开发环境，所以没有包含任何开发工具（如编译器和调试器）
<font color=DarkTurquoise>**JVM**</font>  是Java Virtual Machine（Java虚拟机）的缩写，JVM是一种用于计算设备的规范，它是一个虚构出来的计算机，是通过在实际的计算机上仿真模拟各种计算机功能来实现的。

# JDK安装思路

 1. 下载JDK
 2. 上传linux服务器，解压
 3. 设置环境变量
 4. 查看版本

## 1. 下载JDK
JDK官网下载地址：https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html
请根据自己服务器情况下载对应版本，linux是下载tar.gz结尾的包

## 2. 上传linux服务器
把JDK包上传服务器有多种方法，我是用Xshell自带的ftp功能上传服务器，比较简单，这边不多讲。上传完成后

## 3. 设置环境变量

粘贴命令直接执行
``` bash
tar xf jdk-8u221-linux-x64.tar.gz -C /opt

cat >> /etc/profile << 'EOF'
#jdk8
export JAVA_HOME=/opt/jdk1.8.0_221
export JRE_HOME=$JAVA_HOME/jre
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
EOF

source /etc/profile
java -version
```
