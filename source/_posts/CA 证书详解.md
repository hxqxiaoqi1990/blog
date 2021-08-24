---
title: CA 证书详解
tags: [CA]
categories: linux基础
date: 2020-04-21 20:00:00
---

# 介绍

[博客链接](https://blog.csdn.net/lk2684753/article/details/100160856) 该博客非常详细的讲解了`数字证书及CA`

**简单说明：**

- CA：权威认证机构
- PKI：一种标准
- 编码格式
  - X.509 DER(Distinguished Encoding Rules)编码，后缀为：.der .cer .crt
  - X.509 BASE64编码(PEM格式)，后缀为：.pem .cer .crt
- server.key：私钥，正式环境中，私钥一般都有权威认证机构提供。
- server.csr：证书请求，内容为公司或个人信息，配合私钥可生成证书
- server.crt：由私钥签名后生成的证书

## 使用openssl生成https证书

```bash
# 使用openssl工具生成一个RSA私钥
openssl genrsa -des3 -out server.key 2048

# 生成CSR（证书签名请求）说明：需要依次输入国家，地区，城市，组织，组织单位，Common Name和Email。其中Common Name，可以写自己的名字或者域名，如果要支持https，Common Name应该与域名保持一致，否则会引起浏览器警告。可以将证书发送给证书颁发机构（CA），CA验证过请求者的身份之后，会出具签名证书，需要花钱。另外，如果只是内部或者测试需求，也可以使用OpenSSL实现自签名。
openssl req -new -key server.key -out server.csr

# 删除密钥中的密码，说明：如果不删除密码，在应用加载的时候会出现输入密码进行验证的情况，不方便自动化部署。
openssl rsa -in server.key -out server.key

# 生成自签名证书，内部或者测试使用，只要忽略证书提醒就可以了。
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

# 生成pem格式的公钥，有些服务，需要有pem格式的证书才能正常加载，可以用下面的命令：
openssl x509 -in server.crt -out server.pem -outform PEM

# 查看证书有效期
openssl x509 -in server.crt -noout -dates

# 查看证书信息
openssl x509 -in server.crt -noout -text
```

