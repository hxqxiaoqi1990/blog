---
title: hadoop完全分布式搭建
tags: [hadoop]
categories: 大数据
date: 2020-03-20 15:00:00
---

# 介绍

<font color='32CD32'>Hadoop</font>
Hadoop是一个由Apache基金会所开发的分布式系统基础架构。

Hadoop实现了一个分布式文件系统（Hadoop Distributed File System），简称HDFS。HDFS有高容错性的特点，并且设计用来部署在低廉的（low-cost）硬件上；而且它提供高吞吐量（high throughput）来访问应用程序的数据，适合那些有着超大数据集（large data set）的应用程序。HDFS放宽了（relax）POSIX的要求，可以以流的形式访问（streaming access）文件系统中的数据。

Hadoop的框架最核心的设计就是：HDFS和MapReduce。HDFS为海量的数据提供了存储，则MapReduce为海量的数据提供了计算。

<font color='32CD32'>YARN：</font> (分布式资源管理器) 不同计算框架可以共享同一个HDFS集群上的数据，享受整体的资源调度。

YARN的基本思想是将资源管理和作业调度/监控的功能分解为单独的守护进程（守护进程(daemon)是一类在后台运行的特殊进程，用于执行特定的系统任务。很多守护进程在系统引导的时候启动，并且一直运行直到系统关闭。另一些只在需要的时候才启动，完成任务后就自动结束。）。 这个想法是有一个全局的ResourceManager（RM）和每个应用程序的ApplicationMaster（AM）。 应用程序可以是单个作业，也可以是DAG作业。

<font color='32CD32'>工作组件介绍</font>

`Client`：客户端，系统使用者。

1. 文件切分。文件上传 HDFS 的时候，Client 将文件切分成 一个一个的Block，然后进行存储。
2. 与 NameNode 交互，获取文件的位置信息。
3. 与 DataNode 交互，读取或者写入数据。
4. Client 提供一些命令来管理 HDFS，比如启动或者关闭HDFS。
5. Client 可以通过一些命令来访问 HDFS。

`NameNode`：就是 master，它是一个主管、管理者。

1. 管理 HDFS 的名称空间。
2. 管理数据块（Block）映射信息
3. 配置副本策略
4. 处理客户端读写请求。

`DataNode`：就是Slave。NameNode 下达命令，DataNode 执行实际的操作。

1. 存储实际的数据块。
2. 执行数据块的读/写操作。
3. 定期向NameNode发送心跳信息，汇报本身及其所有的block信息，健康情况。

`Secondary NameNode`：并非 NameNode 的热备。当NameNode 挂掉的时候，它并不能马上替换 NameNode 并提供服务。注意：在hadoop 2.x 版本，当启用 hdfs ha 时，将没有这一角色

1. 辅助 NameNode，分担其工作量。
2. 定期合并 fsimage和fsedits，并推送给NameNode。
3. 在紧急情况下，可辅助恢复 NameNode。

`ResourceManager`， 简称RM，ResourceManager是仲裁系统中所有应用程序之间资源的最终权威机构。（大管理员）

1. 整个集群同一时间提供服务的RM只有一个，它负责集群资源的统一管理和调度。
2. 处理客户端的请求，例如：提交作业或结束作业等。
3. 监控集群中的NM，一旦某个NM挂了，那么就需要将该NM上运行的任务告诉AM来如何进行处理。
4. ResourceManager主要有两个组件：`Scheduler` 和 `ApplicationManager`。
5. Scheduler是一个资源调度器，它主要负责协调集群中各个应用的资源分配，保障整个集群的运行效率。Scheduler的角色是一个纯调度器，它只负责调度Containers，不会关心应用程序监控及其运行状态等信息。同样，它也不能重启因应用失败或者硬件错误而运行失败的任务。Scheduler是一个可插拔的插件，它可以调度集群中的各种队列、应用等。
6. ApplicationManager主要负责接收job的提交请求，为应用分配第一个Container来运行ApplicationMaster，还有就是负责监控ApplicationMaster，在遇到失败时重启ApplicationMaster运行的Container。

`NodeManager`， 简称NM，（执行者，小员工）

1. 整个集群中会有多个NM，它主要负责自己本身节点的资源管理和使用。
2. 定时向RM汇报本节点的资源使用情况。
3. 接收并处理来自RM的各种命令，例如：启动Container。
4. NM还需要处理来自AM的命令，例如：AM会告诉NM需要启动多少个Container来跑task，单个节点的资源管理。

