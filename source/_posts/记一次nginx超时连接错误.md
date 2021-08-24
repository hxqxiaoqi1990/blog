---
title: 记一次nginx超时连接错误
tags: [nginx]
categories: linux基础
date: 2019-06-07 17:27:43
---


# 连接错误代码

``` bash
2019/06/05 17:15:05 [error] 10560#0: *7830037 connect() failed (110: Connection timed out) while connecting to 
upstream, client: 192.168.1.100, server: 123.abc.com, request: "OPTIONS //tax/school/login/resetToken/101 HTTP/1.1",
upstream: "http://192.168.1.20//aaa/bbb/login/resetToken/101", host: "123.abc.com", referrer: "http://abc.qwe.com/"
```
# 分析
网络上有很多的解决方法，但实际上都是根据自身遇到的问题而找到对应的服务解决，都没有讲为什么这个报错，报错的原因，其实仔细分析nginx报错的代码，也能很好的了解到你的问题所在。
分析报错的代码：

 1. Connection timed out：按字面上的意思就可以知道，这是连接超时，也就是连接服务时，没有在规定时间内取得回应，才会报错。
 2. connecting to upstream：如果了解nginx负载均衡，那你应该知道upstream是nginx负载均衡的设置，也就是问题出在负载均衡的连接上。
 3. client: 192.168.1.100, server: 123.abc.com：客户端 192.168.1.100 请求服务端 123.abc.com 地址，返回错误。
 4. upstream: "http://192.168.1.20//aaa/bbb/login/resetToken/101" ： 这句话比较重要，负载到http://192.168.1.20//aaa/bbb/login/resetToken/101 这个地址是连接出问题。

# 解决思路
 显然，上面的报错代码可以看出，问题是出在负载均衡地址的http://192.168.1.20//aaa/bbb/login/resetToken/101 上，定位问题后，接下去就比较好解决了。nginx负载无法访问到192.168.1.20服务器上的服务：

 1. 服务挂了，可以登陆服务器上查看
 2. 服务运行错误，可以查看服务的日志
 3. 防火墙问题，查看对应的Ip和端口是否开放
 4. 网络传输问题，tcpdump抓取数据查看是否有传输数据

 