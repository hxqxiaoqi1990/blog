---
title: nginx之proxy_pass规则
tags: [nginx]
categories: linux基础
date: 2020-06-04 18:40:43
---

# 介绍

在两个模块中，两个`proxy_pass`都是用来做后端代理的指令。
`ngx_stream_proxy_module`模块的`proxy_pass`指令只能在server段使用使用, 只需要提供域名或ip地址和端口。可以理解为端口转发，可以是tcp端口，也可以是udp端口。
`ngx_http_proxy_module`模块的`proxy_pass`指令需要在location段，location中的if段，limit_except段中使用，处理需要提供域名或ip地址和端口外，还需要提供协议，如"http"或"https"，还有一个可选的uri可以配置。

# 实验

在nginx中配置proxy_pass代理转发时，如果在proxy_pass后面的url加/，表示绝对根路径；如果没有/，表示相对路径，把匹配的路径部分也给代理走。

假设下面四种情况分别用 http://192.168.1.1/proxy/test.html 进行访问。

**第一种：**

代理到URL：http://127.0.0.1/test.html

```bash
location /proxy/ {
proxy_pass http://127.0.0.1/;
}
```

**第二种**（相对于第一种，最后少一个 / ）

代理到URL：http://127.0.0.1/proxy/test.html

```bash
location /proxy/ {
proxy_pass http://127.0.0.1;
}
```

**第三种：**

代理到URL：http://127.0.0.1/aaa/test.html

```bash
location /proxy/ {
proxy_pass http://127.0.0.1/aaa/;
}
```

**第四种**（相对于第三种，最后少一个 / ）

代理到URL：http://127.0.0.1/aaatest.html

```bash
location /proxy/ {
proxy_pass http://127.0.0.1/aaa;
}
```

**proxy_pass后，后端服务器的url(request_uri)情况分析**

```bash
server {
    listen      80;
    server_name www.test.com;
 
    # 情形A
    # 访问 http://www.test.com/testa/aaaa
    # 后端的request_uri为: /testa/aaaa
    location ^~ /testa/ {
        proxy_pass http://127.0.0.1:8801;
    }
    
    # 情形B
    # 访问 http://www.test.com/testb/bbbb
    # 后端的request_uri为: /bbbb
    location ^~ /testb/ {
        proxy_pass http://127.0.0.1:8801/;
    }
 
    # 情形C
    # 下面这段location是正确的
    location ~ /testc {
        proxy_pass http://127.0.0.1:8801;
    }
 
    # 情形D
    # 下面这段location是错误的
    #
    # nginx -t 时，会报如下错误: 
    #
    # nginx: [emerg] "proxy_pass" cannot have URI part in location given by regular 
    # expression, or inside named location, or inside "if" statement, or inside 
    # "limit_except" block in /opt/app/nginx/conf/vhost/test.conf:17
    # 
    # 当location为正则表达式时，proxy_pass 不能包含URI部分。本例中包含了"/"
    location ~ /testd {
        proxy_pass http://127.0.0.1:8801/;   # 记住，location为正则表达式时，不能这样写！！！
    }
 
    # 情形E
    # 访问 http://www.test.com/ccc/bbbb
    # 后端的request_uri为: /aaa/ccc/bbbb
    location /ccc/ {
        proxy_pass http://127.0.0.1:8801/aaa$request_uri;
    }
 
    # 情形F
    # 访问 http://www.test.com/namea/ddd
    # 后端的request_uri为: /yongfu?namea=ddd
    location /namea/ {
        rewrite    /namea/([^/]+) /yongfu?namea=$1 break;
        proxy_pass http://127.0.0.1:8801;
    }
 
    # 情形G
    # 访问 http://www.test.com/nameb/eee
    # 后端的request_uri为: /yongfu?nameb=eee
    location /nameb/ {
        rewrite    /nameb/([^/]+) /yongfu?nameb=$1 break;
        proxy_pass http://127.0.0.1:8801/;
    }
}
```