`ApplicationMaster`， 简称AM，应用级别（小管理员）

1. 每个应用程序都对应着一个AM。例如：MapReduce会对应一个、Spark会对应一个，它主要负责应用程序的管理。
2. 为应用程序向RM申请资源（Core、Memory），将资源分配给内部的task。
3. AM需要与NM通信，以此来启动或停止task。遇到失败的任务还负责重启它。

`Container`

1. 封装了CPU、Memory等资源的一个容器
2. 是一个任务运行环境的抽象

# 搭建

<font color='32CD32'>实验环境：</font>

| 主机名  |     IP地址     |                   运行组件                   |
| :-----: | :------------: | :------------------------------------------: |
| hadoop0 | 192.168.40.100 | NameNode，ResourceManager，SecondaryNameNode |
| hadoop1 | 192.168.40.101 |            NodeManager，DataNode             |
| hadoop2 | 192.168.40.102 |            NodeManager，DataNode             |

## JDK 安装

所有节点均安装：[jdk安装教程](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/)

## hosts 配置

所有节点均配置

```bash
cat >> /etc/hosts << EOF
192.168.40.100 hadoop0
192.168.40.101 hadoop1
192.168.40.102 hadoop2
EOF
```

## ssh 免密配置

在所有节点上执行：

```bash
# 需要输入的，一路回车就行
ssh-keygen -t rsa
```

在 192.168.40.100 执行：

```bash
cp ~/.ssh/id_rsa.pub ~/.ssh/authorized_keys
```

将其他节点的公钥添加进authorized_keys，之后执行以下命令：

```
scp ~/.ssh/authorized_keys root@hadoop1:/root/.ssh/
scp ~/.ssh/authorized_keys root@hadoop2:/root/.ssh/
```

## ntp 时间同步

所有节点执行

```bash
yum install ntp
ntpdate -u ntp1.aliyun.com
```

## 关闭防火墙

所有节点执行

```bash
# 查看防火墙状态
firewall-cmd --state

# 停止firewall
systemctl stop firewalld.service

# 禁止firewall开机启动
systemctl disable firewalld.service
```

## hadoop 安装

<font color='32CD32'>以下操作只有在 192.168.40.100 上执行，全部配置好后，再传给其它节点。</font>

```bash
# 下载
wget http://mirror.bit.edu.cn/apache/hadoop/common/hadoop-2.7.7/hadoop-2.7.7.tar.gz

# 解压
tar xf hadoop-2.7.7.tar.gz -C /opt/

# 切换到配置目录
cd /opt/hadoop-2.7.7/etc/hadoop/
```

<font color='32CD32'>修改 hadoop-env.sh</font>

```bash
# 修改jdk环境变量
export JAVA_HOME=/opt/jdk1.8.0_221
```

<font color='32CD32'>修改 yarn-env.sh</font>

```bash
# 添加jdk环境变量
export JAVA_HOME=/opt/jdk1.8.0_221/
```

<font color='32CD32'>修改 core-site.xml</font>

注：file后的路径下的temp文件夹需要自己创建，所有节点均创建

```xml
<configuration>
    	<!-- 指定HDFS老大（namenode）的通信地址 -->
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://hadoop0:9000</value>
        </property>
        <property>
                <name>io.file.buffer.size</name>
                <value>131072</value>
        </property>
    	<!-- 指定hadoop运行时产生文件的存储路径 -->
        <property>
                <name>hadoop.tmp.dir</name>
                <value>file:/usr/temp</value>
        </property>
        <property>
                <name>hadoop.proxyuser.root.hosts</name>
                <value>*</value>
        </property>
        <property>
                <name>hadoop.proxyuser.root.groups</name>
                <value>*</value>
        </property>
</configuration>
```

<font color='32CD32'>修改 hdfs-site.xml</font>

注：file后的路径下的/dfs/文件夹需要自己创建，所有节点均创建

```xml
<configuration>
    	<!-- 设置secondarynamenode的http通讯地址 -->
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>hadoop0:9001</value>
        </property>
    	<!-- 设置namenode存放的路径 -->
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/usr/dfs/name</value>
        </property>
    	<!-- 设置datanode存放的路径 -->
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/usr/dfs/data</value>
        </property>
    	<!-- 设置hdfs副本数量 -->
        <property>
                <name>dfs.replication</name>
                <value>2</value>
        </property>
        <property>
                <name>dfs.webhdfs.enabled</name>
                <value>true</value>
        </property>
        <property>
                <name>dfs.permissions</name>
                <value>false</value>
        </property>
        <property>
                <name>dfs.web.ugi</name>
                <value>supergroup</value>
        </property>
</configuration>
```

