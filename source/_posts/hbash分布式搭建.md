---
title: hbase 分布式搭建
tags: [hadoop]
categories: 大数据
date: 2020-03-20 18:00:00
---

# 介绍

<font color='32CD32'>HBase 简介</font>

1、HBase是Apache Hadoop的数据库，能够对大型数据提供随机、实时的读写访问，是Google的BigTable的开源实现。

2、HBase的目标是存储并处理大型的数据，更具体地说仅用普通的硬件配置，能够处理成千上万的行和列所组成的大型数据库。
3、HBase是一个开源的、分布式的、多版本的、面向列的存储模型。可以直接使用本地文件系统，也可使用Hadoop的HDFS文件存储系统。

为了提高数据的可靠性和系统的健壮性，并且发挥HBase处理大型数据的能力，还是使用HDFS作为文件存储系统更佳。

另外，HBase存储的是松散型数据，具体来说，HBase存储的数据介于映射（key/value）和关系型数据之间。HBase存储的数据从逻辑上看就是一张很大的表，并且它的数据列可以根据需要动态增加。每一个cell中的数据又可以有多个版本（通过时间戳来区别），从下图来看，HBase还具有 " 向下提供存储，向上提供运算 " 的特点。

<font color='32CD32'>HBase 体系结构</font>

HBase的服务器体系结构遵从简单的主从服务器架构，它由HRegion Server群和HBase Master服务器构成。

HBase Master负责管理所有的HRegion Server，而HBase中的所有RegionServer都是通过ZooKeeper来协调，并处理HBase服务器运行期间可能遇到的错误。

HBase Master Server本身并不存储HBase中的任何数据，HBase逻辑上的表可能会被划分成多个Region，然后存储到HRegion Server群中。HBase Master Server中存储的是从数据到HRegion Server的映射。

`Client`：HBase Client 使用HBase的RPC机制与HMaster和HRegionServer进行通信：对于管理类操作，Client与HMaster进行RPC；对于数据读写类操作，Client与HRegionServer进行RPC。

`Zookeeper`：Zookeeper Quorum中除了存储了-ROOT-表的地址和HMaster的地址，HRegionServer也会把自己以Ephemeral方式注册到 Zookeeper中，使得HMaster可以随时感知到各个HRegionServer的健康状态。此外，Zookeeper也避免了HMaster的 单点问题。

`HMaster`：每台HRegionServer都会与HMaster进行通信，HMaster的主要任务就是要告诉每台HRegion Server它要维护哪些HRegion。当一台新的HRegionServer登录到HMaster时，HMaster会告诉它等待分配数据。而当一台HRegion死机时，HMaster会把它负责的HRegion标记为未分配，然后再把它们分配到其他的HRegion Server中。
HBase已经解决了HMaster单点故障问题（SPFO），并且HBase中可以启动多个HMaster，那么它就能够通过Zookeeper来保证系统中总有一个Master在运行。HMaster在功能上主要负责Table和Region的管理工作。

 `HRegion`：当表的大小超过设置值得时候，HBase会自动地将表划分为不同的区域，每个区域包含所有行的一个子集。对用户来说，每个表是一堆数据的集合，靠主键来区分。从物理上来说，一张表被拆分成了多块，每一块就是一个HRegion。我们用表名+开始/结束主键来区分每一个HRegion，一个HRegion会保存一个表里面某段连续的数据，从开始主键到结束主键，一张完整的表格是保存在多个HRegion上面。 

- 管理用户对Table的增删改查操作
- 管理HRegionServer的负载均衡，调整Region分布
- 在Region Split后，负责新Region的分配
- 在HRegionServer停机后，负责失效HRegionServer上的Region迁移

`HRegionServer`：主要负责响应用户I/O请求，向HDFS文件系统中读写数据，是HBase中最核心的模块。HRegionServer内部管理了一系列HRegion对象，每个HRegion对应了Table中的一个 Region，HRegion中由多个HStore组成。每个HStore对应了Table中的一个Column Family的存储，可以看出每个Column Family其实就是一个集中的存储单元，因此最好将具备共同IO特性的column放在一个Column Family中，这样最高效。

