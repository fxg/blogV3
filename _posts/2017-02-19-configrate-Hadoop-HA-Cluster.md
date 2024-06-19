---
layout: post
title:  "Hadoop 高可用集群配置"
date:   2017-02-19 17:22:18 +0800
categories: [技术]
tags: [Hadoop]
---

> 本文成文过久，请谨慎参考。
{: .prompt-info }

- Table of Contents
{:toc}

## 环境描述
* 关闭iptables，如果有的话
* IP列表：

|IP|主机名|功能|
|-|-|-|
|192.168.1.200|hadoopmaster|Zookeeper, NameNode, DataNode, ResourceManager, NodeManager, Drillbit, kafka|
|192.168.1.201|hadoopslave1|Zookeeper, NameNode, DataNode, NodeManager, Drillbit, kafka|
|192.168.1.202|hadoopslave2|Zookeeper, DataNode, NodeManager, Drillbit, kafka|

* hadoop系统命令执行用户： root
* 操作用户： hadoop，所有hadoop命令均以sudo执行
* root，hadoop 用户在三台机器间做无密码访问
* 软件版本：

|软件|版本|
|-|-|
|Ubuntu Server|16.04|
|java|1.7稳定版|
|zookeeper|3.4.9|
|hadoop|2.7.3|
|Drill|1.9.0|
|kafka|2.11-0.10.1.1|

## Zookeeper 安装配置
### 配置
编辑Zookeeper配置文件：
```shell
vim zookeeper/conf/zoo.cfg
```
修改为如下配置：
```ini
tickTime=2000
initLimit=10
syncLimit=5
clientPort=2181
dataLogDir=/usr/local/zookeeper/logs
dataDir=/usr/local/zookeeper/data
server.1=192.168.1.200:2888:3888
server.2=192.168.1.201:2888:3888
server.3=192.168.1.202:2888:3888
```
在dataDir对应的文件夹下，建立myid文件，将配置中的server.\*对应的id，写入此文件，然后将整个文件复制到**所有需要安装zookeeper的机器**。
### 环境变量
```shell
vim /etc/profile
```
增加如下类似内容：
```shell
export ZOOKEEEPER_HOME=/home/hadoop/zookeeper
export PATH=$ZOOKEEEPER_HOME/bin:$PATH
```

文档链接：http://zookeeper.apache.org/doc/trunk/
### 启动
在安装zookeeper的机器上都启动：
```shell
sudo zookeeper/bin/zkServer.sh start
```
查看zookeeper的状态：
```shell
sudo zookeeper/bin/zkServer.sh status
```
**注意事项**
1. server.\*对应的IP设置，各台机器本地配置需要修改为**0.0.0.0**；
2. /etc/hosts文件中要把localhost和主机名分开，或者127.0.0.1不要关联主机名，只留localhost。

## Hadoop 安装配置
Hadoop的配置文件都在：HADOOP_HOME 下的etc/hadoop/下。
### 1. 设置环境变量
解压 Hadoop 2.7.3 后，在 /etc/profile 中设置 HADOOP_HOME，并增加到 PATH 中：
```shell
vim /etc/profile
```
增加如下类似内容：
```shell
export HADOOP_HOME=/home/hadoop/hadoop
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
```
文档链接：http://hadoop.apache.org/docs/stable/

如果 JAVA_HOME 没有设置，还需要再次设置。如果系统安装了多个版本的JDK，可以在 HADOOP_HOME 下的 etc/hadoop/hadoop-env.sh 文件中明确指定JAVA_HOME：
```shell
export JAVA_HOME=/usr/lib/jvm/java-7-oracle
```
### 2. core-site.xml

