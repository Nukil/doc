# Kafka常见问题及解决方案

标签（空格分隔）： Kafka

*@nukil V1.0*

[TOC]

---

## 为什么 consumer 消费不到数据 
- 问题描述
  使用创建好的`Kafka consumer`消费数据，拉取到的数据一直为0条，服务正常运行没有报错，确认`topic`里有数据。

- 解决方案

  - 使用终端命令消费数据

    在消费命令里加入`--from-beginning`参数 ，或者在命令执行后向该`topic`内继续发数据

  - 客户端消费

     默认情况下，当一个`consumer`第一次启动时，它会忽略`topic`上已经存在的所有数据而只是消费其启动之后生产的数据。如果是这种情况，应该在`consumer`启动后发送新数据到`topic`中。或者设置`consumer`的`auto.offset.reset`为`smallest`（0.8版本）或者`earliest`（0.10版本）。

## Connection reset by peer
- 问题描述
  `Kafka`的日志里频繁出现`Connection reset by peer`异常。
```shell
2017-11-03 16:49:30,146 ERROR [kafka-network-thread-27330-1] network.Processor - Closing socket for /1.2.3.4 because of error
java.io.IOException: Connection reset by peer
    at sun.nio.ch.FileDispatcherImpl.read0(Native Method)
    at sun.nio.ch.SocketDispatcher.read(SocketDispatcher.java:39)
    at sun.nio.ch.IOUtil.readIntoNativeBuffer(IOUtil.java:223)
    at sun.nio.ch.IOUtil.read(IOUtil.java:197)
    at sun.nio.ch.SocketChannelImpl.read(SocketChannelImpl.java:379)
    at kafka.utils.Utils$.read(Utils.scala:380)
    at kafka.network.BoundedByteBufferReceive.readFrom(BoundedByteBufferReceive.scala:54)
    at kafka.network.Processor.read(SocketServer.scala:444)
    at kafka.network.Processor.run(SocketServer.scala:340)
    at java.lang.Thread.run(Thread.java:745)
```

该问题经常在0.8及以下版本内出现，但是在客户端里并没显示出异常信息，不影响使用，在0.8及以下版本当客户端断开连接时，`Kafka`服务端会频繁出现该日志。

- 解决方案 
1. 这是当`Kafka`服务端重置了与客户端的连接时输出的日志，一般由服务端连接达到上限引起的，这其实是老版本`Kafka`的日志等级输出错误，在`0.9.0`版本已经改成`warn`等级。
2. 增大`Kafka`服务端的连接数。

## NoSuchElementException: Either.right.value on left
- 问题描述
  程序启动后，服务创建`Kafka consumer`时抛出`NoSuchElementException: Either.right.value on left`异常，然后卡住不动。
```shell
14-11-2017 16:17:36 CST integrationserver_dg INFO - Exception in thread "main" java.util.NoSuchElementException: Either.right.value on Left
14-11-2017 16:17:36 CST integrationserver_dg INFO - at scala.util.Either$RightProjection.get(Either.scala:454)
14-11-2017 16:17:36 CST integrationserver_dg INFO - at netposa.realtime.kafka.utils.DirectKafkaConsumer.initPartitions(DirectKafkaConsumer.scala:271)
```
- 解决方案
  该问题一般是由`Kafka`相关配置错误引起的，请检查`topic`名称等有没有填写正确。

## Isr副本同步异常

- 问题描述
  检查`Kafka` `offset` 发现`Isr`副本同步异常。
  <center>![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/TIM%E6%88%AA%E5%9B%BE20171114173737.png)</center>