`HStore`：存储是HBase存储的核心，其中由两部分组成，一部分是MemStore，一部分是StoreFiles。 MemStore是Sorted Memory Buffer，用户写入的数据首先会放入MemStore，当MemStore满了以后会Flush成一个StoreFile(底层实现是HFile)， 当StoreFile文件数量增长到一定阈值，会触发Compact合并操作，将多个StoreFiles合并成一个StoreFile，合并过程中会进行版本合并和数据删除，因此可以看出HBase其实只有增加数据，所有的更新和删除操作都是在后续的compact过程中进行的，这使得用户的写操作只要 进入内存中就可以立即返回，保证了HBase I/O的高性能。当StoreFiles Compact后，会逐步形成越来越大的StoreFile，当单个StoreFile大小超过一定阈值后，会触发Split操作，同时把当前 Region Split成2个Region，父Region会下线，新Split出的2个孩子Region会被HMaster分配到相应的HRegionServer 上，使得原先1个Region的压力得以分流到2个Region上。

# 安装

## hadoop 安装

[hadoop 分布式集群安装教程](https://hxqxiaoqi.gitee.io/2020/03/20/hadoop%20完全分布式搭建/#hadoop-安装) ：hbase是基于hadoop的hdfs所设计的NoSql数据库，所以需要先安装hadoop集群

## zookeeper 安装

[zookeeper 安装教程](https://hxqxiaoqi.gitee.io/2019/09/03/zookeeper安装/)

## hbase 安装

<font color='32CD32'>在 192.168.40.100 上操作：</font>

```bash
# 下载
wget https://mirror.bit.edu.cn/apache/hbase/stable/hbase-2.2.3-bin.tar.gz
tar xf hbase-2.2.3-bin.tar.gz -C /opt/
cd /opt/hbase-2.2.3/conf/
```

<font color='32CD32'>修改 hbase-env.sh，添加以下内容：</font>

```bash
# jdk 环境变量
export JAVA_HOME=/opt/jdk1.8.0_221
# hbase 配置目录环境变量
export HBASE_CLASSPATH=/opt/hbase-2.2.3/conf
# 关闭内部zookeeper，使用外部zookeeper的配置
export HBASE_MANAGES_ZK=false
```

<font color='32CD32'>修改 hbase-site.xml ：</font>

```xml
 <configuration>
    <property>
      <name>hbase.unsafe.stream.capability.enforce</name>
      <value>false</value>
    </property>
     <property>
         <name>hbase.zookeeper.property.clientPort</name>
         <value>2181</value>
     </property>
     <property>
         <name>hbase.zookeeper.quorum</name>
         <value>hadoop0,hadoop1,hadoop2</value>
     </property>
     <property>
         <name>hbase.zookeeper.property.dataDir</name>
         <value>/var/zookeeper</value>
     </property>
     <property>
         <name>hbase.rootdir</name>
         <value>hdfs://hadoop0:9000/hbase</value>
     </property>
     <property>
         <name>hbase.cluster.distributed</name>
         <value>true</value>
     </property>
 </configuration>
```

<font color='32CD32'>修改 regionservers ：</font>

```bash
hadoop0
hadoop1
hadoop2
```

<font color='32CD32'>配置高可用</font>

```bash
echo "hadoop1" > /opt/hbase-2.2.3/conf/backup-masters
```

<font color='32CD32'>分发配置</font>

```bash
scp -r /opt/hbase-2.2.3/ root@hadoop1:/opt/
scp -r /opt/hbase-2.2.3/ root@hadoop2:/opt/
```

<font color='32CD32'>启动与关闭</font>

```bash
# 在master上启动集群
/opt/hbase-2.2.3/bin/start-hbase.sh
/opt/hbase-2.2.3/bin/stop-hbase.sh

# 单独启动master
bin/hbase-daemon.sh start master
# 单独启动HRegionServer
bin/hbase-daemon.sh start regionserver
```

<font color='32CD32'>访问</font>

```bash
curl http://192.168.40.100:16010/
```

## hbase 常用操作命令

```bash
# 登录hbase数据库
cd /opt/hbase-2.2.3/bin/
./hbase shell

# 列出当前数据库中的所有namespace：
> list_namespace

# 列出hbase中的表：
> list

# 查看帮助信息，找到创建的语法格式，注意要加上引号：
> help 'create_namespace'

# 创建namespace：
> create_namespace 'nstest'

# 描述查看namespace的结构：
> drop_namespace 'nstest'

# 创建表，表名user，列簇info：
示例：create 'ns1:t1', {NAME => 'f1', VERSIONS => 5}
# 1）ns1指的就是namespace
# 2）t1代表table_name
# 3）ns1:t1这样的格式就是唯一确定了一张表
# 4）在hbase中=>符号表示等于
# 5）f指的是列簇，建表时要指定一个列簇
# 6）VERSIONS => 5代表同时能够存储的版本数
# 7）可以指定多个列簇，一个大括号中只能指定一个NAME（变量）
# 8）一个列簇就是一个大括号
# 9）在建表的时候可以指定在某个namespace下，比如：ns1:t1，没有指定就是在默认的数据库下面创建
> create 'user','info'

# 查看表user的结构：
> describe 'user'

# 向表user中插入数据。表名user，rowkey为10001，列簇info，列名name等，cell值为zhangsan：
> put 'user','10001','info:name','zhangsan'
> put 'user','10001','info:age','25'
> put 'user','10001','info:sex','male'
> put 'user','10001','info:address','shanghai'

# HBase中的数据查询有三种方式：
# 1）依据rowkey查询，这是最快的，使用get命令；
# 2）依据范围查询，这是最常用的，使用scan range命令；
# 3）全表扫描，这是最慢的，使用scan命令。

# 查询user表中rowkey为10001的信息：
> get 'user','10001'

# 查询user表中rowkey为10001，列簇为info的信息：
> get 'user','10001','info'

# 查询user表中rowkey为10001，列簇为info，列名为name的信息：
> get 'user','10001','info:name'

# 插入rowkey为10002的信息：
> put 'user','10002','info:name','wangwu'
> put 'user','10002','info:age','30'
> put 'user','10002','info:tel','25354212'
> put 'user','10002','info:qq','232523551'

# 全表扫描：
> scan

# 全表扫描user表：
> scan 'user'

# 插入user表中列簇为10003的信息：
> put 'user','10003','info:name','zhaoliu'

# 范围查询：查询user表中的name列和age列的信息：
> scan 'user',{COLUMNS => ['info:name','info:age']}

# 范围查询：查询user表中起始rowkey为10002开始的行信息：
> scan 'user', {STARTROW=>'10002'}

# STARTROW代表开始的行号，大括号中的所有变量都必须是大写：
> scan 'ns1:t1', {COLUMNS => ['c1', 'c2'], LIMIT => 10, STARTROW => 'xyz'}
> scan 'nstest:tb1', {STARTROW => '20170521_10002'}

# STOPROW代表结束的行号，包头不包尾：
> scan 'nstest:tb1', {STARTROW => '20170521_10001',STOPROW => '20170521_10003'}

# 删除user表中rowkey为10001，列簇为info，列名为name，值为zhangsan的数据（可能该值有多个版本）：
> delete 'user','10001','info:name','zhangsan'

# 删除user表中rowkey为10001，列簇为info，列名为name的列数据：全表扫描user表：查看结果
> delete 'user','10001','info:name'

# 删除user表中rowkey为10001的全部信息：全表扫描user表：查看结果
> deleteall 'user','10001'

# 禁用user表：
> disable 'user'
# 启用user表：
> enable 'user'

# 删除user表：
> drop 'user'

# 退出hbase shell命令行：
> exit

# 删除g开头的表,\ny：自动确认
echo -e "disable_all 'g.*'\ny" | /data/hbase-2.2.3/bin/hbase shell -n
echo -e "drop_all 'g.*'\ny" | /data/hbase-2.2.3/bin/hbase shell -n
```

