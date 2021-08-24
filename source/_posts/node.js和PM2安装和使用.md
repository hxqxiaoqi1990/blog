---
title: node.js和PM2安装和使用
tags: [node]
categories: linux基础
date: 2019-10-26 15:41:00
---

# 介绍
**Node.js：** 就是运行在服务端的 JavaScript。
Node.js 是一个基于Chrome JavaScript 运行时建立的一个平台。
Node.js是一个事件驱动I/O服务端JavaScript环境，基于Google的V8引擎，V8引擎执行Javascript的速度非常快，性能非常好。

**NPM：** 是随同NodeJS一起安装的包管理工具，能解决NodeJS代码部署上的很多问题，常见的使用场景有以下几种：

 1. 允许用户从NPM服务器下载别人编写的第三方包到本地使用。
 2. 允许用户从NPM服务器下载并安装别人编写的命令行程序到本地使用。
 3. 允许用户将自己编写的包或命令行程序上传到NPM服务器供别人使用。

**PM2：** 是node进程管理工具，可以利用它来简化很多node应用管理的繁琐任务，如性能监控、自动重启、负载均衡等，而且使用非常简单。

下面就对PM2进行入门性的介绍，基本涵盖了PM2的常用的功能和配置。
# node.js安装

``` bash
wget https://cdn.npm.taobao.org/dist/node/v12.13.0/node-v12.13.0-linux-x64.tar.xz

tar xf node-v12.13.0-linux-x64.tar.xz -C /opt/
cd /opt
mv node-v12.13.0-linux-x64 node
ln -s /opt/node/bin/npm /usr/local/bin/
ln -s /opt/node/bin/node /usr/local/bin/

node -v
npm -v
```
## npm常用命令
1. **查看node版本**
node --version
2. **查看npm 版本**
npm -v
3. **安装cnpm (国内淘宝镜像源)**
npm install cnpm -g --registry=https://registry.npm.taobao.org
4. **安装express模块**
npm install express
5. **全局安装express模块**
npm install -g express
6. **列出已安装模块**
npm list
7. **显示模块详情**
npm show express
8. **升级当前目录下的项目的所有模块**
npm update
9. **升级当前目录下的项目的指定模块**
npm update express
10. **升级全局安装的express模块**
npm update -g express
11. **删除指定的模块**
npm uninstall express
12. **更新node 版本**
首先需要确保是否安装 n 模块，这个是node升级需要
没有安装执行：npm i n -g -f
检测使用: n --version
更新node命令：n stable

## pm2常用命令
1. **启动app.js应用程序**
pm2 start app.js              
2. **cluster mode 模式启动4个app.js的应用实例** 
pm2 start app.js -i 4
3. **启动应用程序并命名为 "api"**
pm2 start app.js --name="api" 
4. **当文件变化时自动重启应用**
pm2 start app.js --watch     
5. **启动 bash 脚本**
pm2 start script.sh
6. **列表 PM2 启动的所有的应用程序**
pm2 list                      
7. **显示每个应用程序的CPU和内存占用情况**
pm2 monit                    
8. **显示应用程序的所有信息**
pm2 show app          
9. **显示所有应用程序的日志**
pm2 logs                      
10. **显示指定应用程序的日志**
pm2 logs app
11. **清空所有日志文件**
pm2 flush
12. **停止所有的应用程序**
pm2 stop all                  
13. **停止 id为 0的指定应用程序**
pm2 stop 0                   
14. **重启所有应用**
pm2 restart all              
15. **重启 cluster mode下的所有应用**
pm2 reload all               
16. **重启集群所有应用**
pm2 gracefulReload all
17. **关闭并删除所有应用**
pm2 delete all                
18. **删除指定应用 id 0**
pm2 delete 0                  
19. **把名字叫api的应用扩展到10个实例**
pm2 scale api 10              
20. **重置重启数量**
pm2 reset app   
21. **创建开机自启动命令**
pm2 startup                   
22. **保存当前应用列表**
pm2 save                      
23. **重新加载保存的应用列表**
pm2 resurrect                 

