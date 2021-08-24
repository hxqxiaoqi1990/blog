---
title: hexo主题多终端管理
tags: [hexo]
categories: 随笔
date: 2019-06-02 21:29:43
---

# 搭建目的
很多人可能家里一台笔记本，公司一个台式机，想两个同时管理博客，同时达到备份的博客主题、文章、配置的目的。 
下面就介绍一下用gitee来备份博客并同步博客。

# 搭建步骤
以下步骤统一用A和B电脑举例，A电脑为已经部署hexo客户端，B为另一台没有任何部署的电脑，如果之前都没部署过，请客之前教程[heox主题搭建](https://hxqxiaoqi.gitee.io/2019/06/02/heox%E4%B8%BB%E9%A2%98%E6%90%AD%E5%BB%BA/)和[hexo主题部署到gitee教程](https://hxqxiaoqi.gitee.io/2019/06/02/hexo%E4%B8%BB%E9%A2%98%E9%83%A8%E7%BD%B2%E5%88%B0gitee%E6%95%99%E7%A8%8B/)两个教程

 1. 新建一个gitee分支hexo，用于上传A电脑的hexo代码
 2. A电脑上传hexo代码到gitee上
 3. B电脑安装好环境，克隆gitee分支代码
 4. 常用命令

## 1. 新建一个gitee分支hexo
登陆gitee，新建一个分支hexo，用于上传A电脑的hexo代码，master为存放hexo发布的代码，新建hexo分支为hexo客户端代码，我们就是下客户端代码到本机，之后写文章或样式发布到master。
## 2. A电脑上传hexo代码到gitee上

进入博客根目录文件夹下，找到.gitignore文件，在最后增加两行内容/.deploy_git和/public
然后运行git bash here，初始化仓库，执行：

``` bash
git init
```
如果提示已经有初始化，需要删除目录下所有.git文件，再执行初始化

添加远程仓库，origin为本地远程仓库名，后面是gitee仓库地址：

``` bash
git remote add origin https://gitee.com/xxx/xxx.git 
```

如果远程仓库已存在文件，需要执行下面命令：
``` bash
# 把远程仓库和本地同步，消除差异
git pull origin master --allow-unrelated-histories
```

添加目录下所有文件到暂存区，执行：

``` bash
git add . 
```
提交并添加更新说明到版本库，执行：

``` bash
git commit -m "更新说明"
```
推送更新到远程仓库hexo分支，执行：

``` bash
git push -u origin hexo
```
如果仓库已经有代码会报错：

``` bash
xu:QProj xiaokai$ git push origin master
To https://gitee.com/XXXXX.git
 ! [rejected]        master -> master (non-fast-forward)
error: failed to push some refs to 'https://gitee.com/XXXXX.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```
执行以下命令：

``` bash
git fetch		#从仓库更新代码到本地暂存区
git merge origin/hexo	#暂存区更新代码合并
```
执行git merge又报错：

``` bash
git merge
fatal: refusing to merge unrelated histories
```
接着执行：

``` bash
git pull origin hexo --allow-unrelated-histories 
```
然后继续git merge,依然有问题：

``` bash
fatal: You have not concluded your merge (MERGE_HEAD exists).
Please, commit your changes before you merge.
```
这个就好处理了，是我们没有提交当前的变化， git add .,git commit -am "提交信息"

然后输入git pull,显示如下:

``` bash
Already up-to-date.
```
最后就可以执行:

``` bash
git push origin hexo 
```
有时还会报错，执行以下命令上传代码：

``` bash
git push origin HEAD:hexo
```
现在可以去gitee上查看hexo分支，已经有hexo客户端代码

## 3. B电脑安装好环境，克隆gitee分支代码

在B电脑上同样先安装好node、git、ssh、hexo，安装好插件，新建目录，进入git bash here，执行以下命令克隆代码，会在当前目生成仓库同名目录，可以在这个目录下更新文章，发布博客到gitee。

``` bash
git clone -b hexo https://gitee.com/xxx/xxx.git 
```

## 4.  常用命令

``` bash
git pull 				#同步更新
hexo new post "新建文章" 		#简写形式 hexo n "新建文章"
hexo clean 				#清除旧的public文件夹
hexo g 					#生成静态文件 简写形式 hexo g
hexo d					#发布到github上 简写形式 hexo d
git add . 				#添加更改文件到缓存区
git commit -m "更新说明" 		#提交到本地仓库
git push -u origin HEAD:hexo 		#推送到远程仓库进行备份
```