- 原因

  1. 网络受阻

     ```shell
     [2017-11-20 18:21:49,350] WARN [ReplicaFetcherThread-0-1], Error in fetch kafka.server.ReplicaFetcherThread$FetchRequest@bd8ffdb (kafka.server.ReplicaFetcherThread)
     java.io.IOException: Connection to poseidonM1:9092 (id: 1 rack: null) failed
       at kafka.utils.NetworkClientBlockingOps$$anonfun$blockingReady$extension$2.apply (NetworkClientBlockingOps.scala:63)
       at kafka.utils.NetworkClientBlockingOps$$anonfun$blockingReady$extension$2.apply (NetworkClientBlockingOps.scala:59)
       at kafka.utils.NetworkClientBlockingOps$.recursivePoll$1(NetworkClientBlockingOps.scala:112)
       at kafka.utils.NetworkClientBlockingOps$.kafka$utils$NetworkClientBlockingOps$$pollUntil$extension (NetworkClientBlockingOps.scala:120)
       at kafka.utils.NetworkClientBlockingOps$.blockingReady$extension (NetworkClientBlockingOps.scala:59)
       at kafka.server.ReplicaFetcherThread.sendRequest(ReplicaFetcherThread.scala:239)
       at kafka.server.ReplicaFetcherThread.fetch(ReplicaFetcherThread.scala:229)
       at kafka.server.ReplicaFetcherThread.fetch(ReplicaFetcherThread.scala:42)
       at kafka.server.AbstractFetcherThread.processFetchRequest(AbstractFetcherThread.scala:107)
       at kafka.server.AbstractFetcherThread.doWork(AbstractFetcherThread.scala:98)
       at kafka.utils.ShutdownableThread.run(ShutdownableThread.scala:63)
     ```

  2. 磁盘IO过高导致副本无法在`replica.lag.time.max.ms`（默认10s）内完成同步，`Kafka`会把无法完成同步的副本从Isr列表中移除。

  3. 新增副本在同步数据未出现在Isr列表中

  目前正在围绕这几个点排查问题。

## Kafka数据重复消费

- 问题描述

  人脸人体服务使用某一个`group id`将Kafka topic里的数据消费完后，停掉服务，第二天在不更换`group id`的情况下启动服务，数据会重复消费。

- 目前持续跟踪，由于该问题不是100%复现，目前处理优先级较低，但是问题很严重，可能会导致现场数据重复消费



## Kafka无法连接

- 问题描述

  `Kafka`在创建`consumer`对象时失败抛出异常，或者输出`dead for group`信息后卡主不动。

- 解决方案

  在`Kafka`服务端正常运行的情况下，`Kafka`无法连接情况主要有以下两种：

  1. `DNS`解析失败

     ```shell
     [2017-11-21 13:34:35,112] WARN Removing server from bootstrap.servers as DNS resolution failed: PoseidonM1:9092 (org.apache.kafka.clients.ClientUtils)
     [2017-11-21 13:34:37,665] WARN Removing server from bootstrap.servers as DNS resolution failed: PoseidonM2:9092 (org.apache.kafka.clients.ClientUtils)
     [2017-11-21 13:34:40,216] WARN Removing server from bootstrap.servers as DNS resolution failed: PoseidonM3:9092 (org.apache.kafka.clients.ClientUtils)
     Exception in thread "main" org.apache.kafka.common.KafkaException: Failed to construct kafka consumer
       at org.apache.kafka.clients.consumer.KafkaConsumer.<init>(KafkaConsumer.java:702)
       at org.apache.kafka.clients.consumer.KafkaConsumer.<init>(KafkaConsumer.java:587)
       at org.apache.kafka.clients.consumer.KafkaConsumer.<init>(KafkaConsumer.java:569)
       at kafka.PullDataFromKafka.main(PullDataFromKafka.java:30)
     Caused by: org.apache.kafka.common.config.ConfigException: No resolvable bootstrap urls given in bootstrap.servers
       at org.apache.kafka.clients.ClientUtils.parseAndValidateAddresses(ClientUtils.java:59)
       at org.apache.kafka.clients.consumer.KafkaConsumer.<init>(KafkaConsumer.java:620)
     ```

     这种情况出现在未在`Kafka`客户端运行的机器上的`hosts`文件里配上服务端的`IP`和主机名，并且在客户端连接`Kafka`的配置项里，配置服务端地址里的代码里使用了`host:port`的格式，如下：

     ```java
     Properties prop = new Properties();
     prop.put("bootstrap.servers", "PoseidonM1:9092,PoseidonM2:9092,PoseidonM3:9092");
     ```

  2. dead for group (group id)

     ```shell
     [2017-11-21 13:46:08,786] INFO Discovered coordinator poseidonM3:9092 (id: 2147483644 rack: null) for group kafka-consumer-17. (org.apache.kafka.clients.consumer.internals.AbstractCoordinator)
     [2017-11-21 13:46:11,363] INFO Marking the coordinator poseidonM3:9092 (id: 2147483644 rack: null) dead for group kafka-consumer-17 (org.apache.kafka.clients.consumer.internals.AbstractCoordinator)
     ```

     这种情况出现在未在`Kafka`客户端运行的机器上的`hosts`文件里配上服务端的`IP`和主机名，并且在客户端连接`Kafka`的配置项里，配置服务端地址里的代码里使用了`ip:port`的格式，如下：

     ```java
     Properties prop = new Properties();
     prop.put("bootstrap.servers","192.168.62.200:9092,192.168.62.201:9092,192.168.62.202:9092")
     ```

  以上两种情况均可通过修改`hosts`文件，在`hosts`文件里填上`Kafka`集群的`IP`和对应的主机名解决。




