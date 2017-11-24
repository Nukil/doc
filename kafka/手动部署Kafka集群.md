# 手动部署Kafka集群
标签（空格分隔）： Kafka ZooKeeper

*@nukil V1.0*

[TOC]

-----
## 安装环境
|       IP       | CPU  |  内存  |  主机名   |
| :------------: | :--: | :--: | :----: |
| 192.168.62.142 |  1   | 512M | kafka1 |
| 192.168.62.147 |  1   | 512M | kafka2 |
| 192.168.62.152 |  1   | 512M | kafka3 |
## 安装ZooKeeper
以下操作在三台机器上同时进行。

- 安装Java
```shell
yum list java*
yum -y install java-1.7.0-openjdk*
```
- 通过[Apache ZooKeeper官网][1]获取最新的ZooKeeper，文中使用的是ZooKeeper 3.4.9，3台机器软件目录均为/home。
```shell
# 解压。
tar zxvf zookeeper-3.4.9.tar.gz
# 在快照日志的存储路径创建数据目录
mkdir data
```
- 修改配置文件
```shell
cd /home/zookeeper-3.4.9/conf

[root@kafka1 conf]# ll
total 12
-rw-rw-r--. 1 1001 1001  535 Aug 23  2016 configuration.xsl
-rw-rw-r--. 1 1001 1001 2161 Aug 23  2016 log4j.properties
-rw-rw-r--. 1 1001 1001 1026 Oct 25 04:04 zoo_sample.cfg

```
其中`zoo_sample.cfg`是官方给出的一份配置文件样例，需要将其**复制**或**重命名**为`zoo.cfg`。
```properties
vim zoo.cfg

# 服务器之间或客户端与服务器之间维持心跳的时间间隔，也就是每个 tickTime 时间就会发送一个心跳
tickTime=2000
# 这个配置项是用来配置 Zookeeper Leader接受Follower初始化连接时最长能忍受多少个tickTime间隔数
initLimit=10
# 这个配置项标识 Leader 与Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度
syncLimit=5
# 快照日志的存储路径
dataDir=/home/data
# 户端连接 Zookeeper 服务器的端口，Zookeeper 会监听这个端口，接受客户端的访问请求
clientPort=2181

server.1=192.168.62.142:12888:13888
server.2=192.168.62.147:12888:13888
server.3=192.168.62.152:12888:13888
```
`server.1`这个1是服务器的标识也可以是其他的数字， 表示这个是第几号服务器，用来标识服务器，这个标识要写到快照目录下面myid文件里(即`/home/data/myid`)。192.168.62.142为集群里的IP地址，第一个端口是`master`和`slave`之间的通信端口，默认是2888，第二个端口是`leader`选举的端口，集群刚启动的时候选举或者`leader`挂掉之后进行新的选举的端口，默认是3888。

- 创建myid文件
```sh
# kafka1
echo "1" > /home/data/myid
# kafka2
echo "2" > /home/data/myid
# kafka3
echo "3" > /home/data/myid
```

- 启动ZK
```shell
cd /home/zookeeper-3.4.9/bin
./zkServer.sh start 在三个节点上均执行此操作

# kafka1
[root@kafka1 bin]# ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/zookeeper-3.4.9/bin/../conf/zoo.cfg
Mode: follower 该节点为从节点
# kafka2
[root@kafka2 bin]# ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/zookeeper-3.4.9/bin/../conf/zoo.cfg
Mode: leader 该节点被选举为主节点
# kafka3
[root@kafka3 bin]# ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/zookeeper-3.4.9/bin/../conf/zoo.cfg
Mode: follower 该节点为从节点
```

## 安装Kafka
- 通过[Apache Kafka官网][2]获取Kafka安装包，文中使用的是`kafka_2.10-0.8.2.1`。
```sh
tar zxvf kafka_2.10-0.8.2.1.tgz
cd kafka_2.10-0.8.2.1
```

