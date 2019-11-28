---
title: kafka安装
date: 2019-06-11 14:22:37
tags: kafka
---

Apache Kafka is a distributed streaming platform

ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services

<!-- more -->

> 首先讲一下Kafka中涉及的一些名词概念

* Broker: 服务代理.实质上就是kafka集群的一个物理节点.
* Topic: 特定类型的消息流.”消息”是字节的有效负载(payload),话题是消息的分类的名或种子名
* Partition: Topic的子概念.一个Topic可以有多个Partition,但一个Partition只属于一个Topic.此外,Partition则是Consumer消费的基本单元.消费时.每个消费线程最多只能使用一个Partition.一个topic中partition的数量,就是每个user group中消费该topic的最大并行数量.
* UserGroup: 为了便于实现MQ中多播,重复消费等引入的概念.如果ConsumerA和ConsumerB属于同一个UserGroup,那么对于ConsumerA消费过的数据,ConsumerB就不能再消费了.也就是说,同一个user group中的consumer使用同一套offset
* Offset: Offset是专门对于Partition和UserGroup而言的,用于记录某个UserGroup在某个Partition中当前已经消费到达的位置.
* Producer: 生产者,能够发布消息到话题的任何对象.直接向某topic下的某partition发送数据.leader负责主备策略、写入数据、发送ack.
* Consumer: 消费者.可以订阅一个或者多个话题,通过fetch的方式从Broker中拉取数据,从而消费这些已经发布的信息的对象.kafka server不直接负责每个consumer消费到了哪,所以需要client和zk联合维护每个partition读到了哪里,即offset

> 搭建Kafka集群

#### 1. 搭建

如果你的机器可以联机互联网的话就再好不过了,这将节省很多时间而且操作更简单,如果是离线环境则需要将用的tar包下载自行拷贝到目标机器.
我这里以linux的机器为例.

##### 首先安装zookeeper

