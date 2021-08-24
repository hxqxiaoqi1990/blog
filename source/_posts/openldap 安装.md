---
title: openldap 安装
tags: [ldap]
categories: linux基础
date: 2020-04-30 12:27:43
---

# 介绍

<font color='32CD32'>LDAP</font>（Lightweight Directory Access Protocol）是基于X.500标准的轻量目录访问协议。它比X.500具有更快的查询速度和更低的资源消耗，精简灵活，支持TCP/IP协议。LDAP为用户管理提供了统一认证服务，解决了办公环境中多套用户认证和项目管理系统相互独立分散的难题，而OpenLDAP 是LDAP的开源实现。

其实就是各个支持ldap协议的应用，统一账号的地方，而不需要在每个应用上注册管理单独账号。

<font color='32CD32'>cn ：</font>common name 用户名

对象的属性为CN，例如一个用户的名字为：张三，那么“张三”就是一个CN。

<font color='32CD32'>ou : </font>OrganizationUnit 组织单位

o和ou都是ldap目录结构的一个属性，建立目录的时候可选新建o，ou 等。在配置我司交换设备ldap的时候具体是配置ou，o还是cn等，要具体看ldap服务器的相应目录是什么属性。

<font color='32CD32'>o：</font>organizationName 组织名

<font color='32CD32'>uid: </font>userid

对象的属性为uid，例如我司一个员工的名字为：zsq，他的UID为：z02691，ldap查询的时候可以根据cn，也可以根据uid。配置ldap查询的时候需要考虑用何种查询方式。我司两种方式都支持，具体我司设备配置根据何种方式查询需要有ldap服务器的相关配置来决定。

<font color='32CD32'>dc：</font>域组件

DC类似于dns中的每个元素，例如h3c.com，“.”符号分开的两个单词可以看成两个DC，

<font color='32CD32'>dn：</font>Distinguished Name

类似于DNS，DN与DNS的区别是：组成DN的每个值都有一个属性类型，例如:H3c.com是一个dns，那么用dn表示为：dc=h3c,dc=com 级别越高越靠后。H3c和com的属性都是DC。DN可以表示为ldap的某个目录，也可以表示成目录中的某个对象，这个对象可以是用户等。

> 例如：CN=test,OU=developer,DC=domainname,DC=com 
>
> 在上面的代码中 cn=test 可能代表一个用户名，ou=developer 代表一个 active directory 中的组织单位。这句话的含义可能就是说明 test 这个对象处在domainname.com 域的 developer 组织单元中。

# 安装步骤

<font color='32CD32'>实验环境</font>

| 主机名 |       IP       |                  安装服务                  | 系统版本  |
| :----: | :------------: | :----------------------------------------: | :-------: |
| server | 192.168.40.100 | openldap openldap-servers openldap-clients | centos7.4 |
| client | 192.168.40.101 | openldap openldap-servers openldap-clients | centos7.4 |

<font color='32CD32'>关闭防火墙和selinux</font>

```bash
systemctl stop firewalld
systemctl disable firewalld
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

<font color='32CD32'>配置epel源</font>

```bash
yum install epel-release -y
```

<font color='32CD32'>OpenLDAP安装</font>

注：以下操作没有特殊说明，均在server主机操作。

```bash
yum install openldap openldap-servers openldap-clients -y
cp /usr/share/openldap-servers/DB_CONFIG.example /var/lib/ldap/DB_CONFIG
systemctl start slapd
systemctl enable slapd
systemctl status slapd
```

<font color='32CD32'>配置管理员密码</font>

```bash
slappasswd -s 111111
{SSHA}AjIAg98NFvRRlhoBOvsfVqMeAGZwi5O9
```

<font color='32CD32'>生成的密码写入到 rootpwd.ldif 文件中</font>

```bash
cat <<EOF>rootpwd.ldif
dn: olcDatabase={0}config,cn=config
changetype: modify
add: olcRootPW
olcRootPW: {SSHA}AjIAg98NFvRRlhoBOvsfVqMeAGZwi5O9
EOF

