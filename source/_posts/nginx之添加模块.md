---
title: nginx之添加模块
tags: [nginx]
categories: linux基础
date: 2020-07-20 17:40:43
---

查看模块

```bash
# 查看可用模块及已安装模块
cd nginx-1.14.2
cat auto/options | grep YES

# 查看已安装模块
./nginx -V
```

添加模块

```bash
# 到安装包
cd nginx-1.14.2

# 编译，别make install，会覆盖原配置
./configure --prefix=/opt/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module --with-http_realip_module

# 编译
make

# 备份nginx文件
cp /opt/nginx/sbin/nginx /mnt

# 复制新的nginx文件到nginx安装目录下
cp objs/nginx /opt/nginx/sbin/

# 查看安装模块
/opt/nginx/sbin/nginx -V
```

添加第三方模块 (nginx_upstream_check_module-master)

```bash
# 下载、解压
wget https://codeload.github.com/yaoweibin/nginx_upstream_check_module/zip/master
tar xf nginx_upstream_check_module-master.zip

# 到nginx安装包下
cd nginx-1.14.2

# 导入模块
patch -p1 < ../nginx_upstream_check_module-master/check_1.14.0+.patch

# 试编译
./configure --prefix=/opt/nginx --user=nginx --group=nginx --with-http_ssl_module --with-http_stub_status_module --with-http_gzip_static_module --with-http_realip_module --add-module=../nginx_upstream_check_module-master/

# 编译，千万别执行：make install
make

# 备份nginx文件
cp /opt/nginx/sbin/nginx /mnt

# 复制新的nginx文件到nginx安装目录下
cp objs/nginx /opt/nginx/sbin/

# 查看安装模块
/opt/nginx/sbin/nginx -V
```

