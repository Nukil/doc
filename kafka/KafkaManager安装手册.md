# KafkaManager安装手册

标签（空格分隔）： Kafka

*@nukil V1.0*

[TOC]

## Kafka Manager简介

`Kafka Manager`是一个用于管理`Kafka`集群的第三方工具，它有以下特性：

- 同时管理多个群集
- 检查群集状态（`topics`, `consumers`, `offsets`, `brokers`, `replica` `distribution`, `partition` `distribution`）
- 进行副本选举
- 手动增加`partition`并指定它归属哪个`broker`
- 对分区进行重新分配（基于已经生成的分配好的分区）
- 使用指定的`topic` `config`创建`topic`（0.8.1.1与0.8.2+的版本配置不同）
- 删除主题（仅在0.8.2+版本支持，需要在`broker`中设置`delete.topic.enable = true`）
- `topic`标记为删除（仅支持0.8.2+）
- 批量为多个`topic`的生成`partition`，并为他们指定`broker`
- 批量运行重新分配多个主题的分区
- 将分区添加到现有的`topic`
- 更新现有`topic`的配置
- 为`broker`和`topic`启用`JMX`轮询
- 可以过滤掉在`zookeeper`中没有`id / owners / offset`目录的`topic`

## 安装Kafka Manager

### 安装环境

- [Kafka 0.8../0.9../0.10../0.11..](http://kafka.apache.org/downloads)
- Java 8+

### 安装步骤

- 在[Kafka Manager github主页](https://github.com/yahoo/kafka-manager)下载`Kafka Manager`的最新代码

- 解压源码包并安装

  ```shell
  [root@poseidonM1 KafkaOffsetMonitor]# unzip kafka-manager-master.zip 
  [root@poseidonM1 KafkaOffsetMonitor]# cd kafka-manager-master
  [root@poseidonM1 kafka-manager-master]# ls
  app  build.sbt  conf  img  LICENCE  public  README.md  sbt  src  test
  [root@poseidonM1 kafka-manager-master]# ./sbt clean dist

  # 等待一段时间即可完成编译
  # 编译后会在target/universal目录下生成一个zip包
  [root@poseidonM1 kafka-manager-master]# ls target/universal/
  kafka-manager-1.3.3.14.zip  tmp
  # 解压该zip包
  [root@poseidonM1 kafka-manager-1.3.3.14]# ls
  application.home_IS_UNDEFINED  bin  conf  lib  README.md  share
  ```

## 启动Kafka Manager

### configuration配置策略

打开`conf/application.conf`文件，修改：

```properties
kafka-manager.zkhosts="192.168.62.200:2181,192.168.62.201:2181,192.168.62.202:2181"
```

配置上`zookeeper`集群的地址。

可以通过配置`application.conf`文件选择是否开启以下功能：

```properties
application.features=["KMClusterManagerFeature","KMTopicManagerFeature","KMPreferredReplicaElectionFeature","KMReassignPartitionsFeature"]
# KMClusterManagerFeature 允许从Kafka Manager增加，更新，删除Kafka集群
# KMTopicManagerFeature 允许从Kafka集群增加，更新，删除topic
# KMPreferredReplicaElectionFeature 允许进行副本选举
# KMReassignPartitionsFeature 允许生成partition并分配partition
```

如果`Kafka`集群规模较大，可以开启`JMX`，并在在该配置文件内增加以下配置参数：

```properties
kafka-manager.broker-view-thread-pool-size = <3 * number_of_brokers>
kafka-manager.broker-view-max-queue-size = <3 * total # of partitions across all topics> 
kafka-manager.broker-view-update-seconds = <kafka-manager.broker-view-max-queue-size / (10 * number_of_brokers) 
```

假如有一个10个`broker`，100个`topic`，并且每个`topic`有10个`partition`的集群，配置如下：

```properties
kafka-manager.broker-view-thread-pool-size=30
kafka-manager.broker-view-max-queue-size=3000
kafka-manager.broker-view-update-seconds=30
```

以下配置用于控制consumer的offset的线程池和队列：

```properties
kafka-manager.offset-cache-thread-pool-size = <default is # of processors>
kafka-manager.offset-cache-max-queue-size = <default is 1000>
kafka-manager.kafka-admin-client-thread-pool-size = <default is # of processors>
kafka-manager.kafka-admin-client-max-queue-size = <default is 1000>
```

实际使用中可以扩大`#`所代表的数字，但是需要按照服务器的`cpu`线程数进行合理配置。

### 启动Kafka Manager

```sh
# 使用默认配置启动
$ bin/kafka-manager
# 使用指定的配置文件和端口启动
$ bin/kafka-manager -Dconfig.file=config/application.conf -Dhttp.port=8080
# 指定java home启动
$ bin/kafka-manager -java-home /usr/local/oracle-java-8
# 如果需要增加认证授权，可以通过添加一份使用SASL认证的JAAS的配置文件
$ bin/kafka-manager -Djava.security.auth.login.config=/path/to/my-jaas.conf
```

### 访问Kafka Manager

`Kafka Manager`启动后，可以通过`http://ip:port`进行访问