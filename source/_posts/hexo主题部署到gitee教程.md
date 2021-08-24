---
title: hexo主题部署到gitee教程
tags: hexo
categories: 随笔
date: 2019-06-01 18:11:43
---

# 使用gitee原因

目前国内访问GitHub速度慢，还可能被墙，所以Gitee来构建个人博客。Gitee类似国内版的GitHub，访问速度有保证。

# 安装前提

 1. 已经安装本地hexo主题，如果还未安装，请查看 [hexo搭建教程](https://hxqxiaoqi.gitee.io/2019/06/02/heox主题搭建/)
 2. 已经注册gitee账号

# 安装步骤

 1. 创建gitee仓库，获取仓库地址
 2. 启用gitee Pages服务，获取url 地址
 3. 修改hexo根目录config.xml配置
 4. 发布到gitee上
 5. 更新gitee Pages服务

## 1. 创建gitee仓库，获取仓库地址

 1. 登陆gitee，点击创建仓库
 2. 仓库名称最好取跟个性地址一致的名称，后期完成部署就生成的url可以不需要指定二级目录，博客样式也不会出错
 3. 选择公开
 4. 其他默认，点击创建
 5. 在克隆/下载的标签上可以看到你的仓库地址（例：https://gitee.com/xxx/xxx.git ）

## 2. 启用gitee Pages服务，获取url 地址

 1. 点击-服务，选择gitee Pages
 2. 勾选-强制使用https
 3. 点击-启动
 4. 等一会儿就会生成你的url，也就是完成部署后的博客访问地址，（例：https://xxx.gitee.io ），如果之前仓库名称没有与自己的个性地址一致，生成的地址应该是（https://xxx.gitee.io/xxx ）

## 3. 修改hexo根目录config.xml配置
url地址，也就是域名地址，如果url是（ https://xxx.gitee.io/xxx ）这种格式，root设置二级目录名称
修改：
``` xml
url: https://xxx.gitee.io
root: /
```
仓库地址，用于上传hexo主题的地址，分支选择，如果没有修改，默认是master
添加：
``` xml
deploy:
  type: git
  repository: https://gitee.com/xxx/xxx.git
  branch: master
```
注意：冒号后面需要空格

## 4. 发布到gitee上

安装自动部署发布工具
``` bash
npm install hexo-deployer-git --save
```


清除本地hexo缓存

``` bash
hexo clean
```
编译

``` bash
hexo g
```

## 码云认证

``` bash
git config --global user.name "用户名"
git config --global user.email "邮箱"
```

``` bash
# 第一次需要再输入码云账户和密码
hexo d
```

## 5. 更新gitee Pages服务

 1. 点击服务，选择gittee Pages，点击更新
 2. 访问：https://xxx.gitee.io
 3. 每次hexo发布，都需要更新gittee Pages服务。
 4. 至此，博客发布成功