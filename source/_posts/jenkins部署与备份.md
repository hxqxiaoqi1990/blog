---
title: jenkins部署与备份 
tags: [jenkins]
categories: 中间件
date: 2019-07-30 14:27:43
---

# jenkins介绍

Jenkins只是一个平台，真正运作的都是插件，它是一个自动化部署工具，通过插件完成拉起源代码、编译、发布等功能。

下载地址：http://mirrors.jenkins.io/war-stable/latest/jenkins.war

# jenkins部署
部署jenkins之前需要安装jdk环境，请按照：[jdk环境部署](https://hxqxiaoqi.gitee.io/tags/jdk/) 操作安装jdk。

## 部署方式一
1.下载的**Jenkins.war**包上传到服务器
2.服务器上执行以下命令：

``` bash
#前台运行
java -jar jenkins.war

#后台运行
nohup java -jar jenkins.war &> ./jen.log &

#获取Jenkins密码，访问时需要
cat /root/.jenkins/secrets/initialAdminPassword
```
3.访问：**localhost:8080**，输入密码
4.选择默认模式
5.设置登录账号密码
## 部署方式二

1.安装tomcat
2.下载的Jenkins.war包上传到tomcat目录下的webapps目录下
3.运行tomcat
4.访问：localhost:8080/jenkins

## jenkins工作目录说明
第一次启动jenkins时，会在用户家目录下自动生成 **.jenkins** 的隐藏工作目录。

**.jenkins：**
**config.xml：** jenkins 的核心配置文件
**jobs：** 构建作业的配置细节，及构建产物和数据
**workspace：** jenkins 对当前作业进行构建的地方
**builds：** 包含当前作业的构建历史
**config.xml：** 存放当前作业的所有配置细节
**nextBuildNumber：** 下一次构建的 number
**lastStable：** 最后一个稳定构建的链接（成功的构建）
**lastSuccessful：** 最近成功的构建链接（没有任何编译错误）
**plugins：** 存放所有已安装的插件，更新 jenkins 不需要重新安装插件
**users：** 当使用 jenkins 本地用户数据库时，用户信息会存放在这个目录下
**updates：** 存放可用的插件更新
**userContent：** 存放用户自己为 jenkins 服务器定制化的一些内容
**war：** 存放扩展的 web 应用程序，当以单机应用程序的形式运，jenkins 时，会把 web 应用程序解压到这个目录。

## 更改工作目录

``` bash
#默认jenkins主目录在/root/.jenkins
cp -a /root/.jenkins /home/.jenkins

#添加profile
vim /etc/profile
	#jenkins
	JENKINS_HOME="/home/.jenkins"
	export JENKINS_HOME
	
#重新加载profile
source /etc/profile
```
# jenkins备份与恢复
1.登录Jenkins-->选择系统管理-->选择插件管理-->选择可选插件
2.搜索ThinBackup插件，安装
3.安装成功后，在系统管理中出现ThinBackup选项，点击ThinBackup

设置说明：
Backup Now：立即备份，需要设置备份路径，之后在服务器上创建该目录
Restore：恢复，按时间选择备份文件
Settings：备份设置，设置备份路径，定时备份时间，备份方式等
# jenkins配置项目
这里以java项目举例说明：
配置java项目需要安装插件：**Publish Over SSH** 、 **Maven Release Plug-in**和**Git**

1.登录jenkins，点击新建视图
2.输入项目名，选择简单视图，保存
3.在该项目下点击新建任务，输入任务名称，选择构建maven项目，选择ok
4.选择**源码管理**的类型，我这边选择git，输入git地址和账号密码
5.在**Build**栏目，添加pom配置路径

``` bash
例1：pom在当前目录
	./pom.xml
	clean install package
		
例2：pom在二级目录
	ytb-manager-server/pom.xml
	clean install -pl ../ytb-manager-server -am
```

6.在**Post Steps**点击 **Add post-build step**，选择**send files or execute commands over SSH**
7.ssh配置举例：

``` bash
name：选择jar包传输的服务器，需要在系统设置中先添加ssh主机

#生成jar包位置
Source files：ytb-manager-server/target/*.jar

#移除Source files路径
Remove prefix：ytb-manager-server/target/

#jar上传到服务的目录
Remote directory：/home/opt/ytb

#服务器执行的脚本
Exec command：
	#!/bin/bash
	cd /home/opt/ytb/manager-server
	sh ./stop.sh
	sh ./start.sh
```
8.保存配置，之后点击立即构建即可查看是否成功。

# 遇到的问题
## 安装插件失败
因为国内防火墙的问题，有时候无法获取到Jenkins的插件，或下载失败
1.更换插件源地址
2.https://updates.jenkins.io/update-center.json 更换为 http://mirror.xmission.com/jenkins/updates/update-center.json
3.或更换为https://updates.jenkins.io/update-center.json
4.点击提交
5.点击立即获取
