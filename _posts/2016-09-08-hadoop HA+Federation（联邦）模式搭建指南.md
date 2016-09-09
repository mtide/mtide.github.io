---
layout: post
title: "hadoop HA+Federation（联邦）模式搭建指南"
comments: true
description: "hadoop HA+Federation（联邦）模式搭建指南"
keywords: "storm, hadoop, topology, ha, federation"
category: "BIGDATA"
tag: "hadoop, ferdation, ha"
---

### 简述
***
hadoop 集群一共有4种部署模式，详见[《hadoop 生态圈介绍》](http://www.jianshu.com/p/c3a834e45ae3)。
HA联邦模式解决了单纯HA模式的性能瓶颈（主要指Namenode、ResourceManager），将整个HA集群划分为两个以上的集群，不同的集群之间通过Federation进行连接，使得HA集群拥有了横向扩展的能力。理论上，在该模式下，能够通过增加计算节点以处理无限增长的数据。联邦模式下的配置在原HA模式的基础上做了部分调整。

所有四种模式的部署指南见： 

* [hadoop 伪分布式搭建指南](http://www.jianshu.com/p/38a94bade2b4)
* [hadoop 完全分布式搭建指南](http://www.jianshu.com/p/3a16f8ecf883)
* [hadoop HA高可用集群模式搭建指南](http://www.jianshu.com/p/8a8fb958f11f)
* [hadoop HA+Federation（联邦）模式搭建指南](http://www.jianshu.com/p/ccee07a31ca9)

### 搭建过程
***

##### 系统环境
* Ubuntu 14.04 x64 Server LTS
* Hadoop 2.7.2
* vagrant 模拟4台主机，内存都为2G

##### 集群节点规划

|     IP                     |主机名      | 角色描述 | 集群        |
|: --------------------  :|: ---------- --|: ---------- :|: ---------- :|
|192.168.100.201 | h01.vm.com | namenode-ns1-nn1, zkfc, QuorumPeerMain, resourcemanager | ns1 |
|192.168.100.202 | h02.vm.com | namenode-ns1-nn2, zkfc, QuorumPeerMain, resourcemanager, journalnode, | ns1 |
|192.168.100.203 | h03.vm.com | namenode-ns2-nn3, zkfc, QuorumPeerMain, journalnode, nodemanager, datanode | ns2 |
|192.168.100.204 | h04.vm.com | namenode-ns2-nn4, zkfc, journalnode, nodemanager, datanode| ns2 |

上表中：

1. QuorumPeerMain 是zookeeper集群的入口进程；
2. zkfc 是 Zookeeper FailoverController 的简称，主要用于实现两个NN之间的容灾。
3. resourcemanager 是 yarn 中负责资源协调和管理的进程
4. nodemanager 是 yarn 中单个节点上的代理进程，向 RM 汇报信息，监控该节点资源
5. datanode 是 hdfs 的工作节点，负责实际的数据存储和任务计算
6. journalnode 是QJM模式下两个NN节点同步数据的进程，每个HA集群里面的高可用依赖它
7. ns1,ns2 是集群的逻辑名称
8. nn1,nn2, nn3, nn4 是集群中NN的逻辑名称

> zookeeper 节点需要配置奇数台，一般配置3-7台即可。2000多个节点的集群也仅需要5-9台zk；journalnode与zk类似，也是配置奇数台，且最少需要3台，同样不需要太多；另外zkfc需要在启动namenode的节点上也启动，以保障NN间的心跳机制。

###### 更新软件源索引
* 分别在 h01 h02 h03 h04 操作

```bash
sudo apt-get update
```

###### 安装基础软件
* 分别在 h01 h02 h03 h04 操作

```bash
sudo apt-get install ssh
sudo apt-get install rsync
```

###### 配置主机域名
* 分别在 h01 h02 h03 h04 操作

```bash
sudo vim /etc/hostname # centos系统可能没有该文件，创建即可
h01.vm.com # 该节点主机名
```
将该文件内容修改为对应的主机名，例如 h01.vm.com

###### 域名解析
* 搭建内网DNS服务器（可选，但推荐），可阅读vincent的博文
http://blog.kissdata.com/2014/07/10/ubuntu-dns-bind.html
* 配置 /etc/hosts，将以下代码追加到文件末尾即可（如搭建了DNS服务器，则跳过此步骤）
* 分别在 h01 h02 h03 h04 操作

```bash
sudo vim /etc/hosts
192.168.100.201 h01.vm.com h01
192.168.100.202 h02.vm.com h02
192.168.100.203 h03.vm.com h03
192.168.100.204 h04.vm.com h04
```

> !!! Ubuntu系统，须删掉 /etc/hosts 映射 127.0.1.1/127.0.0.1 !!!
> Check that there isn't an entry for your hostname mapped to 127.0.0.1 or 127.0.1.1 in /etc/hosts (Ubuntu is notorious for this).
> 127.0.1.1 h01.vm.com # must remove
>
> 不然可能会引起 hadoop、zookeeper 节点间通信的问题

###### 时间同步（生产环境中务必配置）
在内网中搭建 ntp 服务器，可阅读vincent的博文
http://blog.kissdata.com/2014/10/28/ubuntu-ntp.html

###### 准备jdk、hadoop和zookeeper软件包
* 须到官方网站下载stable版本
jdk-7u79-linux-x64.tar.gz
hadoop-2.7.2.tar.gz
zookeeper-3.4.8.tar.gz
* 所有的软件包都统一解压到 /home/vagrant/VMBigData 目录下，其中 vagrant 是linux系统的用户名，由于我是使用 vagrant 虚拟的主机，所以默认是 vagrant
* 在 h01 操作

```bash
# 先在其中一台机子操作，后面会使用 scp 命令或者其他方法同步到其他主机
mkdir -p /home/vagrant/VMBigData/hadoop /home/vagrant/VMBigData/java /home/vagrant/VMBigData/zookeeper
tar zxf jdk-7u79-linux-x64.tar.gz -C /home/vagrant/VMBigData/java
tar zxf hadoop-2.7.2.tar.gz -C /home/vagrant/VMBigData/hadoop
tar zxf zookeeper-3.4.8.tar.gz -C /home/vagrant/VMBigData/zookeeper
```

###### 配置软连接，方便以后升级版本
* 在 h01 操作，后面通过 scp 同步到其他主机

```bash
ln -s /home/vagrant/VMBigData/java/jdk1.7.0_79/  /home/vagrant/VMBigData/java/default
ln -s /home/vagrant/VMBigData/hadoop/hadoop-2.7.2/  /home/vagrant/VMBigData/hadoop/default
ln -s /home/vagrant/VMBigData/zookeeper/zookeeper-3.4.8/ /home/vagrant/VMBigData/zookeeper/default
```

###### 配置环境变量
* 分别在 h01 h02 h03 h04 操作

```bash
sudo vim /etc/profile
export HADOOP_HOME=/home/vagrant/VMBigData/hadoop/default
export JAVA_HOME=/home/vagrant/VMBigData/java/default
export PATH=$JAVA_HOME/bin:$HADOOP_HOME/bin:$PATH
source /etc/profile
```

###### 配置免密码ssh登录
hadoop主节点需要能远程登陆集群内的所有节点（包括自己），以执行命令。所以需要配置免密码的ssh登陆。可选的ssh秘钥对生成方式有rsa和dsa两种，这里选择rsa。
* 分别在 h01 h02 h03 h04 ，即4个主节点上操作

```bash
ssh-keygen -t rsa -C "youremail@xx.com"
# 注意在接下来的命令行交互中，直接按回车跳过输入密码
```

* 分别在 h01 h02 h03 h04 上操作。以下命令将本节点的公钥 id_rsa.pub 文件的内容追加到远程主机的 authorized_keys 文件中（默认位于 ~/.ssh/）

```bash
ssh-copy-id vagrant@h01.vm.com # vagrant是远程主机用户名
ssh-copy-id vagrant@h02.vm.com
ssh-copy-id vagrant@h03.vm.com
ssh-copy-id vagrant@h04.vm.com
```

* 在 h01 h02 h03 h04 上测试无密码 ssh 登录到 h01 h02 h03 h04

```bash
ssh h01.vm.com
ssh h02.vm.com
ssh h03.vm.com
ssh h04.vm.com
```

> !!! 注意使用rsa模式生成密钥对时，不要轻易覆盖原来已有的，确定无影响时方可覆盖 !!!

###### 配置从节点
在 slaves 文件中配置的主机即为从节点，将自动运行datanode, nodemanager服务
* 在 h01 操作，后面通过 scp 同步到其他主机

```bash
vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/slaves
h03.vm.com
h04.vm.com
```

> 也可以在不同集群里配置不同的从节点

###### 建立存储数据的相应目录
* 在 h01 操作，后面通过 scp 同步到其他主机

```bash
mkdir -p /home/vagrant/VMBigData/hadoop/data/hdfs/tmp
mkdir -p /home/vagrant/VMBigData/hadoop/data/journal/data
mkdir -p /home/vagrant/VMBigData/hadoop/data/pid
mkdir -p /home/vagrant/VMBigData/hadoop/data/namenode1
mkdir -p /home/vagrant/VMBigData/hadoop/data/namenode2
mkdir -p /home/vagrant/VMBigData/hadoop/data/datanode1
mkdir -p /home/vagrant/VMBigData/hadoop/data/datanode2
mkdir -p /home/vagrant/VMBigData/hadoop/data/local-dirs
mkdir -p /home/vagrant/VMBigData/hadoop/data/log-dirs    
```

###### 配置hadoop参数
在 h01 操作，后面通过 scp 同步到其他主机
* vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/hadoop-env.sh

```bash
# export JAVA_HOME=${JAVA_HOME} # 注意注释掉原来的这行
export JAVA_HOME=/home/vagrant/VMBigData/java/default
export HADOOP_PREFIX=/home/vagrant/VMBigData/hadoop/default
# export HADOOP_PID_DIR=${HADOOP_PID_DIR} # 注意注释掉原来的这行
export HADOOP_PID_DIR=/home/vagrant/VMBigData/hadoop/data/hdfs/pid
export YARN_PID_DIR=/home/vagrant/VMBigData/hadoop/data/hdfs/pid
# export HADOOP_SECURE_DN_PID_DIR=${HADOOP_PID_DIR} # 注意注释掉原来的这行
export HADOOP_SECURE_DN_PID_DIR=${HADOOP_PID_DIR}
```

* vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/mapred-env.sh

```bash
export HADOOP_MAPRED_PID_DIR=/home/vagrant/VMBigData/hadoop/data/hdfs/pid
```

* vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/core-site.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
# <configuration> # 注意此处的修改
<configuration xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include href="/home/vagrant/VMBigData/hadoop/default/etc/hadoop/cmt.xml" /> # 此处引入federation的额外配置文件
  <property> 
    <!-- 指定hdfs的nameservice名称，在 cmt.xml 文件中会引用。注意此处的修改 -->  
    <name>fs.defaultFS</name>  
    <value>viewfs://nsX</value> 
  </property>  
  <!-- 指定hadoop数据存储目录 -->  
  <property> 
    <name>hadoop.tmp.dir</name>  
    <value>/home/vagrant/VMBigData/hadoop/data/hdfs/tmp</value> 
  </property>
  <property> 
    <!-- 注意此处将该配置项从 hdfs-site.xml 文件中迁移过来了 -->
    <name>dfs.journalnode.edits.dir</name>  
    <value>/home/vagrant/VMBigData/hadoop/data/journal/data</value> 
  </property>
  <!-- 指定zookeeper地址 -->  
  <property> 
    <name>ha.zookeeper.quorum</name>  
    <value>h01.vm.com:2181,h02.vm.com:2181,h03.vm.com:2181</value> 
  </property> 
</configuration>
```

* vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/cmt.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <property> 
    <!-- 将 hdfs 的 /view_ns1 目录挂载到 ns1 的NN下管理，整个federation的不同HA集群也是可以读写此目录的，但是在指定路径是需要指定完全路径 -->
    <name>fs.viewfs.mounttable.nsX.link./view_ns1</name>  
    <value>hdfs://ns1</value> 
  </property> 
  <property> 
    <name>fs.viewfs.mounttable.nsX.link./view_ns2</name>  
    <value>hdfs://ns2</value> 
  </property> 
  <property> 
    <!-- 指定 /tmp 目录，许多依赖hdfs的组件可能会用到此目录 -->
    <name>fs.viewfs.mounttable.nsX.link./tmp</name>  
    <value>hdfs://ns1/tmp</value> 
  </property> 
</configuration> 
```

* vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/hdfs-site.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- HDFS-HA 配置 -->
<configuration> 
  <property> 
    <!-- 因为集群规划中只配置了2各datanode节点，所以此处只能设置小于2，因为hadoop默认不允许将不同的副本存放到相同的节点上 -->  
    <name>dfs.replication</name>  
    <value>2</value> 
  </property>  
  
  <property> 
    <!-- 白名单：仅允许以下datanode连接到NN，一行一个，也可以指定一个文件 -->
    <name>dfs.hosts</name>  
    <value>
    <!-- ~/VMBigData/hadoop/default/etc/hadoop/hosts.allow -->
    h01.vm.com
    h02.vm.com
    h03.vm.com
    h04.vm.com
    </value> 
  </property> 
  <property> 
    <!-- 黑名单：不允许以下datanode连接到NN，一行一个，也可以指定一个文件 -->
    <name>dfs.hosts.exclude</name>  
    <value></value> 
  </property> 

  <property> 
    <!-- 集群的命名空间、逻辑名称，可配置多个，但是与 cmt.xml 配置对应 -->
    <name>dfs.nameservices</name>  
    <value>ns1,ns2</value> 
  </property> 
  <property> 
    <!-- 命名空间中所有NameNode的唯一标示。该标识指示集群中有哪些NameNode。目前单个集群最多只能配置两个NameNode -->  
    <name>dfs.ha.namenodes.ns1</name>  
    <value>nn1,nn2</value> 
  </property>  
  <property> 
    <name>dfs.ha.namenodes.ns2</name>  
    <value>nn3,nn4</value> 
  </property>  
  <property> 
    <name>dfs.namenode.rpc-address.ns1.nn1</name>  
    <value>h01.vm.com:9000</value> 
  </property>  
  <property> 
    <name>dfs.namenode.http-address.ns1.nn1</name>  
    <value>h01.vm.com:50070</value> 
  </property>  
  <property> 
    <name>dfs.namenode.rpc-address.ns1.nn2</name>  
    <value>h02.vm.com:9000</value> 
  </property>  
  <property> 
    <name>dfs.namenode.http-address.ns1.nn2</name>  
    <value>h02.vm.com:50070</value> 
  </property>  
  <property> 
    <name>dfs.namenode.rpc-address.ns2.nn3</name>  
    <value>h03.vm.com:9000</value> 
  </property>  
  <property> 
    <name>dfs.namenode.http-address.ns2.nn3</name>  
    <value>h03.vm.com:50070</value> 
  </property>  
  <property> 
    <name>dfs.namenode.rpc-address.ns2.nn4</name>  
    <value>h04.vm.com:9000</value> 
  </property>  
  <property> 
    <name>dfs.namenode.http-address.ns2.nn4</name>  
    <value>h04.vm.com:50070</value> 
  </property>  
 
  <property> 
    <!-- JournalNode URLs，ActiveNameNode 会将 Edit Log 写入这些 JournalNode 所配置的本地目录即 dfs.journalnode.edits.dir -->  
    <name>dfs.namenode.shared.edits.dir</name>  
    <!-- 注意此处的ns1，当配置文件所在节点处于ns1集群时，此处为ns1，当处于ns2集群时，此处为ns2 -->
<value>qjournal://h02.vm.com:8485;h03.vm.com:8485;h04.vm.com:8485/ns1</value> 
  </property>  
  <!-- JournalNode 用于存放 editlog 和其他状态信息的目录 -->  
  <property> 
    <name>dfs.journalnode.edits.dir</name>  
    <value>/home/vagrant/VMBigData/hadoop/data/journal</value> 
  </property>  
  <property> 
    <name>dfs.ha.automatic-failover.enabled</name>  
    <value>true</value> 
  </property>  
  <property> 
    <name>dfs.client.failover.proxy.provider.ns1</name>  
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value> 
  </property>  
  <property> 
    <name>dfs.client.failover.proxy.provider.ns2</name>  
    <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value> 
  </property>  
  <!-- 一种关于 NameNode 的隔离机制(fencing) -->  
  <property> 
    <name>dfs.ha.fencing.methods</name>  
    <value>
        sshfence
        shell(/bin/true)
    </value> 
  </property>  
  <property> 
    <name>dfs.ha.fencing.ssh.private-key-files</name>  
    <value>/home/vagrant/.ssh/id_rsa</value> 
  </property>  
  <property> 
    <name>dfs.ha.fencing.ssh.connect-timeout</name>  
    <value>30000</value> 
  </property>  

  <property> 
    <name>dfs.namenode.name.dir</name>  
    <!-- 创建的namenode文件夹位置，如有多个用逗号隔开。配置多个的话，每一个目录下数据都是相同的，达到数据冗余备份的目的 -->  
    <value>file:///home/vagrant/VMBigData/hadoop/data/namenode1,file:///home/vagrant/VMBigData/hadoop/data/namenode2</value> 
  </property>  
  <property> 
    <name>dfs.datanode.data.dir</name>  
    <!-- 创建的datanode文件夹位置，多个用逗号隔开，实际不存在的目录会被忽略 -->  
    <value>file:///home/vagrant/VMBigData/hadoop/data/datanode1,file:///home/vagrant/VMBigData/hadoop/data/datanode2</value> 
  </property> 
</configuration>
```

* vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/yarn-env.sh

```bash
# export JAVA_HOME=/home/y/libexec/jdk1.6.0/
export JAVA_HOME=/home/vagrant/VMBigData/java/default/
```

* vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/yarn-site.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- YARN-HA 配置 -->
<configuration> 
  <!-- YARN HA 配置开始，与NN HA很相似 -->
  <property> 
    <name>yarn.resourcemanager.zk-address</name>  
    <value>h01.vm.com:2181,h02.vm.com:2181,h03.vm.com:2181</value> 
  </property>  
  <property> 
    <!-- 启用RM的高可用模式 -->
    <name>yarn.resourcemanager.ha.enabled</name>  
    <value>true</value> 
  </property> 
  <property> 
    <!-- 配置HA节点的逻辑名称 -->
    <name>yarn.resourcemanager.ha.rm-ids</name>  
    <value>rm1,rm2</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.hostname.rm1</name>  
    <value>h01.vm.com</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.hostname.rm2</name>  
    <value>h02.vm.com</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.address.rm1</name>  
    <value>h01.vm.com:8032</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.address.rm2</name>  
    <value>h02.vm.com:8032</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.scheduler.address.rm1</name>  
    <value>h01.vm.com:8030</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.scheduler.address.rm2</name>  
    <value>h02.vm.com:8030</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.resource-tracker.address.rm1</name>  
    <value>h01.vm.com:8031</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.resource-tracker.address.rm2</name>  
    <value>h02.vm.com:8031</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.webapp.address.rm1</name>  
    <value>h01.vm.com:8088</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.webapp.address.rm2</name>  
    <value>h02.vm.com:8088</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.ha.automatic-failover.enabled</name>  
    <value>true</value> 
  </property> 
  <property> 
    <!-- 配置集群ID，使得yarn能够在正确的集群上Active -->
    <name>yarn.resourcemanager.cluster-id</name>  
    <value>hd0703-yarn</value> 
  </property> 
  <property> 
    <name>yarn.client.failover-proxy-provider</name>  
    <value>org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider</value> 
  </property> 
  <property> 
    <name>yarn.resourcemanager.recovery.enabled</name>  
    <value>true</value> 
  </property> 
  <property>
    <!-- 两个可选值：org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore 以及 默认值org.apache.hadoop.yarn.server.resourcemanager.recovery.FileSystemRMStateStore -->
    <name>yarn.resourcemanager.store.class</name>  
    <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value> 
  </property> 
  <!-- YARN HA 配置结束 -->

  <property> 
    <name>yarn.log-aggregation-enable</name>  
    <!-- 打开日志聚合功能，这样才能从web界面查看日志 -->  
    <value>true</value> 
  </property>  
  <property> 
    <name>yarn.log-aggregation.retain-seconds</name>  
    <!-- 聚合日志最长保留时间 -->  
    <value>86400</value> 
  </property>  
  <property> 
    <name>yarn.nodemanager.resource.memory-mb</name>  
    <!-- NodeManager总的可用内存，这个要根据实际情况合理配置 -->  
    <value>1024</value> 
  </property>  
  <property> 
    <name>yarn.scheduler.minimum-allocation-mb</name>  
    <!-- MapReduce作业时，每个task最少可申请内存 -->  
    <value>256</value> 
  </property>  
  <property> 
    <name>yarn.scheduler.maximum-allocation-mb</name>  
    <!-- MapReduce作业时，每个task最多可申请内存 -->  
    <value>512</value> 
  </property>  
  <property> 
    <name>yarn.nodemanager.vmem-pmem-ratio</name>  
    <!-- 可申请使用的虚拟内存，相对于实际使用内存大小的倍数。实际生产环境中可设置的大一些，如4.2 -->  
    <value>2.1</value> 
  </property>  
  <property> 
    <name>yarn.nodemanager.vmem-check-enabled</name>  
    <value>false</value> 
  </property>  
  <property> 
    <name>yarn.nodemanager.local-dirs</name>  
    <!-- 中间结果存放位置。注意，这个参数通常会配置多个目录，已分摊磁盘IO负载。 -->  
    <value>/home/vagrant/VMBigData/hadoop/data/localdir1,/home/vagrant/VMBigData/hadoop/data/localdir2</value> 
  </property>  
  <property> 
    <name>yarn.nodemanager.log-dirs</name>  
    <!-- 日志存放位置。注意，这个参数通常会配置多个目录，已分摊磁盘IO负载。 -->  
    <value>/home/vagrant/VMBigData/hadoop/data/hdfs/logdir1,/home/vagrant/VMBigData/hadoop/data/hdfs/logdir2</value> 
  </property>  
  <property> 
    <name>yarn.nodemanager.aux-services</name>  
    <value>mapreduce_shuffle</value> 
  </property>  
  <property> 
    <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>  
    <value>org.apache.hadoop.mapred.ShuffleHandler</value> 
  </property> 
</configuration>
```

* vim /home/vagrant/VMBigData/hadoop/default/etc/hadoop/mapred-site.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration> 
  <property> 
    <name>mapreduce.framework.name</name>  
    <value>yarn</value> 
  </property>  
  <property> 
    <name>yarn.app.mapreduce.am.resource.mb</name>  
    <!-- 默认值为 1536,可根据需要调整，调小一些也是可接受的 -->  
    <value>512</value> 
  </property>  
  <property> 
    <name>mapreduce.map.memory.mb</name>  
    <!-- 每个map task申请的内存，每一次都会实际申请这么多 -->  
    <value>384</value> 
  </property>  
  <property> 
    <name>mapreduce.map.java.opts</name>  
    <!-- 每个map task中的child jvm启动时参数，需要比 mapreduce.map.memory.mb 设置的小一些 -->  
    <!-- 注意：map任务里不一定跑java，可能跑非java（如streaming） -->  
    <value>-Xmx256m</value> 
  </property>  
  <property> 
    <name>mapreduce.reduce.memory.mb</name>  
    <value>384</value> 
  </property>  
  <property> 
    <name>mapreduce.reduce.java.opts</name>  
    <value>-Xmx256m</value> 
  </property>  
  <property> 
    <name>mapreduce.tasktracker.map.tasks.maximum</name>  
    <value>2</value> 
  </property>  
  <property> 
    <name>mapreduce.tasktracker.reduce.tasks.maximum</name>  
    <value>2</value> 
  </property>  
  <property> 
    <name>mapred.child.java.opts</name>  
    <!-- 默认值为 -Xmx200m，生产环境可以设大一些 -->  
    <value>-Xmx384m</value> 
  </property>  
  <property> 
    <name>mapreduce.task.io.sort.mb</name>  
    <!-- 任务内部排序缓冲区大小 -->  
    <value>128</value> 
  </property>  
  <property> 
    <name>mapreduce.task.io.sort.factor</name>  
    <!-- map计算完全后的merge阶段，一次merge时最多可有多少个输入流 -->  
    <value>100</value> 
  </property>  
  <property> 
    <name>mapreduce.reduce.shuffle.parallelcopies</name>  
    <!-- reuduce shuffle阶段并行传输数据的数量 -->  
    <value>50</value> 
  </property>  
  <property> 
    <name>mapreduce.jobhistory.address</name>  
    <value>h01.vm.com:10020</value> 
  </property>  
  <property> 
    <name>mapreduce.jobhistory.webapp.address</name>  
    <value>h01.vm.com:19888</value> 
  </property> 
</configuration>
```

> !!! 特别要注意 !!!
> 在 hdfs-site.xml 文件中的 dfs.namenode.shared.edits.dir 配置项：
> 当配置文件所在节点处于ns1集群时，此处值末尾部分为ns1，当处于ns2集群时，则为ns2

###### 安装配置zookeeper
* 在 h01 操作，后面通过 scp 同步到其他主机

```bash
cd /home/vagrant/VMBigData/zookeeper/default/conf/
cp zoo_sample.cfg zoo.cfg
vim zoo.cfg
# 对该文件做出以下修改
dataDir=/home/vagrant/VMBigData/zookeeper/data/tmp
# 如果无法启动zookeeper，可将以下代码对应的行改为 0.0.0.0:2888:3888
# 注意zookeeper解析该文件很死板，不要输入多余的空格和空行
server.1=h01.vm.com:2888:3888
server.2=h02.vm.com:2888:3888
server.3=h03.vm.com:2888:3888
```

```bash
mkdir -p /home/vagrant/VMBigData/zookeeper/data/tmp
vim /home/vagrant/VMBigData/zookeeper/data/tmp/myid
# 在此文件中输入节点编号，比如h01节点就输入1，h02节点就输入2
```

###### 将hadoop所需文件同步到其他主机
* 在 h01 上操作

```bash
scp -r /home/vagrant/VMBigData vagrant@h02.vm.com:/home/vagrant
scp -r /home/vagrant/VMBigData vagrant@h03.vm.com:/home/vagrant
scp -r /home/vagrant/VMBigData vagrant@h04.vm.com:/home/vagrant
```

> !!! 注意：default 软连接需要重建 !!!

* 修改各节点的 zookeeper 的 /home/vagrant/VMBigData/zookeeper/data/tmp/myid 文件，内容为各节点编号，本例中为 1,2,3

###### 启动zookeeper
* 在 h01 h02 h03 操作

```bash
cd /home/vagrant/VMBigData/zookeeper/default
bin/zkServer.sh start
```

###### 启动JournalNode
* 在任一配置了journalnode的节点操作

```bash
cd /home/vagrant/VMBigData/hadoop/default
sbin/hadoop-daemons.sh --hostnames "h02.vm.com h03.vm.com h04.vm.com" start journalnode
```

###### 格式化namenode
* 在 h01 和 h03 即每个集群其中一台namenode的节点上执行
* 注意需要指定集群ID

```bash
hdfs namenode -format -clusterid hd0703
```

> !!! 注意仅在首次启动时执行，因为此命令会删除hadoop集群所有的数据 !!!

###### 启动格式化后的namenode
* 在已经格式化过的 h01 和 h03 namenode 节点运行

```bash
cd /home/vagrant/VMBigData/hadoop/default
sbin/hadoop-daemons.sh  --hostnames "h01.vm.com h03.vm.com" start namenode
```

###### 同步四个namenode的数据
* 在 h02 和 h04 执行同步

```bash
cd /home/vagrant/VMBigData/hadoop/default
hdfs namenode -bootstrapStandby
```

###### 启动同步后的namenode
* 在已经同步过的 h02 和 h04 namenode 节点运行

```bash
cd /home/vagrant/VMBigData/hadoop/default
sbin/hadoop-daemons.sh  --hostnames "h02.vm.com h04.vm.com" start namenode
```

###### 格式化zkfc
* 在 h01 和 h03 (主namenode) 上操作

```bash
hdfs zkfc -formatZK
```

> !!! 注意仅在首次启动时执行 !!!

###### 启动zkfc
* 在 h01 操作即可

```bash
cd /home/vagrant/VMBigData/hadoop/default
sbin/hadoop-daemons.sh --hostnames "h01.vm.com h02.vm.com h03.vm.com h04.vm.com" start zkfc
# sbin/hadoop-daemons.sh stop zkfc #  停止
```

###### 启动hadoop集群：

*启动hdfs*
* 可在任意主节点执行

```bash
cd /home/vagrant/VMBigData/hadoop/default
sbin/start-dfs.sh
# sbin/stop-dfs.sh # 停止
```

*启动Yarn*
* 在h01 和 h02 即计划搭载 ResourceManager 的节点上操作

```bash
cd /home/vagrant/VMBigData/hadoop/default
sbin/start-yarn.sh
# sbin/stop-yarn.sh# 停止
```

###### 浏览服务启动情况
NameNode1
http://192.168.100.201:50070

NameNode2
http://192.168.100.202:50070

NameNode3
http://192.168.100.203:50070

NameNode4
http://192.168.100.204:50070

ResourceManager1
http://192.168.100.201:8088

ResourceManager2
http://192.168.100.202:8088

Datanode
http://192.168.100.203:50075
http://192.168.100.204:50075

zookeeper
bin/zkServer.sh status

zookeeper命令行
zkCli.sh -server 127.0.0.1:2181

集群状态
bin/hdfs dfsadmin -report

hadoop进程
jps

### 动态添加/删除HA集群
// todo

### 动态添加/删除datanode
// todo