```xml
<configuration>
  <property>
    <!-- 指定hadoop临时目录 -->
    <name>hadoop.tmp.dir</name>
    <value>/home/hadoop/hadoop/tmp</value>
    <description>Abase for other temporary directories.</description>
  </property>
  <property>
    <!-- 指定hdfs的nameservice为hadoop-cluster -->
    <name>fs.defaultFS</name>
    <value>hdfs://hadoop-cluster</value>
  </property>
  <property>
    <!-- 指定zookeeper地址 -->
    <name>ha.zookeeper.quorum</name>
    <value>hadoopmaster:2181,hadoopslave1:2181,hadoopslave2:2181</value>
  </property>
  <property>
    <name>ha.zookeeper.session-timeout.ms</name>
    <value>1000</value>
  </property>
  <!-- HUE设置，如果没有安装可以忽略 -->
  <property>
    <name>hadoop.proxyuser.hue.hosts</name>
    <value>*</value>
  </property>
  <property>
    <name>hadoop.proxyuser.hue.groups</name>
    <value>*</value>
  </property>
</configuration>
```
### 3. hdfs-site.xml
```xml
<configuration>
  <property>
    <!-- 启动 webhdfs -->
    <name>dfs.webhdfs.enabled</name>
    <value>true</value>
  </property>
  <property>
    <!-- 文件复制份数 -->
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.permissions.enabled</name>
    <value>false</value>
  </property>
  <property>
    <!--指定hdfs的nameservice为 hadoop-cluster，需要和core-site.xml中的保持一致 -->
    <name>dfs.nameservices</name>
    <value>hadoop-cluster</value>
  </property>
  <property>
    <!-- hadoop-cluster 下面有两个 NameNode，分别是 nn1，nn2 -->
    <name>dfs.ha.namenodes.hadoop-cluster</name>
    <value>nn1,nn2</value>
  </property>
  <property>
    <!-- nn1的RPC通信地址 -->
    <name>dfs.namenode.rpc-address.hadoop-cluster.nn1</name>
    <value>hadoopmaster:9000</value>
  </property>
  <property>
    <!-- nn1的http通信地址 -->
    <name>dfs.namenode.http-address.hadoop-cluster.nn1</name>
    <value>hadoopmaster:50070</value>
  </property>
  <property>
    <name>dfs.namenode.rpc-address.hadoop-cluster.nn2</name>
    <value>hadoopslave1:9000</value>
  </property>
  <property>
    <name>dfs.namenode.http-address.hadoop-cluster.nn2</name>
    <value>hadoopslave1:50070</value>
  </property>
  <property>
    <!-- 指定 NameNode 的元数据在 JournalNode 上的存放位置，这是NameNode读写JournalNode组的uri。通过这个uri，NameNodes可以读写edit log内容。URI的格式"qjournal://host1:port1;host2:port2;host3:port3/journalId"。这里的host1、host2、host3指的是Journal Node的地址，这里必须是奇数个,至少3个；其中journalId是集群的唯一标识符，对于多个联邦命名空间，也使用同一个journalId。 -->
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://hadoopmaster:8485;hadoopslave1:8485;hadoopslave2:8485/hadoop-cluster</value>
  </property>
  <property>
    <!-- 指定 JournalNode 在本地磁盘存放数据的位置 -->
    <name>dfs.journalnode.edits.dir</name>
    <value>/home/hadoop/hadoop/journal</value>
  </property>
  <property>
    <!-- 开启NameNode失败自动切换 -->
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property>
  <property>
    <!-- 配置失败自动切换实现方式 -->
    <name>dfs.client.failover.proxy.provider.hadoop-cluster</name>
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>
  <property>
    <!-- 配置隔离机制 -->
    <name>dfs.ha.fencing.methods</name>
    <value>shell(/bin/true)</value>
  </property>
  <property>
    <!-- 使用隔离机制时需要ssh免登陆 -->
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/home/hadoop/.ssh/id_rsa</value>
  </property>
</configuration>
```
### 4. mapred-site.xml
```xml
<configuration>
  <property>
    <!-- 指定mr框架为yarn方式 -->
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <!-- 指定jobhistory -->
  <property>
    <name>mapreduce.jobhistory.address</name>
    <value>hadoopmaster:10020</value>
  </property>
  <property>
    <name>mapreduce.jobhistory.webapp.address</name>
    <value>hadoopmaster:19888</value>
  </property>
</configuration>
```
### 5. yarn-site.xml
```xml
<configuration>
  <property>
    <!-- 指定resourcemanager地址 -->
    <name>yarn.resourcemanager.hostname</name>
    <value>hadoopmaster</value>
  </property>
  <property>
    <!-- 指定nodemanager启动时加载server的方式为shuffle server -->
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
</configuration>
```
### 6. slaves
```
hadoopmaster
hadoopslave1
hadoopslave2
```
### 7. 将编辑好的hadoop目录复制到其他机器。
### 8. 第一次启动：
* 1). 所有节点启动zookeeper，如果还没启动；
* 2). 所有节点启动执行：sudo hadoop-daemon.sh start journalnode。**注意只有第一次需要这么启动，之后start-dfs.sh有包含journalnode的启动**;
* 3). 格式化主NameNode：sudo hdfs namenode –format。**这个过程仅仅在第一次使用之前执行一次**;
* 4). 主NameNode上执行：sudo hdfs zkfc -formatZK;
* 5). 备NameNode上执行：sudo hdfs namenode -bootstrapStandby。其实就是把主NameNode的元数据目录 file://${hadoop.tmp.dir}/dfs/name 拷贝一份到备NameNode主机同样位置；
* 6). 主NameNode上执行：sudo start-dfs.sh；
* 7). 主NameNode上执行：sudo start-yarn.sh；
* 8). 主NameNode上启动MR-history-server：sudo mr-jobhistory-daemon.sh start historyserver，查看状态：http://192.168.1.200:19888/jobhistory；
* 9). 测试系统是否可用：
  * hadoop fs -put ./test.txt /
  * hadoop jar ~/hadoop-2.7.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-2.7.1.jar wordcount /test.txt /out
  * hadoop fs -ls /out
  * hadoop fs -cat /out/part-r-00000

