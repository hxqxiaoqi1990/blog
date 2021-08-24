---
title: nginx之php-fpm
tags: [nginx]
categories: linux基础
date: 2020-06-04 17:40:43
---

# 介绍

nginx 是一个高性能的http服务器和反向代理服务器。即nginx可以作为一个HTTP服务器进行网站的发布处理，也可以作为一个反向代理服务器进行负载均衡。但需要注意的是：nginx本身并不会对php文件进行解析。对PHP页面的请求将会被nginx交给FastCGI进程监听的IP地址及端口，由php-fpm(第三方的fastcgi进程管理器)作为动态解析服务器处理，最后将处理结果再返回给nginx。即nginx通过反向代理功能将动态请求转向后端php-fpm，从而实现对PHP的解析支持，这就是Nginx实现PHP动态解析的基本原理。 

- Nginx 是非阻塞IO & IO复用模型，通过操作系统提供的类似 epoll 的功能，可以在一个线程里处理多个客户端的请求。Nginx 的进程就是线程，即每个进程里只有一个线程，但这一个线程可以服务多个客户端。
- PHP-FPM 是阻塞的单线程模型，pm.max_children 指定的是最大的进程数量，pm.max_requests 指定的是每个进程处理多少个请求后重启(因为 PHP 偶尔会有内存泄漏，所以需要重启)。PHP-FPM 的每个进程也只有一个线程，但是一个进程同时只能服务一个客户端。
- fastCGI ：为了解决不同的语言解释器(如php、python解释器)与webserver的通信，于是出现了cgi协议。只要你按照cgi协议去编写程序，就能实现语言解释器与webwerver的通信。如php-cgi程序。但是webserver每收到一个请求，都会去fork一个cgi进程，请求结束再kill掉这个进程。这样有10000个请求，就需要fork、kill php-cgi进程10000次。 fastcgi是cgi的改良版本。fast-cgi每次处理完请求后，不会kill掉这个进程，而是保留这个进程，使这个进程可以一次处理多个请求。大大提高了效率。

# php-fpm安装

```bash
# 安装
yum -y remove php*
yum install epel-release -y
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
yum -y install php72w php72w-cli php72w-fpm php72w-common php72w-devel php72w-embedded php72w-gd php72w-mbstring php72w-mysqlnd php72w-opcache php72w-pdo php72w-xml


# 修改/etc/php-fpm.d/www.conf，解决访问403问题
security.limit_extensions = .php .php3 .php4 .php5 .php7 .html .js .css .jpg .jpeg .gif .png .htm

# 启动
systemctl enable php-fpm.service
systemctl start php-fpm.service
```



# nginx配置php

```bash
    server {
        listen       86;
        server_name  localhost;
        root /data/winbb;
        location / {
            index  index.php index.html index.htm;
            try_files $uri $uri/ /index.php?$args;
        }

        #php 的单独写一个匹配location,这样让静态文件走location /{}
        location ~* \.php {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include  fastcgi_params;
        }
}
```

如上配置，当一个http请求到来时，被处理的过程如下：

1. http请求到来后，通过server全局块里监听的端口号，匹配到相应server。然后接下来进行location路径的匹配。
2. 首先匹配到location / ，在这个匹配规则中，通过try_files 先在root目录(/home/leimengyao/api/app/htdocs)下查找是否有$uri文件；没有匹配到，然后再查找root目录下是否有$uri/目录；同样没有匹配到，则匹配最后一项/index.php?$args，即发出一个"内部子请求"，也就相当于nginx发起了一个http请求到http://10.94.120.124:8000/index.php?c=1&d=4
3. 这个子请求会被location ~ \.php${ ... }catch住，也就是进入 FastCGI 的处理程序（nginx需要通过FastCGI模块配置，将相关php参数传递给php-fpm处理。在该项中设置了fastcgi_pass相关参数，将用户请求的资源发给php-fpm进行解析，这里涉及到nginx FastCGI模块的相关配置语法下文会介绍）。而具体的 URI 及参数是在 REQUEST_URI 中传递给 FastCGI 和 WordPress 程序的，因此不受 URI 变化的影响！！！！。