- 修改Kafka配置文件
```properties
vim /home/kafka_2.10-0.8.2.1/config/server.properties

# 当前机器在集群中的唯一标识，和zookeeper的myid性质一样
broker.id=1 
# 当前kafka对外提供服务的端口默认是9092
port=9092
# 这个参数默认是关闭的，在0.8.1有个bug，DNS解析问题，失败率的问题。
host.name=192.168.62.142 
# borker进行网络处理的线程数
num.network.threads=3 
# borker进行I/O处理的线程数
num.io.threads=8 
# 消息存放的目录，这个目录可以配置为","逗号分割的表达式，上面的num.io.threads要大于这个目录的个数这个目录，如果配置多个目录，新创建的topic他把消息持久化的地方是当前以逗号分割的目录中，哪个分区数最少就放那一个
log.dirs=/opt/kafka/kafkalogs/ 
# 发送缓冲区buffer大小，数据不是一下子就发送的，先回存储到缓冲区了到达一定的大小后在发送，能提高性能
socket.send.buffer.bytes=102400 
# kafka接收缓冲区大小，当数据到达一定大小后再序列化到磁盘
socket.receive.buffer.bytes=102400 
# 向kafka请求消息或者向kafka发送消息的请请求的最大数，这个值不能超过java的堆栈大小
socket.request.max.bytes=104857600
# 默认的分区数，一个topic默认1个分区数
num.partitions=1
# 默认消息的最大持久化时间，168小时，7天
log.retention.hours=168
# 消息保存的最大值5M
message.max.byte=5242880  
# kafka保存消息的副本数，如果一个副本失效了，另一个还可以继续提供服务
default.replication.factor=2
# 取消息的最大直接数
replica.fetch.max.bytes=5242880  
# kafka的消息是以追加的形式落地到文件，当超过这个值的时候，kafka会新起一个文件
log.segment.bytes=1073741824 
# 每隔300000毫秒去检查上面配置的log失效时间（log.retention.hours=168 ），到目录查看是否有过期的消息如果有，删除
log.retention.check.interval.ms=300000 
# 是否启用log压缩，一般不用启用，启用的话可以提高性能
log.cleaner.enable=false 
# 设置zookeeper的连接端口
zookeeper.connect=kafka1:2181,kafka2:2181,kafka3:2181 
```

- 启动Kafka
```shell
cd /home/kafka_2.10-0.8.2.1
bin/kafka-server-start.sh config/server.properties

# 新建一个topic
[root@kafka1 kafka_2.10-0.8.2.1]# bin/kafka-topics.sh --zookeeper kafka1:2181 --create --replicaton-facotr 1 --partitions 1 --topic 
kafka_topic
[root@kafka1 kafka_2.10-0.8.2.1]# bin/kafka-topics.sh --zookeeper kafka1:2181 --list
kafka_topic

# 在其他两个节点上查看
[root@kafka2 kafka_2.10-0.8.2.1]# bin/kafka-topics.sh --zookeeper kafka1:2181 --list
kafka_topic
[root@kafka3 kafka_2.10-0.8.2.1]# bin/kafka-topics.sh --zookeeper kafka1:2181 --list
kafka_topic
```

## 常见问题及解决方案
### 问题1：Host Unreachable
<center>![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/20171030103857.png)</center>
解决方法：

- 在集群的每个节点的/etc/hosts文件里对其他两个节点做DNS
- 关闭每个节点的防火墙
```sh
# 暂时关闭，立即生效，重启后失效
service iptables stop
# 永久关闭，重启后生效
chkconfig iptables off 
```

### 问题2：Connection Refused
<center>![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/20171030105702.png)</center>

这是由于只启动了一个zk节点导致的，只要把集群里的其他节点启动起来即可解决。

[1]: http://zookeeper.apache.org/releases.html#download
[2]: http://kafka.apache.org/downloads