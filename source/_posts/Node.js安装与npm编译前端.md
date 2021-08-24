---
title: Node.js安装与npm编译前端
tags: [node]
categories: linux基础
date: 2019-07-25 18:27:43
---
# Node.js 介绍

简单的说 Node.js 就是运行在服务端的 JavaScript。
Node.js 是一个基于Chrome JavaScript 运行时建立的一个平台。
Node.js是一个事件驱动I/O服务端JavaScript环境，基于Google的V8引擎，V8引擎执行Javascript的速度非常快，性能非常好。
# NPM 使用介绍
NPM是随同NodeJS一起安装的包管理工具，能解决NodeJS代码部署上的很多问题，常见的使用场景有以下几种：

 1. 允许用户从NPM服务器下载别人编写的第三方包到本地使用。
 2. 允许用户从NPM服务器下载并安装别人编写的命令行程序到本地使用。
 3. 允许用户将自己编写的包或命令行程序上传到NPM服务器供别人使用。

# linux安装Node.js

``` bash
#下载
wget https://npm.taobao.org/mirrors/node/v10.16.0/node-v10.16.0-linux-x64.tar.gz

#解压
tar xf node-v10.16.0-linux-x64.tar.gz
cd node-v10.16.0-linux-x64/bin/

#设置软链接，需要绝对路径
ln -s /root/node-v10.16.0-linux-x64/bin/node /usr/bin/
ln -s /root/node-v10.16.0-linux-x64/bin/npm /usr/bin/

#查看版本
node -v
v10.16.0	

npm -v	#安装成功
6.9.0
```

# NPM安装淘宝镜像
大家都知道国内直接使用 npm 的官方镜像是非常慢的，这里推荐使用淘宝 NPM 镜像。

``` bash
#安装
npm install cnpm -g --registry=https://registry.npm.taobao.org

#软链接
ln -s /root/node-v10.16.0-linux-x64/bin/cnpm /usr/bin/

#查看版本
cnpm -v
```

# NPM全局安装与本地安装
npm 的包安装分为本地安装（local）、全局安装（global）两种，从敲的命令行来看，差别只是有没有-g而已。

**本地安装：** 将安装包放在 ./node_modules 下（运行 npm 命令时所在的目录），如果没有 node_modules 目录，会在当前执行 npm 命令的目录下生成 node_modules 目录。

**全局安装：** 将安装包放在 /usr/local 下或者你 node 的安装目录。

``` bash
npm install express     # 本地安装
npm install express -g   # 全局安装
```

# NPM编译前端源码包
以下以web为源码包举例：

``` bash
#使用本地安装，cd到源码包目录下
cd web

#安装源码包依赖
cnpm install

#打包，会在web目录中生成build的包，这个就是可以发布的前端包
cnpm run build
```
# NPM常用命令
## 安装模块
以下都是本地安装，全局安装加参数-g
npm install <Name>

## 查看安装模块信息
npm list

## 卸载模块
npm uninstall express

## 更新模块
npm update express

## 搜索模块
npm search express

## 创建模块
### 生成 package.json 文件
npm init

### 在 npm 资源库中注册用户
npm adduser

### 发布模块
npm publish












