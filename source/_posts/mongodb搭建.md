---
title: mongodb搭建
tags: [mongodb]
categories: 数据库
date: 2019-10-28 16:20:00
---


# 介绍
**MongoDB：** 是由C++语言编写的一个基于分布式文件存储的开源数据库系统，它的目的在于为WEB应用提供可扩展的高性能数据存储解决方案。是一个介于关系型数据库和非关系型数据库之间的产品，是非关系型数据库当中功能最丰富，最像关系型数据库的。它支持的数据结构非常松散，会将数据存储为一个文档，数据结构由键值对(key=>value)组成，是类似于json的bson格式，字段值可以包含其它文档、数组和文档数组，因此可以存储比较复杂的数据类型。

# 安装
[官网下载](https://fastdl.mongodb.org)
以下为自动安装脚本
``` bash
#!/bin/bash
# 自动安装mongodb
workdir=/data
mongodb_version=mongodb-linux-x86_64-3.6.14
[ -e $workdir ] || mkdir $workdir 
mongodb_dir=$workdir/$mongodb_version

yum -y install wget
wget https://fastdl.mongodb.org/linux/$mongodb_version.tgz

tar xf $mongodb_version.tgz -C /data
cd $mongodb_dir
mkdir {data,logs}

cat > mongodb.conf <<EOF
# 数据库路径
dbpath=$mongodb_dir/data

# 日志输出文件路径
logpath=$mongodb_dir/logs/mongodb.log

# 错误日志采用追加模式
logappend=true

# 启用日志文件，默认启用
journal=true

# 这个选项可以过滤掉一些无用的日志信息，若需要调试使用请设置为false
quiet=true

# 端口号 默认为27017
port=27017

# 允许远程访问
bind_ip=0.0.0.0

# 开启子进程
fork=true

# 为每一个数据库按照数据库名建立文件夹存放
directoryperdb=true

#开启认证，必选先添加用户
#auth=true
EOF

$mongodb_dir/bin/mongod -f $mongodb_dir/mongodb.conf
```

# 启动脚本

``` bash
#!/bin/bash
# mongodb启动脚本
# chkconfig: 2345 10 90
dir=/data/mongodb-linux-x86_64-3.6.14
start() {
$dir/bin/mongod --config $dir/mongodb.conf
}
stop() {
$dir/bin/mongod --config $dir/mongodb.conf --shutdown
}

case "$1" in
start)
 start
 ;;
stop)
 stop
 ;;
restart)
 stop
 start
 ;;
*)
 echo $"Usage: $0 {start|stop|restart}"
 exit 1
esac
```
## 开机自启动
把以上启动脚本存为mongodb
``` shell
mv mongodb /etc/rc.d/init.d/
chmod +x /etc/rc.d/init.d/mongodb
chkconfig --add mongodb
chkconfig mongodb on
chkconfig --list
```
# 备份和恢复
**备份整个库**
``` bash
例如： 
mongodump -h localhost -d users -o /root/mongdbbak/

-h：
MongDB所在服务器地址，例如：127.0.0.1或localhost，当然也可以指定端口号：127.0.0.1:27017

-d：
需要备份的数据库实例名，例如：users

-o：
指定备份的数据存放的目录位置，例如：/root/mongdbbak/，当然该目录需要提前建立，在备份完成后，系统自动在/root/mongdbbak/目录下建立一个users目录，这个目录里面存放该数据库实例的备份数据。数据形式是以JSON的格式文件存储。
```

**恢复整个库**

``` bash
例如：
mongorestore -h localhost -d users --dir /root/mongdbbak/users

--host <:port>, -h <:port>：
MongoDB所在服务器地址，默认为:localhost:27017

-d ：
需要恢复的数据库实例名，例如：users，当然这个名称也可以和备份时候的不一样，比如user2

--dir：
指定备份的目录。
```
**导出集合**

``` bash
例如：
mongoexport -d mydb -c promotionConfiguration -o promotionConfiguration.json

-h ：数据库地址，MongoDB 服务器所在的 IP 与 端口，如 localhost:27017

-d ：指明使用的数据库实例，如 test

-c 指明要导出的集合，如 demo

-o 指明要导出的文件名，demo.json，文件类型支持txt、xls、docs 等等
```
**导入集合**

``` bash
例如：
mongoimport -d mydb -c u_vip_card_item  --type=json --file u_vip_card_item.json

-h ： 数据库地址，MongoDB 服务器所在的 IP 与 端口，如 localhost:27017

-d ：指明使用的库，指明使用的数据库实例，如 test

-c ：指明要导入的集合，如 demo可以和导出时不一致，自定义即可，不存在时会直接创建。
```

# 常用命令
**关闭mongodb**
mongod  --shutdown  --dbpath /database/mongodb/data/
或
use admin;
db.shutdownServer();

**查看库大小**
单位B查看
db.stats();

单位M查看
db.stats(1048576);

单位G查看
db.stats(1073741824);

**查看表大小(单位字节)**
db.tables.stats()

**查看全部数据库**
show dbs;                  

**显示当前数据库中的集合**（类似关系数据库中的表）
show collections;  

**查看当前数据库的用户信息**
show users;                

 **切换数据库跟mysql一样** （如果没用库，则创建，需要有数据才能生成库）
use <db name>;            

**查看当前所在数据库**
db;
db.getName(); 

 **对于当前数据库中的foo集合进行数据查找**（由于没有条件，会列出所有数据） 
db.foo.find();          

**对于当前数据库中的foo集合进行查找** （条件是数据中有一个属性叫a，且a的值为1）
db.foo.find( { a : 1 } );  

**创建表**
db.test.insert({"_id":"520","name":"xiaoming"})        

**创建用户**
use admin
db.createUser({user:"xiaoming",pwd:"123456",roles:[{role:"userAdmin",db:"test"}]})   

 **删除用户**
db.removeUser("userName");      

**显示当前所有用户**
show users;   

**删除当前所在库**
db.dropDatabase();

**查看当前库状态**
db.stats();

**查看当前连接数和最大连接数**
db.serverStatus().connections

**实时查运行状态**
mongostat

**查看压力**
mongotop