## Drill 安装配置
### 配置
如果是高可用Hadoop，需要将hdfs-site.xml，复制到Drill的conf目录，在hdfs的storage的配置中，hdfs:// 后边可以直接赋值集群名称，实现hdfs的高可用访问，sys.store.provider.zk.blobroot的hdfs对应集群名字。
建立hdfs目录：
```shell
hadoop fs -mkdir /user/zookeeper/pstore_drill
```
编辑conf/drill-override.conf:
```conf
drill.exec: {
  cluster-id: "hadoop-cluster",
  zk.connect: "hadoopmaster:2181,hadoopslave1:2181,hadoopslave2:2181",
  sys.store.provider.zk.blobroot: "hdfs://hadoop-cluster/user/zookeeper/pstore_drill"
}
```
复制此文件夹到所有机器。

文档链接：https://drill.apache.org/docs/
### 环境变量
```shell
vim /etc/profile
```
增加如下类似内容：
```shell
export DRILL_HOME=/home/hadoop/drill
export PATH=$DRILL_HOME/bin:$PATH
```
### 运行
每台机器都要启动：sudo bin/drillbit.sh start。注意要先启动zookeeper。
### 使用
命令行使用Drill：
```shell
sudo drill/bin/drill-conf
```
Web访问，任意一台节点的8047端口：
http://192.168.1.200:8047/

## Kafka 安装配置
### 配置
编辑config/server.properties
```ini
broker.id=0 #整个集群内唯一id号，整数，一般从0开始
listeners=PLAINTEXT://hostname:9092
advertised.listeners=PLAINTEXT://hostname:9092
log.dirs=/home/hadoop/kafka/logs #kafka存储数据的目录
zookeeper.connect=hadoopmaster:2181,hadoopslave1:2181,hadoopslave2:2181 #zookeeper 集群列表
```
复制文件到各个机器，并修改broker.id及对应的hostname，每台机器的broker.id都是唯一的，从0开始编号。

**关于listeners和advertised.listeners：**
如果advertised.listeners设置后，kafka就会忽略listeners配置，所以，可以只设置advertised.listeners。并且如果需要**外网访问**，则可以将advertised.listeners设置为外网地址即可。关于advertised.listeners这个配置的含义，官网有解释：Listeners to publish to ZooKeeper for clients to use, if different than the listeners above. In IaaS environments, this may need to be different from the interface to which the broker binds. If this is not set, the value for listeners will be used.

