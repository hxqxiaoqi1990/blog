---
title: flink 部署
tags: [flink]
categories: 大数据
date: 2020-04-26 13:00:00
---

# 介绍

<font color='32CD32'>Flink</font> 是新的stream计算引擎，用java实现。既可以处理stream data也可以处理batch data，可以同时兼顾Spark以及Spark streaming的功能，与Spark不同的是，Flink本质上只有stream的概念，batch被认为是special stream。Flink在运行中主要有三个组件组成，JobClient，JobManager 和 TaskManager

用户首先提交Flink程序到JobClient，经过JobClient的处理、解析、优化提交到JobManager，最后由TaskManager运行task。

# 组件说明

<font color='32CD32'>JobManager</font>

JobManager是一个进程，主要负责申请资源，协调以及控制整个job的执行过程，具体包括，调度任务、处理checkpoint、容错等等，在接收到JobClient提交的执行计划之后，针对收到的执行计划，继续解析，因为JobClient只是形成一个operaor层面的执行计划，所以JobManager继续解析执行计划（根据算子的并发度，划分task），形成一个可以被实际调度的由task组成的拓扑图，最后向集群申请资源，一旦资源就绪，就调度task到TaskManager。

<font color='32CD32'>TaskManager</font>

TaskManager是一个进程，及一个JVM（Flink用java实现）。主要作用是接收并执行JobManager发送的task，并且与JobManager通信，反馈任务状态信息，比如任务分执行中，执行完等状态，上文提到的checkpoint的部分信息也是TaskManager反馈给JobManager的。如果说JobManager是master的话，那么TaskManager就是worker主要用来执行任务。在TaskManager内可以运行多个task。多个task运行在一个JVM内有几个好处，首先task可以通过多路复用的方式TCP连接，其次task可以共享节点之间的心跳信息，减少了网络传输。TaskManager并不是最细粒度的概念，每个TaskManager像一个容器一样，包含一个多或多个Slot。

<font color='32CD32'>Slot</font>

Slot是TaskManager资源粒度的划分，每个Slot都有自己独立的内存。所有Slot平均分配TaskManger的内存，比如TaskManager分配给Solt的内存为8G，两个Slot，每个Slot的内存为4G，四个Slot，每个Slot的内存为2G，值得注意的是，Slot仅划分内存，不涉及cpu的划分。同时Slot是Flink中的任务执行器（类似Storm中Executor），每个Slot可以运行多个task，而且一个task会以单独的线程来运行。

<font color='32CD32'>ResourceManager</font>

ResourceManager主要负责管理任务管理器（TaskManager）的插槽（slot），Slot时Flink定义的处理资源单元；ResourceManager将有空闲插槽的TaskManager分配给JobManager。如果ResourceManager没有足够的插槽来满足JobManager的请求，它可以向资源提供平台发起会话，以提供启动TaskManager进程的容器。

# flink 部署

Flink 有三种部署模式，分别是 Local、Standalone Cluster 和 Yarn Cluster。

- Local 单机模式，适合用于实验环境
- Standalone Cluster 集群模式，适合用于测试环境，配合zk和hdfs，可部署高可用模式，可用于生产环境
- Yarn Cluster 基于hadoop Yarn 组件进行部署，支持高可用，适合用于生产环境

## Local 模式

1. 安装jdk
2. 下载包解压
3. 直接运行即可

## Standalone 模式

<font color='32CD32'>实验环境</font>

|       IP       | 主机名  |      安装服务      |
| :------------: | :-----: | :----------------: |
| 192.168.40.100 | master  | jdk1.8，flink1.7.1 |
| 192.168.40.101 | worker1 | jdk1.8，flink1.7.1 |
| 192.168.40.102 | worker2 | jdk1.8，flink1.7.1 |

<font color='32CD32'>jdk1.8安装</font>