<font color='32CD32'>修改 mapred-site.xml</font>

注：要将mapred-site.xml.template重命名为 .xml的文件   mv mapred-site.xml.template mapred-site.xml

```xml
<configuration>
    	<!-- 通知框架MR使用YARN -->
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
        <property>
                <name>mapreduce.jobhistory.address</name>
                <value>hadoop0:10020</value>
        </property>
        <property>
                <name>mapreduce.jobhistory.webapp.address</name>
                <value>hadoop0:19888</value>
        </property>
</configuration>
```

<font color='32CD32'>修改 yarn-site.xml</font>

```xml
<configuration>
    	<!-- reducer取数据的方式是mapreduce_shuffle -->
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
        <property>
                <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
                <value>org.apache.hadoop.mapred.ShuffleHandler</value>
        </property>
        <property>
                <name>yarn.resourcemanager.address</name>
                <value>hadoop0:8032</value>
        </property>
        <property>
                <name>yarn.resourcemanager.scheduler.address</name>
                <value>hadoop0:8030</value>
        </property>
        <property>
                <name>yarn.resourcemanager.resource-tracker.address</name>
                <value>hadoop0:8031</value>
        </property>
        <property>
                <name>yarn.resourcemanager.admin.address</name>
                <value>hadoop0:8033</value>
        </property>
    	<!-- resourcemanager web访问端口 -->
        <property>
                <name>yarn.resourcemanager.webapp.address</name>
                <value>hadoop0:8088</value>
        </property>
</configuration>
```

<font color='32CD32'>修改 slaves</font>

指定从节点：会运行 `NodeManager` 与 `DataNode` 组件

```bash
hadoop1
hadoop2
```

<font color='32CD32'>配置 hadoop 环境变量</font>

```bash
cat >> /etc/profile << EOF
# hadoop环境变量
export  HADOOP_HOME=/opt/hadoop-2.7.7/
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
EOF

source /etc/profile
```

<font color='32CD32'>拷贝文件到其它节点</font>

```bash
# 拷贝hadoop-2.7.
scp -r /opt/hadoop-2.7.7/ root@hadoop1:/opt/
scp -r /opt/hadoop-2.7.7/ root@hadoop2:/opt/

# 拷贝profile文件，之后需要在其它节点执行：source /etc/profile 使配置生效
scp /etc/profile root@hadoop1:/etc/
scp /etc/profile root@hadoop2:/etc/
```

<font color='32CD32'>格式化主节点的namenode，我的主节点是 192.168.40.100</font>

```bash
# 主节点上进入hadoop目录
cd /opt/hadoop-2.7.7/

# 然后执行：提示：successfully formatted表示格式化成功
./bin/hadoop namenode -format

#新版本用下面的语句不用hadoop命令了
./bin/hdfs namenode -format
```

<font color='32CD32'>启动与关闭</font>

```bash
cd /opt/hadoop-2.7.7/
./sbin/start-all.sh
./sbin/stop-all.sh
```

<font color='32CD32'>查看各节点进程</font>

```bash
# 主节点
[root@hadoop0 hadoop]# jps
4051 NameNode
4389 ResourceManager
9611 Jps
4238 SecondaryNameNode

# 两个子节点
[root@hadoop1 hadoop]# jps
1990 NodeManager
3575 Jps
1885 DataNode
```

如果全部启动成功，表示安装完成

<font color='32CD32'>访问</font>

```bash
# 访问 hdfs web管理界面
curl http://192.168.40.100:50070

# 访问 yarn web管理界面
curl http://192.168.40.100:8088
```

# HA自动切换配置

参考文档：https://blog.csdn.net/weixin_37838429/article/details/81710045

## zk安装

