---
title: heox主题搭建
tags: hexo
categories: 随笔
date: 2019-05-31 20:27:43
---
# hexo简介
Hexo是一款基于Node.js的静态博客框架，依赖少易于安装使用，可以方便的生成静态网页托管在GitHub和Coding上，是搭建博客的首选框架。大家可以进入hexo官网进行详细查看，因为Hexo的创建者是台湾人，对中文的支持很友好，可以选择中文进行查看。

# Hexo搭建步骤
 1. 安装Git
 2. 安装Node.js
 3. 安装Hexo
 4. 更换博客主题

## 1. 安装git

Git是目前世界上最先进的分布式版本控制系统，可以有效、高速的处理从很小到非常大的项目版本管理。也就是用来管理你的hexo博客文章，上传到GitHub的工具。到git官网上下载,**https://gitforwindows.org** 直接双击安装即可。

## 2. 安装Node.js

Hexo是基于nodeJS编写的，所以需要安装一下nodeJs和里面的npm工具。官网下载：**https://nodejs.org/en/download/** 直接安装。

## 3. 安装Hexo

前面git和nodejs安装好后，就可以安装hexo了。

创建一个自定义文件夹blog，右击该文件夹选择git bash here，会跳出git命令框。安装命令：
``` bash
 npm install -g hexo-cli	#安装hexo

 hexo -v			#查看版本
 
 hexo init		#初始化目录（会在目录下生成安装文件）
```


初始化后，blog文件夹目录下有：

 - node_modules: 依赖包
 - public：存放生成的页面
 - scaffolds：生成文章的一些模板
 - source：用来存放你的文章
 - themes：主题目录（默认已经有一个主题landscape）
 - _config.yml: 博客的配置文件

启动主题

``` bash
hexo g				#编译
hexo s				#启动主题
```
访问localhost:4000即可看到博客内容
至此hexo搭建完成

## 4. 更换博客主题

主题官网：**https://hexo.io/themes/** 这里有200多个主题可以选。点击主题名（以BlueLake）会链接到github上，下载zip包，然后放到themes文件夹下解压为BlueLake。

接下来打开blog目录下的_config.yml配置文件，修改：

``` bash
theme: BlueLake			#名字为解压的主题名
```

运行命令：

``` bash
hexo g				#编译
hexo s				#启动主题
```
访问localhost:4000，主题已更换

**注：有些主题还需要安装其它样式模块，可以在主题目录下查看README.md文件，是否需要安装模块。**