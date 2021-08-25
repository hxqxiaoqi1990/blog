---
title: git基础使用
tags: git
categories: 中间件
date: 2019-06-03 15:27:43
---

# git是什么
Git 是一个开源的分布式版本控制系统，用于敏捷高效地处理任何或小或大的项目。
Git 是 Linus Torvalds 为了帮助管理 Linux 内核开发而开发的一个开放源码的版本控制软件。
Git 与常用的版本控制工具 CVS, Subversion 等不同，它采用了分布式版本库的方式，不必服务器端软件支持。

# git安装
linux安装

``` bash
$ yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
$ yum -y install git-core
$ git --version
```
windows安装

官网下载:https://gitforwindows.org/ 下载后直接安装

# git基本概念
我们先来理解下Git 工作区、暂存区和版本库概念:

<font color=DarkTurquoise>**工作区：**</font>就是你在电脑里能看到的目录。
<font color=DarkTurquoise>**暂存区：**</font>英文叫stage, 或index。一般存放在 ".git目录下" 下的index文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）。
<font color=DarkTurquoise>**版本库：**</font>工作区有一个隐藏目录.git，这个不算工作区，而是Git的版本库。

当对工作区修改（或新增）的文件执行 "git add" 命令时，暂存区的目录树被更新，同时工作区修改（或新增）的文件内容被写入到对象库中的一个新的对象中，而该对象的ID被记录在暂存区的文件索引中。

当执行提交操作（git commit）时，暂存区的目录树写到版本库（对象库）中，master 分支会做相应的更新。即 master 指向的目录树就是提交时暂存区的目录树。

当执行 "git reset HEAD" 命令时，暂存区的目录树会被重写，被 master 分支指向的目录树所替换，但是工作区不受影响。

当执行 "git rm --cached file" 命令时，会直接从暂存区删除文件，工作区则不做出改变。

当执行 "git checkout ." 或者 "git checkout -- file" 命令时，会用暂存区全部或指定的文件替换工作区的文件。这个操作很危险，会清除工作区中未添加到暂存区的改动。

当执行 "git checkout HEAD ." 或者 "git checkout HEAD file" 命令时，会用 HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和以及工作区中的文件。这个命令也是极具危险性的，因为不但会清除工作区中未提交的改动，也会清除暂存区中未提交的改动。

# git 基本命令
<font color=DarkTurquoise>**git config**</font>
配置个人的用户名称和电子邮件地址,新的设定保存在当前项目的 .git/config 文件里。

``` bash
$ git config --global user.name "runoob"
$ git config --global user.email test@runoob.com
```
要检查已有的配置信息，可以使用 git config --list 命令,这些配置我们也可以在 ~/.gitconfig 或 /etc/gitconfig 看到

``` bash
$ git config --list
http.postbuffer=2M
user.name=runoob
user.email=test@runoob.com
```
<font color=DarkTurquoise>**git init**</font>
该命令执行完后会在当前目录生成一个 .git 目录,该目录包含了资源的所有元数据，其他的项目目录保持不变（不像 SVN 会在每个子目录生成 .svn 目录，Git 只在仓库的根目录生成 .git 目录）。

``` bash
$ git init
$ git init "指定目录"
```
<font color=DarkTurquoise>**git clone**</font>
我们使用 git clone 从现有 Git 仓库中拷贝项目
参数说明：
repo:Git 仓库。
directory:本地目录。
``` bash
$ git clone <repo>
```
如果我们需要克隆到指定的目录，可以使用以下命令格式：

``` bash
$ git clone <repo> <directory>
```
如果要指定分支，使用选项-b：

``` bash
$ git clone -b master <repo> <directory>
```
<font color=DarkTurquoise>**git add**</font>
可将该文件添加到缓存

``` bash
$ git add "指定文件或目录"
$ git add .
```
<font color=DarkTurquoise>**git status**</font>
git status 以查看在你上次提交之后是否有修改。