因为是可以连接互联网,所以我们就不使用kafka自带的zk了,来安装一个吧,反正也很简单
在[zookeeper官网](https://zookeeper.apache.org/index.html)里我们找到了[Download页面](https://www.apache.org/dyn/closer.cgi/zookeeper/),根据里面提供的镜像地址直接是有wget下载
`wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.5.5/apache-zookeeper-3.5.5.tar.gz`
等待zk下载并解压完毕后,进入zk的目录下面`cd apache-zookeeper-3.5.5/conf`
这里包含了所有zk的配置文件(也就三个~)

{% asset_img conf.png %}

我们只需要拷贝一份zoo_sample.cfg并作出修改
```bash
cp zoo_sample.cfg zoo.cfg
vim zoo.cfg
```
编辑之后的内容
```text
tickTime=2000
initLimit=10
syncLimit=5
#目录自行创建
dataDir=/data/zookeeper/zk1/data
dataLogDir=/data/zookeeper/zk1/log
clientPort=2181
server.1=[你的host]:2888:3888
```
其中dataDir和dataLogDir根据你自己创建的目录来写.
如果你想要在一台机器上模拟一个集群启动多个zk节点的话,就重复以上操作多拷贝zoo.cfg文件,例如我需要三个节点的zk集群,
```bash
cp zoo_sample.cfg zoo1.cfg
cp zoo_sample.cfg zoo2.cfg
cp zoo_sample.cfg zoo3.cfg
```
并对每个cfg文件作出修改,数据与日志目录不要重复,并且在每一份文件中都要写入如下内容(端口可自定)
```text
server.1=192.168.178.22:2287:3387
server.2=192.168.178.22:2288:3388
server.3=192.168.178.22:2289:3389
```
然后到每个zk节点的dataDir目录下面创建myid文件
```bash
cd /data/zookeeper/zk1/data
echo "1" > myid
```
三个节点都要创建对应的myid文件,值要根据上面cfg文件中写的server.id一一对应

接下来就是启动zk了
```bash
cd ../zookeeper/bin
./zkServer.sh start ../conf/zoo1.cfg
./zkServer.sh start ../conf/zoo2.cfg
./zkServer.sh start ../conf/zoo3.cfg
# 然后查看zk节点的状态
./zkServer.sh status ../conf/zoo2.cfg
ZooKeeper JMX enabled by default
Using config: ../conf/zoo2.cfg
Mode: leader
```
三个节点的zk集群启动成功,id为2的zk节点是leader

##### 安装Kafka

同样我们在[Kafka官网](http://kafka.apache.org/)里找到[下载页面](http://kafka.apache.org/downloads),然后找到最新的[镜像地址](https://www.apache.org/dyn/closer.cgi?path=/kafka/2.2.1/kafka_2.11-2.2.1.tgz)直接下载压缩包
`wget http://mirrors.tuna.tsinghua.edu.cn/apache/kafka/2.2.1/kafka_2.11-2.2.1.tgz`
下载并解压完成后进到目录里`cd kafka/config`

{% asset_img config.png %}

同样我们假设启动两个broker的kafka集群,拷贝两份`server.properties文件`并做修改
```text
cp server.properties server1.properties
# 修改以下内容
broker.id=1
listeners=PLAINTEXT://192.168.178.22:9092
log.dirs=/data/kafka/logs/kafka-logs-1
zookeeper.connect=192.168.178.22:2182,192.168.178.22:2181,192.168.178.22:2183
...
# 配置直接删除topic
delete.topic.enable=true
# 虽然不明白原因,但是不加这两个启动不起来..
advertised.host.name=192.168.178.22
advertised.port=9092

cp server.properties server2.properties
# 修改以下内容
broker.id=2
listeners=PLAINTEXT://192.168.178.22:9093
log.dirs=/data/kafka/logs/kafka-logs-2
zookeeper.connect=192.168.178.22:2182,192.168.178.22:2181,192.168.178.22:2183
...
delete.topic.enable=true
advertised.host.name=192.168.178.24
advertised.port=9093
```
启动两个kafka的broker
```bash
cd /data/kafka/kafka-2.11/bin
# 将日志写入文件并置于后台启动
kafka-server-start.sh ../config/server1.properties > broker1.log 2>&1 &
kafka-server-start.sh ../config/server2.properties > broker2.log 2>&1 &
```
至此,kafka搭建完成,可以使用`jobs`命令查看一下任务状况
```bash
[root@test-hao config]# jobs
[1]-  Running                 kafka-server-start.sh ../config/server2.properties > broker2.log 2>&1 &  (wd: /data/kafka/kafka_2.11-2.2.1/bin)
[2]+  Running                 kafka-server-start.sh ../config/server1.properties > broker1.log 2>&1 &  (wd: /data/kafka/kafka_2.11-2.2.1/bin)
```


#### 2. 测试Kafka

```text
cd /data/kafka/kafka-2.11/bin
#创建主题
./kafka-topics.sh \
--create \
--bootstrap-server 192.168.178.22:9092,192.168.178.22:9093 \
--replication-factor 2 \
--partitions 3 \
--topic test

# 查看broker的情况
./kafka-topics.sh --describe \
--zookeeper 10.4.121.218:3333 \
--topic my-replicated-topic
Topic:my-replicated-topic	PartitionCount:1	ReplicationFactor:3	Configs:
    # 配置情况：分区数据在节点broker.id=0、主节点是broker.id=1、副本集是broker.id=1,0,2、isr是broker.id=1,0,2
	Topic: my-replicated-topic	Partition: 0	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2

# 生产者
./kafka-console-producer.sh \
--broker-list 192.168.178.22:9092,192.168.178.22:9093 \
--topic test
...
test1
test2
^C

# 消费者
# 在开启一个窗口启动消费者可以看到在实时消费数据
./kafka-console-consumer.sh \
--bootstrap-server 192.168.178.22:9092,192.168.178.22:9093 \
--from-beginning \
--topic test
...
test1
test2
^C

# 删除
./kafka-topics.sh --delete \
--zookeeper 192.168.178.22:2181 \
--topic test
# 如果在server.properties文件中没有添加 delete.topic.enable=true 这一项,topic只是被标记了删除,并没有真正被删除,还需要去zk里进行手动删除,所以我一般是添加这一项的(也不会经常出现需要去删除一个topic的情况)
```
