---
title: nginx-1.14.2自动安装脚本
tags: [nginx,shell脚本]
categories: 脚本
date: 2019-06-04 17:40:43
---


# 安装方法
前提服务器需要联网
 1. 在服务器上新建文件：nginx.sh
 2. 把代码复制到文件中
 3. 赋予执行权限：chmod 755 nginx.sh
 4. 执行脚本：./nginx.sh

# 脚本代码

``` bash
#!/bin/bash
# 安装目录
dir=/usr/local

#安装依赖
yum -y install gcc gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel wget

#下载nginx
wget http://nginx.org/download/nginx-1.14.2.tar.gz

#解压并安装
useradd -r -s /sbin/nologin nginx
tar xf ./nginx-1.14.2.tar.gz
cd nginx-1.14.2/
./configure --prefix=$dir/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module --with-http_realip_module
make && make install
```

# nginx基本命令
启动：

``` bash
/usr/local/nginx/sbin/nginx
```
热重启：

``` bash
/usr/local/nginx/sbin/nginx -s reload
```
停止：

``` bash
/usr/local/nginx/sbin/nginx -s stop
```