[jdk1.8 安装跳转](https://hxqxiaoqi.gitee.io/2019/06/04/JDK1.8环境安装-linux/) 所有节点都需要安装

<font color='32CD32'>设置ssh免密</font>

在master上执行以下脚本，根据实际情况修改IP和密码。

```bash
cat > ssh.sh << EOF
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
yum install -y expect

#分发公钥
ssh-keygen -t rsa -P "" -f /root/.ssh/id_rsa
for i in 192.168.40.100 192.168.40.101 192.168.40.102
do
expect -c "
spawn ssh-copy-id -i /root/.ssh/id_rsa.pub root@\$i
        expect {
                \"*yes/no*\" {send \"yes\r\"; exp_continue}
                \"*password*\" {send \"123123\r\"; exp_continue}
                \"*Password*\" {send \"123123\r\";}
        } "
done
EOF
bash ssh.sh
```

<font color='32CD32'>flink1.7.1 安装</font>

以下操作，没有特殊说明，均在master上执行

```bash
# 下载
wget https://archive.apache.org/dist/flink/flink-1.7.1/flink-1.7.1-bin-hadoop27-scala_2.11.tgz
# 解压
tar xf flink-1.7.1-bin-hadoop27-scala_2.11.tgz -C /opt
```

修改配置：/opt/flink-1.7.1/conf/masters

```bash
192.168.40.100:8081
```

修改配置：/opt/flink-1.7.1/conf/slaves

```bash
192.168.40.102
192.168.40.101
```

修改配置：/opt/flink-1.7.1/conf/flink-conf.yaml

```yaml
# jobmanager地址
jobmanager.rpc.address: 192.168.40.100

# JobManager的端口号
jobmanager.rpc.port: 6123

# jobmanager可用最大内存，根据服务器内存设置
jobmanager.heap.size: 1024m

# taskmanager可用最大内存，也就是每个taskmanager所在的服务器能用的最大内存
taskmanager.heap.size: 1024m

# 每台taskmanager最大插槽数，可以根据cpu核数设定，用于划分内存，如：上面的值设置16G，slot设置2，每个slot有8G可用
taskmanager.numberOfTaskSlots: 2

# 默认使用插槽数，每个job默认分配的slot数
parallelism.default: 1
```

配置环境变量

```bash
cat >> /etc/profile << EOF
export FLINK_HOME=/opt/flink-1.7.1
export PATH=\$PATH:\$FLINK_HOME/bin
EOF
```

分发配置

```bash
for node_ip in 192.168.40.101 192.168.40.102
do
    echo ">>> ${node_ip}"
    scp -r /opt/flink-1.7.1/ root@${node_ip}:/opt
    scp /etc/profile root@${node_ip}:/etc/
    ssh root@${node_ip} "source /etc/profile"
done
```

启动与停止

```bash
/opt/flink-1.7.1/bin/start-cluster.sh
/opt/flink-1.7.1/bin/stop-cluster.sh
```

访问

```bash
# flink web管理界面，可以在浏览器访问
curl http://192.168.40.100:8081
```

运行任务

```bash
cd /opt/flink-1.7.1/
./bin/flink run examples/streaming/WordCount.jar --input /opt/flink-1.7.1/README.txt
```

## Standalone HA 模式

首先，我们需要知道 Flink 有两种部署的模式，分别是 Standalone 以及 Yarn Cluster 模式。对于 Standalone 来说，Flink 必须依赖于 Zookeeper 来实现 JobManager 的 HA（Zookeeper 已经成为了大部分开源框架 HA 必不可少的模块）。在 Zookeeper 的帮助下，一个 Standalone 的 Flink 集群会同时有多个活着的 JobManager，其中只有一个处于工作状态，其他处于 Standby 状态。当工作中的 JobManager 失去连接后（如宕机或 Crash），Zookeeper 会从 Standby 中选举新的 JobManager 来接管 Flink 集群。

<font color='32CD32'>zookeeper 安装</font>

[zookeeper 集群安装跳转](https://hxqxiaoqi.gitee.io/2020/03/26/zookeeper%20集群搭建/)

<font color='32CD32'>修改配置：conf/flink-conf.yaml</font>

继续之前的配置

```bash
#jobmanager.rpc.address: master	#注释rpc
high-availability:zookeeper                             #指定高可用模式（必须）
high-availability.zookeeper.quorum:master:2181,worker1:2181,worker2:2181  #ZooKeeper仲裁是ZooKeeper服务器的复制组，它提供分布式协调服务（必须）
high-availability.storageDir:hdfs:///flink/ha/       #JobManager元数据保存在文件系统storageDir中，只有指向此状态的指针存储在ZooKeeper中（必须）
```

<font color='32CD32'>修改：conf/masters</font>

```bash
master:8081
worker1:8081
```

<font color='32CD32'>修改：conf/slaves</font>

```bash
master
worker1
worker2
```

<font color='32CD32'>启动</font>

```bash
./bin/start-cluster.sh
```

<font color='32CD32'>查看</font>

```bash
[root@master flink-1.7.1]# jps
10402 ResourceManager
18563 Jps
18261 TaskManagerRunner
2056 NameNode
17754 StandaloneSessionClusterEntrypoint
2252 SecondaryNameNode
6879 QuorumPeerMain

[root@worker1 flink-1.7.1]# jps
1201 DataNode
3938 QuorumPeerMain
6274 NodeManager
10787 StandaloneSessionClusterEntrypoint
11273 TaskManagerRunner
11453 Jps

[root@worker2 flink-1.7.1]# jps
6177 Jps
5988 TaskManagerRunner
2135 QuorumPeerMain
2698 NodeManager
1199 DataNode
```

<font color='32CD32'>测试</font>

1. 登录web界面，查看master在哪台服务器上
2. kill掉master
3. 查看master是否有更改

## Flink on yarn 部署模式

[安装hadoop集群](https://hxqxiaoqi.gitee.io/2020/03/20/hadoop%20完全分布式搭建/)

因Flink强大的灵活性及开箱即用的原则， 因此提交作业分为2种情况：

- yarn seesion
- Flink run

这2者对于现有大数据平台资源使用率有着很大的区别：

- 1.第一种yarn seesion(Start a long-running Flink cluster on YARN)这种方式需要先启动集群，然后在提交作业，接着会向yarn申请一块空间后，资源永远保持不变。如果资源满了，下一个作业就无法提交，只能等到yarn中的其中一个作业执行完成后，释放了资源，那下一个作业才会正常提交.
- 2.第二种Flink run直接在YARN上提交运行Flink作业(Run a Flink job on YARN)，这种方式的好处是一个任务会对应一个job,即每提交一个作业会根据自身的情况，向yarn申请资源，直到作业执行完成，并不会影响下一个作业的正常运行，除非是yarn上面没有任何资源的情况下。

注意事项:如果是平时的本地测试或者开发，可以采用第一种方案；如果是生产环境推荐使用第二种方案；

Flink on yarn模式部署时，不需要对Flink做任何修改配置，只需要将其解压传输到各个节点之上。但如果要实现高可用的方案，这个时候就需要到Flink相应的配置修改参数，具体的配置文件是FLINK_HOME/conf/flink-conf.yaml。

对于Flink on yarn模式，我们并不需要在conf配置下配置 masters和slaves。因为在指定TM的时候可以通过参数“-n”来标识需要启动几个TM;Flink on yarn启动后，如果是在分离式模式你会发现，在所有的节点只会出现一个 YarnSessionClusterEntrypoint进程；如果是客户端模式会出现2个进程一个YarnSessionClusterEntrypoint和一个FlinkYarnSessionCli进程。

```bash
# 客户端模式
./bin/yarn-session.sh -n 2 -jm 1024 -tm 1024

./bin/flink run ./examples/batch/WordCount.jar -input hdfs://192.168.50.63:9000/LICENSE -output hdfs://192.168.50.63:9000/wordcount-result_1.txt

yarn application --list	# 查看所有yarn容器
yarn application -kill application_1550836652097_0002	# 删除指定yarn

# 分离模式
./bin/yarn-session.sh -nm test3 -d

# Flink run 方式提交
./bin/flink run -m yarn-cluster -yn 1 -yjm 1024 -ytm 1024 ./examples/batch/WordCount.jar

hdfs dfs -put LICENSE /	#上传文件到hdfs

./bin/flink run -m yarn-cluster -yn 1 -yjm 1024 -ytm 1024  ./examples/batch/WordCount.jar -input hdfs://192.168.50.63:9000/LICENSE -output hdfs://192.168.50.63:9000/wordcount-result.txt

# 运行到指定的yarn session
./bin/flink run -yid application_1550579025929_62420 ./examples/batch/WordCount.jar -input hdfs://data-hadoop-112-16.bjrs.zybang.com:8020/flume/events-.1539684881482 -output hdfs://data-hadoop-112-16.bjrs.zybang.com:8020/flink/flink-test02.txt
```







