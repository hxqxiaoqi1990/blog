---
title: nginx重写与重定向
tags: [nginx]
categories: linux基础
date: 2020-03-18 21:00:00
---

# 介绍

重写与重定向功能是现在大多数Web服务器都支持的一项功能，相对于其他产品而言,Nginx中的rewrite模块提供的功能在配置上更加的灵活自由,可定制性非常的高。它的实现方式也非常的简单,只需要通过rewrite指令根据Nginx提供的全局变量或自定义的变量,结合正则表达式以及进一步处理的标识就可以完成URL重写或重定向。

rewrite参数：

- `last` 本条规则匹配完成后继续向下匹配新的location URI规则
- `break` 本条规则匹配完成后终止，不在匹配任何规则
- `redirect` 返回302临时重定向
- `permanent` 返回301永久重定向

注：当flag的值为last或break时，标识当前的设置为重写，当flag的值为redirect或permanent时表示重定向。

# rewrite 重写

```bash
server {
	listen 80;
	server_name localhost;
	index index.html index.htm;
	root html;
	if (!-e $request_filename){
		rewrite "^/.*" /default/default.html break;
	}
}
```

上述第6行配置,通过if指令判断访问不到用户请求的文件或目录时,执行第7行指令。其中,!-e用于判断不存在指定的文件或目录时，执行if块内的语句。内置变量$request_ filename 表示当前请求的文件路径;^/.*用于匹配当前网站下的所有请求，/default/default.html用于替换符合指定规则的请求。

值得一提的是,if指令根据给定的条件进行判断,如果判断结果为true,则执行大括号“{}”内的指令。当判断条件仅是一个变量时,如果值为空或任何以0开头的字符串都会当做false,不再执行大括号内的指令。

在根目录下创建default目录，并在该目录下编写default.html文件：

```html
<h1>Welcome to html/default/default.html!</h>
```

访问一个不存在的文件：`localhost/ffff.df`

break和last标识的区别：

在使用rewrite实现重写时,需要注意flag可选参数值break和last的区别，前者在rewrite指令匹配成功后就不再进行匹配，而后者在rewrite后会根据rewrite匹配的规则重新发起一个请求继续进行匹配。为了明确地看到两者的区别，下面通过二个案例进行验证。

```bash
server {
	listen 80;
	server_name localhost;
	root html;
	location /break/ {
		rewrite ^/break/(.*) /test/$1 break;
		echo "break page";
	}
	location /last/ {
		rewrite ^/last/(.*) /test/$1 last;
		echo "last page";
	}
	location /test/ {
		echo "test page";
	}
}
```

当用户的请求符合第5行location 规则时,执行第6行rewrite指令后，根据break的指定,继续执行第7行配置。而当用户的请求符合第9行location规则时,执行第10行rewrite指令后,根据last的指定,按照匹配到的替换算法发起一个新的请求http://test. ng. test/ test/ last. html,最终匹配到第13行的location规则,然后执行第14行的echo输出语句。

因此,在实际使用rewrite配置重写时，要根据实际情况选择合适的flag可选参数值，，否则会造成与预期不一样的结果。


# rewrite重定向

```bash
server {
	listen 80;
	server_name localhost;
	root html;
	set $name $1;
	rewrite ^/img-([0-9]+).jpg$/img/$name.jpg permanent;
}
# 域名重写
server {
	listen 80;
	server_name localhost;
	root html;
    rewrite ^/(.*)$ https://winbb.com/$1 permanent;
}
```

上述第5行配置，利用set指令为变量$name赋值，$1表示符合正则表达式第一个子模式的值，如第6行中的子模式([0-9]+ )匹配到的值,可以是2、45等由一个或多个数字组成的字符串。第6行用于在用户请求“http://localhost/img-数字. jpg”时，重定向到“http://localhost/img/数字jpg"。

接下来，在localhost的网站目录中创建一个用于存放图片的img目录，然后在该目录中保存一个文件名为2.jpg的图片。为了测试当前配置是否成功，通过浏览器访问http://localhost/img-2. jpg，会跳转到http://localhost/img/2.jpg。

需要注意的是，redirect和permanent在使用时有一定的区别，前者返回的HTTP状态码是302(临时重定向),使得搜索引擎在抓取新内容的同时保留旧的网址,后者返回的HTTP状态码是，301(永久重定向)会让搜索引擎在抓取新内容的同时也将旧的网址永久替换为重定向之后的网址。