``` bash
$ git status
```
 <font color=DarkTurquoise>**git diff**</font>
执行 git diff 来查看执行 git status 的结果的详细信息。
git diff 命令显示已写入缓存与已修改但尚未写入缓存的改动的区别。git diff 有两个主要的应用场景。
尚未缓存的改动：git diff
查看已缓存的改动： git diff --cached
查看已缓存的与未缓存的所有改动：git diff HEAD
显示摘要而非整个 diff：git diff --stat

 <font color=DarkTurquoise>**git commit**</font>
 使用 git add 命令将想要快照的内容写入缓存区， 而执行 git commit 将缓存区内容添加到仓库中。
 m: 提交更新说明
 a: add的使用

``` bash
$ git commit -am
```
 <font color=DarkTurquoise>**git reset HEAD**</font>
用于取消已缓存的内容。

``` bash
$ git reset HEAD hello.php 
```
 <font color=DarkTurquoise>**git rm**</font>
 从工作目录中手工删除文件

``` bash
git rm <file>
```
如果删除之前修改过并且已经放到暂存区域的话，则必须要用强制删除选项 -f

``` bash
git rm -f <file>
```
如果把文件从暂存区域移除，但仍然希望保留在当前工作目录中，换句话说，仅是从跟踪清单中删除，使用 --cached 选项即可

``` bash
git rm --cached <file>
```
 <font color=DarkTurquoise>**git mv**</font>
git mv 命令用于移动或重命名一个文件、目录、软连接。

 <font color=DarkTurquoise>**git branch (branchname)**</font>
 创建分支

 <font color=DarkTurquoise>**git checkout (branchname)**</font>
切换分支命令

 <font color=DarkTurquoise>**git merge (branchname)**</font>
合并分支命令

 <font color=DarkTurquoise>**git branch**</font>
列出分支基本命令

 <font color=DarkTurquoise>**git branch -d**</font>
删除分支命令

 <font color=DarkTurquoise>**git log**</font>
列出历史提交记录

 <font color=DarkTurquoise>**git log --oneline**</font>
 历史记录的简洁的版本

<font color=DarkTurquoise>**git tag** </font>
查看所有标签

<font color=DarkTurquoise>**git tag -a v1.0 -m "标签说明"**</font>
给最新一次提交打上（HEAD）"v1.0"的标签

<font color=DarkTurquoise>**git tag -a v1.0 "commit名"**</font>
之前忘记,后面添加标签

<font color=DarkTurquoise>**git push origin v1.5**</font>
默认上传不上传标签,需要指定

<font color=DarkTurquoise>**git remote add [shortname] [url]**</font>
添加远程库

<font color=DarkTurquoise>**git remote rm [别名]**</font>
删除远程仓库

<font color=DarkTurquoise>**git remote**</font>
<font color=DarkTurquoise>**git remote -v**</font>
查看当前配置有哪些远程仓库

<font color=DarkTurquoise>**git branch -vv**</font>

查看远程与本地查看关系

<font color=DarkTurquoise>**git fetch**</font>
从远程仓库下载新分支与数据
该命令执行完后需要执行git merge 远程分支到你所在的分支。

<font color=DarkTurquoise>**git push [alias] [branch]**</font>
以上命令将你的 [branch] 分支推送成为 [alias] 远程仓库上的 [branch] 分支

# 回滚

```bash
# 查看分支 
git branch

# 切换分支 
git checkout master

# 查看标签(tag版本) 
git tag

# 查看某个标签的详情 
git show v2.22.0

commit d53dcc2287899e95cfd44a294ca3e5068e63022b

# 通过commit的id回退
git reset --hard fb479960c0cec5549463ae123d70bdd72ccf6be7

# 查看状态
git status

# 提交
git push origin master

# 或者加入-f参数，强制提交，远程端将强制跟新到reset版本
git push -f origin master
```

