---
title: nginx之反向代理获取源IP
tags: [nginx]
categories: linux基础
date: 2020-06-04 17:40:43
---

# 介绍

 我们访问互联网上的服务时，大多数时，客户端并不是直接访问到服务端的，而是客户端首先请求到反向代理，反向代理再转发到服务端实现服务访问，通过反向代理实现路由/负载均衡等策略。这样在服务端拿到的客户端IP将是反向代理IP，而不是真实客户端IP，因此需要想办法来获取到真实客户端IP，nginx需要加载模块`--with-http_realip_module`。

# 配置

**nginx反向代理服务器配置**

```bash
server {
    listen 80;
    server_name  localhost;
    location /pos/ {
        proxy_pass http://192.168.40.101;
        # 存放用户的真实ip
        proxy_set_header   X-Real-IP        $remote_addr;
        # 每经过一个反向代理，就会把反向代理IP存放在X-Forwarded-For里
        proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
    }
}
```

**nginx服务端配置**

```bash
    server {
        listen       80;
        server_name  localhost;

        # 告知Nginx哪些是反向代理IP，即排除后剩下的就是真实客户端IP
        set_real_ip_from 192.168.40.100;
        # 告知Nginx真实客户端IP从哪个请求头获取；
        real_ip_header    X-Forwarded-For;
        # 是否递归解析，当real_ip_recursive配置为off时，Nginx会把real_ip_header指定的请求头中的最后一个IP作为真实客户端IP；当real_ip_recursive配置为on时，Nginx会递归解析real_ip_header指定的请求头，最后一个不匹配set_real_ip_from的IP作为真实客户端IP。 
        real_ip_recursive on;

        location / {
            root   html;
            index  index.html index.htm;
        }
	}
```

