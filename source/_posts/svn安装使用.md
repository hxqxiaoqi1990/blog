---
title: svn安装使用
tags: [svn]
categories: 中间件
date: 2019-11-01 20:45:00
---


# 介绍
**Subversion(SVN):** 是一个开源的版本控制系統, 也就是说 Subversion 管理着随时间改变的数据。 这些数据放置在一个中央资料档案库(repository) 中。 这个档案库很像一个普通的文件服务器, 不过它会记住每一次文件的变动。 这样你就可以把档案恢复到旧的版本, 或是浏览文件的变动历史。

# 安装

``` bash
# 安装
yum -y install subversion

# 查看版本
svn --version
```
**创建版本库**

``` bash
# 手动新建版本库目录
mkdir /data/svn

# 利用svn命令创建版本库
svnadmin create /opt/svn/test

# 查看创建生成的文件，其中conf目录下：authz：给用户或组设置权限，passwd：给用户设置密码，svnserve.conf：svn配置文件。
# 数据主要存在db下，格式为fsfs。
ls /opt/svn/test
```
**配置用户**

``` bash
cd  /opt/svn/test/conf

# 创建用户
vim passwd
	huang = 123456
	xiao = 123123

# 给用户授权，[/]：代表/opt/svn/目录
vim authz
	[/]	
	huang = rw
	xiao = r
	
# 配置版本库，取消以下注释。
# anon-access: 匿名用户权限
# auth-access: 登录用户权限
# password-db：密码文件位置
# authz-db: 指定权限配置文件名
# realm: 指定版本库的认证域，即在登录时提示的认证域名称。若两个版本库的 认证域相同，建议使用相同的用户名口令数据文件。 默认值：一个UUID(Universal Unique IDentifier，全局唯一标示)。
vim svnserve.conf
	anon-access = read
	auth-access = write
	password-db = passwd
	realm = My First Repository
```
**启动**

``` bash
# 使用命令svnserve启动服务，-d：后台运行，-r：指定版本库目录如果不指定端口，默认为3690
# svnserve -d -r 目录 --listen-port 端口号
svnserve -d -r /opt/svn/
```

# 使用
下载svn客户端：[TortoiseSVN](https://mirrors.xtom.com.hk/osdn//storage/g/t/to/tortoisesvn/1.13.0/Application/TortoiseSVN-1.13.0.28678-x64-svn-1.13.0.msi)

 1. 下载window版，安装客户端。
 2. 随便创建一个目录，右击该目录，可查看到svn选项，即安装完成。

 **拉去svn版本库test**
 1.选择创建的目录，右击-选择SVN Checkout
 2.在跳出的任务框中输入：svn://10.10.10.10/test 拉取代码

**提交代码**
在已经拉取的目录中，创建一些文件，右击-选择SVN commit，可以输入提交说明和文件

**更新代码**
在已经拉取的目录中，右击-选择SVN UPdate

# 备份恢复
**备份**

``` bash
svnadmin dump /opt/svn/test > test.svn.bak
```

**恢复**

``` bash
# #创建一个新的版本库,用户需要重新配置
svnadmin   create /opt/svn/test   

# 现还原完全备份
svnadmin   load    /opt/svn/test     <  test.svn.bak      
```