# 相关参数说明：
# olcRootPW参数值为slappasswd命令的结果输出。
# olcRootPW: {SSHA}AjIAg98NFvRRlhoBOvsfVqMeAGZwi5O9指定了属性值。
# ldif文件是LDAP中数据交换的文件格式。文件内容采用的是key-value形式。注意value后面不能有空格。
# olc（OnlineConfiguration）表示写入LDAP后立即生效。
# changetype: modify表示修改entry，此外changetype的值也可以是add,delete等。
# add: olcRootPW表示对这个entry新增olcRootPW的属性。
```

<font color='32CD32'>写入配置</font>

执行命令，添加 rootpwd.ldif 配置 到 openldap 中：ldap所有修改操作都需要编写单独的文件，切记不要直接去修改原有的配置。

```bash
ldapadd -Y EXTERNAL -H ldapi:/// -f rootpwd.ldif
```

<font color='32CD32'>导入schema</font>

schema包含支持特殊场景的相关属性，可选择性导入或全部导入

```bash
ls /etc/openldap/schema/*.ldif | while read i; do ldapadd -Y EXTERNAL -H ldapi:/// -f $i; done
```

<font color='32CD32'>新建默认域配置文件</font>

```bash
cat <<EOF> domain.ldif
dn: olcDatabase={1}monitor,cn=config
changetype: modify
replace: olcAccess
olcAccess: {0}to * by dn.base="gidNumber=0+uidNumber=0,cn=peercred,cn=external,cn=auth" read by dn.base="cn=Manager,dc=mydomain,dc=com" read by * none

# olcDatabase={2}hdb，改配置名需要到/etc/openldap/slapd.d/cn\=config/目录查看，不是都一样的，需要修改
dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=mydomain,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
replace: olcRootDN
olcRootDN: cn=Manager,dc=mydomain,dc=com

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcRootPW
# 上面生成的密码填写到这，或可以重新生成一个填写，与管理员密码区分
olcRootPW: {SSHA}7AA7V1xy++LVZcUbBWYYM9/81wfODiIK

dn: olcDatabase={2}hdb,cn=config
changetype: modify
add: olcAccess
olcAccess: {0}to attrs=userPassword,shadowLastChange by dn="cn=Manager,dc=mydomain,dc=com" write by anonymous auth by self write by * none
olcAccess: {1}to dn.base="" by * read
olcAccess: {2}to * by dn="cn=Manager,dc=mydomain,dc=com" write by * read
EOF
```

<font color='32CD32'>写入配置</font>

执行命令，添加 domain.ldif 配置 到 openldap 中

```bash
ldapmodify -Y EXTERNAL -H ldapi:/// -f domain.ldif
```

<font color='32CD32'>添加基本信息</font>

```bash
cat <<EOF> basedomain.ldif
dn: dc=mydomain,dc=com
objectClass: top
objectClass: dcObject
objectclass: organization
o: mydomain com
dc: mydomain

dn: cn=Manager,dc=mydomain,dc=com
objectClass: organizationalRole
cn: Manager
description: Directory Manager

dn: ou=People,dc=mydomain,dc=com
objectClass: organizationalUnit
ou: People

dn: ou=Group,dc=mydomain,dc=com
objectClass: organizationalUnit
ou: Group
EOF
```

<font color='32CD32'>写入</font>

```bash
ldapadd -x -D cn=Manager,dc=mydomain,dc=com -W -f basedomain.ldif
```

<font color='32CD32'>查看信息</font>

```bash
ldapsearch -LLL -W -x -D "cn=Manager,dc=mydomain,dc=com" -H ldap://localhost -b"dc=mydomain,dc=com"
```

如果没有报错，则openldap服务端安装成功。

# 安装 ldapadmin web 管理

在OpenLDAP运行节点安装ldapadmin工具包

```bash
yum install -y httpd php php-mbstring php-pear phpldapadmin
```

修改 /etc/httpd/httpd.conf

```bash
# line 86: 修改admin邮箱地址
ServerAdmin root@openldap.mydomain.world

# line 95: 修改主机域名
ServerName www.mydomain.com:80

# line 152: 修改成如下内容：
AllowOverride All

# line 165: 添加可访问目录名的文件名称
DirectoryIndex index.html index.cgi index.php

# 在文件末尾增加如下两部分内容
# server's response header
ServerTokens Prod

# keepalive is ON
KeepAlive On
```

启动Apache服务并设置开机自启动

```bash
systemctl start httpd
systemctl enable httpd
```

编辑 /etc/phpldapadmin/config.php

```bash
#注释掉398行，并取消397行的注释，修改后内容如下：
$servers->setValue('login','attr','dn');
// $servers->setValue('login','attr','uid');
```

编辑 /etc/httpd/conf.d/phpldapadmin.conf

```bash
<Directory/usr/share/phpldapadmin/htdocs>
  <IfModule mod_authz_core.c>
    # Apache 2.4
    Require local
    Require all granted
  </IfModule>
```

重启Apache服务使配置生效

```bash
systemctl restart httpd
```

访问

账号：cn=Manager,dc=mydomain,dc=com

密码：111111

```bash
curl http://192.168.40.100/ldapadmin/
```

