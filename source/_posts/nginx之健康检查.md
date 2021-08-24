---
title: nginx之健康检查
tags: [nginx]
categories: linux基础
date: 2020-07-20 16:40:43
---

# 介绍

nginx健康检查分为：

- 被动：ngx_http_proxy_module 模块和ngx_http_upstream_module模块，会有失败的连接。（自带）
- 主动：nginx 模块 nginx_upstream_check_module，通过它可以用来检测后端 realserver 的健康状态。（淘宝第三方）

# 配置

被动检查

```bash
upstream name {
    server 10.1.1.110:8080 max_fails=1 fail_timeout=10s;
    server 10.1.1.122:8080 max_fails=1 fail_timeout=10s;
}

max_fails=number        # 设定Nginx与服务器通信的尝试失败的次数。在fail_timeout参数定义的时间段内，如果失败的次数达到此值，Nginx就认为服务器不可用。在下一个fail_timeout时间段，服务器不会再被尝试。 失败的尝试次数默认是1。设为0就会停止统计尝试次数，认为服务器是一直可用的。 你可以通过指令proxy_next_upstream、fastcgi_next_upstream和 memcached_next_upstream来配置什么是失败的尝试。 默认配置时，http_404状态不被认为是失败的尝试。
fail_timeout=time       # 设定服务器被认为不可用的时间段以及统计失败尝试次数的时间段。在这段时间中，服务器失败次数达到指定的尝试次数，服务器就被认为不可用。默认情况下，该超时时间是10秒。
```

主动检查

```bash
# interval检测间隔时间，单位为毫秒，rsie请求2次正常的话，标记此realserver的状态为up，fall表示请求5次都失败的情况下，标记此realserver的状态为down，timeout为超时时间，单位为毫秒。
upstream linuxyan {
    server 192.168.10.21:80;
    server 192.168.10.22:80;
    check interval=3000 rise=2 fall=5 timeout=1000;
}

# web查看realserver状态的页面
location /nstatus {
check_status;
access_log off;
}
```