[安装zookeeper](https://hxqxiaoqi.gitee.io/2020/03/26/zookeeper%20%E9%9B%86%E7%BE%A4%E6%90%AD%E5%BB%BA/)

## 修改配置

在192.168.40.100上执行

```bash
mkdir /opt/HA
cp -r /opt/hadoop-2.7.7/ /opt/HA
```

修改core-site.xml

```xml
<configuration>
<!-- 把两个NameNode）的地址组装成一个集群mycluster -->
    <property>
        <name>fs.defaultFS</name>
            <value>hdfs://mycluster</value>
    </property>

    <!-- 声明journalnode服务本地文件系统存储目录-->
    <property>
        <name>dfs.journalnode.edits.dir</name>
        <value>/opt/HA/hadoop-2.7.7/data/jn</value>
    </property>

    <!-- 指定hadoop运行时产生文件的存储目录 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/HA/hadoop-2.7.7/data/tmp</value>
    </property>
    <property>
        <name>ha.zookeeper.quorum</name>
        <value>hadoop0:2181,hadoop1:2181,hadoop2:2181</value>
    </property>
</configuration>

```

修改hdfs-site.xml

```xml
<configuration>
    <!-- 完全分布式集群名称 -->
    <property>
        <name>dfs.nameservices</name>
        <value>mycluster</value>
    </property>

    <!-- 集群中NameNode节点都有哪些 -->
    <property>
        <name>dfs.ha.namenodes.mycluster</name>
        <value>nn1,nn2</value>
    </property>

    <!-- nn1的RPC通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.mycluster.nn1</name>
        <value>hadoop0:8020</value>
    </property>

    <!-- nn2的RPC通信地址 -->
    <property>
        <name>dfs.namenode.rpc-address.mycluster.nn2</name>
        <value>hadoop1:8020</value>
    </property>

    <!-- nn1的http通信地址 -->
    <property>
        <name>dfs.namenode.http-address.mycluster.nn1</name>
        <value>hadoop0:50070</value>
    </property>

    <!-- nn2的http通信地址 -->
    <property>
        <name>dfs.namenode.http-address.mycluster.nn2</name>
        <value>hadoop1:50070</value>
    </property>

    <!-- 指定NameNode元数据在JournalNode上的存放位置 -->
    <property>
        <name>dfs.namenode.shared.edits.dir</name>
		<value>qjournal://hadoop0:8485;hadoop1:8485;hadoop2:8485/mycluster</value>
    </property>

    <!-- 配置隔离机制，即同一时刻只能有一台服务器对外响应 -->
    <property>
        <name>dfs.ha.fencing.methods</name>
        <value>sshfence</value>
    </property>

    <!-- 使用隔离机制时需要ssh无秘钥登录-->
    <property>
        <name>dfs.ha.fencing.ssh.private-key-files</name>
        <value>/root/.ssh/id_rsa</value>
    </property>

    <!-- 关闭权限检查-->
    <property>
        <name>dfs.permissions.enable</name>
        <value>false</value>
    </property>

    <!-- 访问代理类：client，mycluster，active配置失败自动切换实现方式-->
    <property>
        <name>dfs.client.failover.proxy.provider.mycluster</name>
		<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
    </property>
    <property>
        <name>dfs.ha.automatic-failover.enabled</name>
        <value>true</value>
    </property>
</configuration>
```

分发配置到其它机器上

## 启动步骤

```bash
# 关闭所有HDFS服务：
sbin/stop-dfs.sh

# 启动Zookeeper集群：
bin/zkServer.sh start

# 初始化HA在Zookeeper中状态，在master上执行
bin/hdfs zkfc -formatZK

# 启动HDFS服务：
sbin/start-dfs.sh

# 在各个NameNode节点上启动DFSZK Failover Controller，先在哪台机器启动，哪个机器的NameNode就是Active NameNode
# 说明：如果使用start-dfs.sh启动集群，不需要单独启动zkfc
sbin/hadoop-daemon.sh start zkfc

# 验证:将Active NameNode进程kill
kill -9 namenode的进程id
```

## 常用命令

```bash
# 单独启动journalnode
sbin/hadoop-daemon.sh start journalnode

# 格式化
bin/hdfs namenode -format

# 单独启动namenode
sbin/hadoop-daemon.sh start namenode

# 在[nn2]上，同步nn1的元数据信息：
bin/hdfs namenode -bootstrapStandby

# 启动所有datenode
sbin/hadoop-daemons.sh start datanode

# 将[nn1]切换为Active
bin/hdfs haadmin -transitionToActive nn1

# 查看是否Active
bin/hdfs haadmin -getServiceState nn1

# 开启和关闭安装模式
bin/hdfs dfsadmin -safemode enter
bin/hdfs dfsadmin -safemode leave
```

