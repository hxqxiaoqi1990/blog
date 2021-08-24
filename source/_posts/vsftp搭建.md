---
title: vsftp搭建
tags: [ftp]
categories: linux基础
date: 2019-09-01 18:27:43
---

# 服务说明

ftp是一种古老的明文传输协议，因为其明文传输的特性，也出现过很多重大的危机，进而逐渐演变为支持加密传输的VSFTP（very security FTP），而CentOS默认自带的FTP就为VSFTP。为了避免干扰，实验前请关闭Selinux和IPtables。

# 安装前准备：

``` bash
#关闭防火墙
iptables -L
iptables -F
systemctl stop firewalld
systemctl disable firewalld

#永久关闭selinux
vim /etc/selinux/config	#修改disable

#临时关闭selinux
setsebool -P ftpd_disable_trans 1
 setenforce 0
```



# 安装步骤：

``` bash
#安装vsftp服务
yum -y install vsftpd* pam* db4*

#添加虚拟用户，第一行用户，第二行密码，以此类推
vim /etc/vsftpd/vsftpd.user	

#生成虚拟账户数据表
db_load -T -t hash -f vsftpd.user vsftpd.db

#创建本地用户，并指定家目录
useradd -d /var/ftproot -s /sbin/nologin virtual

#修改校验文件，用于验证ftp虚拟用户
cd /etc/pam.d/
cp -a vsftpd vsftpd.pam
vim vsftpd.pam
添加：
auth required pam_userdb.so  db=/etc/vsftpd/vsftpd
account required pam_userdb.so  db=/etc/vsftpd/vsftpd

#修改配置文件
vim /etc/vsftpd/vsftpd.conf
	修改与添加：
	anonymous_enable=NO
	local_enable=YES
	write_enable=YES
	local_umask=022
	dirmessage_enable=YES
	xferlog_enable=YES
	connect_from_port_20=YES
	xferlog_std_format=YES
	listen=NO
	listen_ipv6=YES
	pam_service_name=vsftpd.pam
	userlist_enable=YES
	tcp_wrappers=YES
	guest_enable=YES
	guest_username=virtual
	user_config_dir=/etc/vsftpd/dir
	allow_writeable_chroot=YES		#新版必须添加否则取消目录w权限

#根据配置文件user_config_dir配置，指定虚拟账户配置文件位置，如果没有目录，需要创建
mkdir /etc/vsftpd/dir
cd /etc/vsftpd/dir

#创建虚拟用户配置文件,添加单独虚拟用户权限
vim aaa
	#指定虚拟用户家目录
	local_root=/share/aaa	
	#允许匿名用户浏览，下载文件,默认没有这一项，只有在虚拟用户的配置文件里才有用
	anon_world_readable_only=NO
	#允许匿名用户上传
	anon_upload_enable=YES
	#允许匿名用户上传/建立目录
	anon_mkdir_write_enable=YES
	#默认没有这一项,允许匿名用户具有建立目录，上传之外的权限，如重命名，删除
	anon_other_write_enable=YES

#本地创建虚拟用户家目录，并赋予权限
mkdir -p /share/aaa
chown virtual.virtual /share/ -R
chmod 770 /share/ -R
```
# 添加虚拟用方法：

``` bash
#添加用户
vim vsftpd.user	

#重新校验
db_load -T -t hash -f vsftpd.user vsftpd.db
systemctl restart vsftpd
```

# 启动与开机启动

``` bash
systemctl restart vsftpd
systemctl enable vsftpd
```


