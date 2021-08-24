---
title: python2之requests模块 
tags: [python]
categories: linux基础
date: 2019-11-02 21:17:00
---


# 介绍
Requests 是使用 Apache2 Licensed 许可证的 基于Python开发的HTTP 库，其在Python内置模块的基础上进行了高度的封装，从而使得Pythoner进行网络请求时，变得美好了许多，使用Requests可以轻而易举的完成浏览器可有的任何操作。requests可以模拟浏览器的请求，比起之前用到的urllib，requests模块的api更加便捷（其本质就是封装了urllib3）

# 简单示例
## get示例
**获取网页内容**

``` python
import requests
response = requests.get('http://httpbin.org/')
print response.text
```

**获取状态码**

``` python
import requests
response = requests.get('http://httpbin.org/').status_code
print response
```
**打印json结果**
``` python
import requests
import json

response = requests.get("http://httpbin.org/get")
print(type(response.text))
print(type(response.json()))

print(json.loads(response.text))
print(response.json())
```
**添加请求头**

``` python
# -*- coding: UTF-8 -*-
# 不加请求头，有些服务会直接报500的错误
import requests
response = requests.get("https://www.zhihu.com/explore")
print(response.text)

# 添加请求头
import requests
headers = {
    'User-Agent': 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36'
}
response = requests.get("https://www.zhihu.com/explore", headers=headers)
print(response.text)
```