文档链接：https://kafka.apache.org/documentation/
### 环境变量
```shell
vim /etc/profile
```
增加如下类似内容：
```shell
export KAFKA_HOME=/home/hadoop/kafka
export PATH=$KAFKA_HOME/bin:$PATH
```
### 运行
每台机器上启动。（注意要先启动zookeeper）:
```shell
sudo bin/kafka-server-start.sh -daemon config/server.properties
```
### 测试
* 创建主题：
```shell
sudo bin/kafka-topics.sh --create --replication-factor 3 --partitions 4 --topic kafkaTopic --zookeeper hadoopmaster:2181,hadoopslave1:2181,hadoopslave2:2181
```
* 查看主题列表
```shell
sudo bin/kafka-topics.sh --list --zookeeper hadoopmaster:2181,hadoopslave1:2181,hadoopslave2:2181
```
* 查看主题详情
```shell
sudo bin/kafka-topics.sh --describe --topic kafkaTopic --zookeeper hadoopmaster:2181,hadoopslave1:2181,hadoopslave2:2181
```
* 启动消费者
```shell
sudo bin/kafka-console-consumer.sh --topic kafkaTopic --bootstrap-server hadoopmaster:9092,hadoopslave1:9092,hadoopslave2:9092 --from-beginning
```
* 启动生产者（另开ssh）
```shell
sudo bin/kafka-console-producer.sh --topic kafkaTopic --broker-list hadoopmaster:9092,hadoopslave1:9092,hadoopslave2:9092
```
此状态下输入内容回车后，消费者端会同时看到内容。
* 关闭任意一台机器上的kafka服务，继续在生产者端输入内容，消费者端依旧能看到对应的内容，说明集群搭建成功。

## 注意事项

#### 1. 禁用IPV6
```shell
vim /etc/sysctl.conf
```
添加：
```ini
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```
保存后：

```shell
sudo sysctl -p
```

#### 2. 参数dfs.ha.fencing.methods
在搭好HA集群之后，想测试一下集群的高可用性，于是先把active的namenode给停掉：

hadoop-daemon.sh stop namenode 或者 直接kill掉该节点namenode的对应进程也可。

但是通过hdfs haadmin -getServiceState master1 查看，发现standby的namenode并没有自动切换成active，直到我把之前kill掉的namenode手动启动才会切换，但是这样就达不到高可用的目的啊。

在网上找了好久才发现原因，原来是在hdfs-site.xml通过参数dfs.ha.fencing.methods来实现，出现故障时通过哪种方式登录到另一个namenode上进行接管工作。如果采用默认的值sshfence的话，设置集群就无法自动切换(下面单独解释)。log信息的是无法连接到standby的namenode。
```xml
<property>
  <name>dfs.ha.fencing.methods</name>
  <value>shell(/bin/true)</value>
</property>
```
修改成上面的值后，问题解决，active的namenode被停掉后秒切到standby的namenode.

扩展阅读：

dfs.ha.fencing.methods参数

系统在任何时候只有一个namenode节点处于active状态。在主备切换的时候，standby namenode会变成active状态，原来的active namenode就不能再处于active状态了，否则两个namenode同时处于active状态会有问题。所以在failover的时候要设置防止2个namenode都处于active状态的方法，可以是java类或者脚本。

fencing的方法目前有两种，sshfence和shell

sshfence方法是指通过ssh登陆到active namenode节点杀掉namenode进程，所以你需要设置ssh无密码登陆，还要保证有杀掉namenode进程的权限。

shell方法是指运行一个shell脚本/命令来防止两个namenode同时处于active，脚本需要自己写。

注意，QJM方式本身就有fencing功能，能保证只有一个namenode能往journalnode上写edits文件，所以是不需要设置fencing的方法就能的。但是，在发生failover的时候，原来的active namenode可能还在接受客户端的读请求，这样客户端很可能读到一些过时的数据（因为新的active namenode的数据已经实时更新了）。因此，还是建议设置fencing方法。如果确实不想设置fencing方法，可以设置一个能返回成功（没有fencing作用）的方法，如“shell(/bin/true)”。这个纯粹为了fencing方法能够成功返回，并不需要真的有fencing作用。这样可以提高系统的可用性，即使在fencing机制失败的时候还能保持系统的可用性。

##### 3. 如果双namenode都是standby状态：
```shell
sudo bin/hdfs zkfc -formatZK
```