## 如何查验证某个topic是否正常工作

- 验证`topic`是否work

  利用`Kakfa`自带的终端命令向topic里发送数据，再读取出来：

  ```shell
  [root@poseidonM2 kafka_2.10-0.10.0.1]# bin/kafka-console-producer.sh --broker-list $BROKER_CONNECT --topic kafka_topic
  hello,world!
  [root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-console-consumer.sh --zookeeper $ZK_CONNECT --topic kafka_topic
  hello,world!
  ```

  如果整个过程没有报错，说明该`topic`可以工作。

- 检查`topic`是否健康

  ```shell
  [root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-topics.sh --zookeeper $ZK_CONNECT --describe --topic kafka_topic
  Topic:kafka_topic	PartitionCount:20	ReplicationFactor:3	Configs:
  	Topic: kafka_topic	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 1	Leader: 3	Replicas: 3,1,2	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 2	Leader: 2	Replicas: 1,2,3	Isr: 2,3
  	Topic: kafka_topic	Partition: 3	Leader: 2	Replicas: 2,1,3	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 4	Leader: 3	Replicas: 3,2,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 5	Leader: 3	Replicas: 1,3,2	Isr: 2,3
  	Topic: kafka_topic	Partition: 6	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 7	Leader: 3	Replicas: 3,1,2	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 8	Leader: 2	Replicas: 1,2,3	Isr: 2,3
  	Topic: kafka_topic	Partition: 9	Leader: 2	Replicas: 2,1,3	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 10	Leader: 3	Replicas: 3,2,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 11	Leader: 3	Replicas: 1,3,2	Isr: 2,3
  	Topic: kafka_topic	Partition: 12	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 13	Leader: 3	Replicas: 3,1,2	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 14	Leader: 2	Replicas: 1,2,3	Isr: 2,3
  	Topic: kafka_topic	Partition: 15	Leader: 2	Replicas: 2,1,3	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 16	Leader: 3	Replicas: 3,2,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 17	Leader: 3	Replicas: 1,3,2	Isr: 2,3
  	Topic: kafka_topic	Partition: 18	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 19	Leader: 3	Replicas: 2,1,3	Isr: 2,3,1
  ```

  该结果可以说明：

  1. 该`topic`有20个`partition`，3个`replicas`
  2. 每个`partition`的`replication`都被分配到哪些`broker`上，并且该 `partition` 的`leader`是谁
  3. 每个`partition`都有哪些`replicas`完成了从`leader`的同步
  4. 从`Isr`角度分析该`topic`是否健康
     - `Isr`为空，说明这个`partition`已经离线无法提供服务了，这种情况在上面没有出现
     - `Isr`有数据，但是`Isr` < `Replicas`，这种情况下对于用户是没有感知的，但是说明有部分`replicas` 已经出问题了，至少是暂时无法和 `leader`同步；比如，图中的`partition2`，`Isr` 只有2和3，说明`replica 1`已经离线
     - `Isr` = `Replicas`，但是`leader`不是`Replicas`中的第一个`replica`，这个说明`leader`是发生过重新选取的，这样可能会导致`brokers`负载不均衡；比如，图中的`partition 19`，`leader`是3，而不是2，说明虽然当前它的所有`replica`都是正常的，但之前发生过重新选举