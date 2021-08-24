---
title: nginx location斜杆区别
tags: [nginx]
categories: linux基础
date: 2021-05-27 17:27:00
---

# 介绍

最新在配置nginx时,意外发现location中目录斜线有和没有的区别，百度了找找发现没有几个人说的清楚，最后找到一个兄弟写的还比较实用，再次谢过（https://blog.csdn.net/ruihaol/article/details/79526749?from=timeline）。

## nginx代理后端服务

nginx 服务器及端口 127.0.0.1:80
后端服务:127.0.0.1:8080
测试url:http://127.0.0.1:80/day06api/api/abc

### A.配置

nginx配置如下：

```bash
location /day06api/ {
   proxy_pass http://127.0.0.1:8080/;
}
```

实际访问的端口服务：http://127.0.0.1:8080/api/abc

### B.配置

```bash
location /day06api {
   proxy_pass http://127.0.0.1:8080/;
}
```

实际访问的端口服务：http://127.0.0.1:8080//api/abc

### C.配置

```bash
location /day06api/ {
   proxy_pass http://127.0.0.1:8080;
}
```

实际访问的端口服务：http://127.0.0.1:8080/day06api/api/abc

### D.配置

```bash
location /day06api {
   proxy_pass http://127.0.0.1:8080;
}
```

实际访问的端口服务：http://127.0.0.1:8080/day06api/api/abc

### E.配置

```bash
location /day06api/ {
   proxy_pass http://127.0.0.1:8080/server/;
}
```

实际访问的端口服务：http://127.0.0.1:8080/server/api/abc

### F.配置

```bash
location /day06api {
   proxy_pass http://127.0.0.1:8080/server/;
}
```

实际访问的端口服务：http://127.0.0.1:8080/server//api/abc

### G.配置

```bash
location /day06api/ {
   proxy_pass http://127.0.0.1:8080/server;
}
```

实际访问的端口服务：http://127.0.0.1:8080/serverapi/abc

### H.配置

```bash
location /day06api {
   proxy_pass http://127.0.0.1:8080/server;
}
```

实际访问的端口服务：http://127.0.0.1:8080/server/api/abc