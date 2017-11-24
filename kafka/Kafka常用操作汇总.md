# Kafka常用操作汇总

标签（空格分隔）： Kafka

*@nukil V1.0*

[TOC]

## 命令行操作
以下操作均在 `Kafka` 的根目录下执行，目录结构如下：

```shell
[root@poseidonM1 kafka_2.10-0.10.0.1]# pwd
/home/likun/kafka_2.10-0.10.0.1
[root@poseidonM1 kafka_2.10-0.10.0.1]# ls
bin  config  libs  LICENSE  logs  NOTICE  site-docs

# ~/.bash_profile文件新增
export ZK_CONNECT=poseidonM1:2181,poseidonM2:2181,poseidonM3:2181
export BROKER_CONNECT=poseidonM1:9092,poseidonM2:9092,poseidonM3:9092

[root@poseidonM1 kafka_2.10-0.10.0.1]# source ~/.bash_profile
```

### 启动Kafka
```shell
bin/kafka-server-start.sh -daemon config/server.properties
```

### 创建

- 创建 `topic`

  ```shell
  # bin/kafka-topics.sh --create --zookeeper [host:port] --replication-factor [replication-factor number] --partitions [partition number] --topic [topic_name]

  bin/kafka-topics.sh --create --zookeeper $ZK_CONNECT --replication-factor 1 --partitions 1 --topic kafka_topic
  Created topic "kafka_topic".
  ```

### 查询
- 查询 `Kafka` 所有的 `topic`

  ```shell
  # kafka-topics.sh --zookeeper [host:port] --list

  [root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-topics.sh --zookeeper $ZK_CONNECT --list
  kafka_topic - marked for deletion
  likun_test - marked for deletion
  ```


- 查看topic的详细信息

  ```shell
  # kafka-topics.sh --zookeeper [host:port] --describe --topic [topic_name]

  [root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-topics.sh --zookeeper $ZK_CONNECT --describe --topic kafka_topic
  Topic:kafka_topic	PartitionCount:20	ReplicationFactor:3	Configs:
  	Topic: kafka_topic	Partition: 0	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 1	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
  	Topic: kafka_topic	Partition: 2	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
  	Topic: kafka_topic	Partition: 3	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
  	Topic: kafka_topic	Partition: 4	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
  	Topic: kafka_topic	Partition: 5	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
  	Topic: kafka_topic	Partition: 6	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 7	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
  	Topic: kafka_topic	Partition: 8	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
  	Topic: kafka_topic	Partition: 9	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
  	Topic: kafka_topic	Partition: 10	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
  	Topic: kafka_topic	Partition: 11	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
  	Topic: kafka_topic	Partition: 12	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 13	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
  	Topic: kafka_topic	Partition: 14	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
  	Topic: kafka_topic	Partition: 15	Leader: 2	Replicas: 2,1,3	Isr: 2,1,3
  	Topic: kafka_topic	Partition: 16	Leader: 3	Replicas: 3,2,1	Isr: 3,2,1
  	Topic: kafka_topic	Partition: 17	Leader: 1	Replicas: 1,3,2	Isr: 1,3,2
  	Topic: kafka_topic	Partition: 18	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
  	Topic: kafka_topic	Partition: 19	Leader: 3	Replicas: 3,1,2	Isr: 3,1,2
  ```

  1. Leader

     `Leader` 是给出的所有 `Partitons` 中负责读写的节点，每个节点都有可能成为`Leader`。

  2. Replicas

     `replicas` 显示给定 `Partitons` 所有副本所存储节点的节点列表，不管该节点是否是 `Leader` 或者是否存活。

  3. Isr

     副本都已同步的的节点集合，这个集合中的所有节点都是存活状态，并且跟`Leader`同步。

- 查看某个topic有哪些消费者group

  ```shell
  # kafka-consumer-groups.sh --new-consumer --bootstrap-server [host:port] --list
  # 该方式列出的是在broker上活跃的group id
  [root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-consumer-groups.sh --new-consumer --bootstrap-server localhost:9092 --list
  kafka-consumer

  # bin/kafka-consumer-groups.sh --zookeeper localhost:2181 --list
  # 该方式列出的是记录在zookeeper上的group id
  [root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-consumer-groups.sh --zookeeper localhost:2181 --list
  console-consumer-38977
  console-consumer-17634
  console-consumer-93660
  console-consumer-22534
  console-consumer-3685
  console-consumer-68768
  console-consumer-93811
  ```


- 查看某个group id的消费情况

  ```shell
  # bin/kafka-consumer-offset-checker.sh --topic [topic_name] --group [group_id] --zookeeper [host:port] (0.8版本)
  [root@poseidonM3 kafka_2.10-0.10.0.1]# bin/kafka-consumer-offset-checker.sh --topic kafka_topic --group kafka-consumer --zookeeper $ZK_CONNECT
  Group           Topic          	Pid       Offset          logSize         Lag      Owner
  kafka-consumer  kafka_topic    	 0         4               4               0        none
  kafka-consumer  kafka_topic    	 1         5               6               1        none
  kafka-consumer  kafka_topic    	 2         3               3               0        none
  kafka-consumer  kafka_topic    	 3         9               9               0        none
  kafka-consumer  kafka_topic    	 4         0               8               8        none
  kafka-consumer  kafka_topic    	 5         5               5               0        none
  kafka-consumer  kafka_topic    	 6         3               5               2        none
  kafka-consumer  kafka_topic    	 7         8               8               0        none
  kafka-consumer  kafka_topic    	 8         11              12              1        none
  kafka-consumer  kafka_topic    	 9         3               3               0        none
  kafka-consumer  kafka_topic    	 10        0               8               8        none
  kafka-consumer  kafka_topic    	 11        7               7               0        none
  kafka-consumer  kafka_topic    	 12        8               9               1        none
  kafka-consumer  kafka_topic    	 13        7               11              4        none
  kafka-consumer  kafka_topic    	 14        5               6               1        none
  kafka-consumer  kafka_topic    	 15        0               8               8        none
  kafka-consumer  kafka_topic    	 16        0               13              13       none
  kafka-consumer  kafka_topic    	 17        4               4               0        none
  kafka-consumer  kafka_topic    	 18        8               8               0        none
  kafka-consumer  kafka_topic    	 19        0               7               7        none

  # kafka-consumer-groups.sh --zookeeper [host:port] --describe --group [groupid] (0.10版本)
  [root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-consumer-groups.sh --zookeeper localhost:2181 --describe --group console-consumer
  GROUP             TOPIC       PARTITION  CURRENT-OFFSET  LOG-END-OFFSET      LAG     OWNER
  console-consumer  kafka_topic     0            4               145           141      none
  console-consumer  kafka_topic     1            7               166           159      none
  console-consumer  kafka_topic     2            3               157           154      none
  console-consumer  kafka_topic     3            9               168           159      none
  console-consumer  kafka_topic     4            8               164           156      none
  console-consumer  kafka_topic     5            5               155           150      none
  console-consumer  kafka_topic     6            5               116           111      none
  console-consumer  kafka_topic     7            8               173           165      none
  console-consumer  kafka_topic     8            12              189           177      none
  console-consumer  kafka_topic     9            3               132           129      none
  console-consumer  kafka_topic     10           8               164           156      none
  console-consumer  kafka_topic     11           8               185           177      none
  console-consumer  kafka_topic     12           9               165           156      none
  console-consumer  kafka_topic     13           11              161           150      none
  console-consumer  kafka_topic     14           6               144           138      none
  console-consumer  kafka_topic     15           9               150           141      none
  console-consumer  kafka_topic     16           13              185           172      none
  console-consumer  kafka_topic     17           4               109           105      none
  console-consumer  kafka_topic     18           8               158           150      none
  console-consumer  kafka_topic     19           7               165           158      none
  ```

  1. Offset / CURRENT-OFFSET

     当前消费位置。

  2. logSize / LOG-END-OFFSET

     数据总量。

  3. Lag / LAG

     剩余未消费数据量。


### 删除
- 删除 `topic`

  ```shell
  # bin/kafka-topics.sh --delete --zookeeper $ZK_CONNECT --topic kafka_topic

  [root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-topics.sh --delete --zookeeper $ZK_CONNECT --topic kafka_topic
  Topic kafka_topic is marked for deletion.
  Note: This will have no impact if delete.topic.enable is not set to true.
  ```
### 生产消息
```shell
# kafka-console-producer.sh --broker-list [host:port] --topic [topic_name]

bin/kafka-console-producer.sh --broker-list $BROKER_CONNECT --topic kafka_topic
```
启动后，在终端输入输入发送的信息即可。

### 消费数据
```shell
[root@poseidonM1 kafka_2.10-0.10.0.1]# bin/kafka-console-producer.sh --broker-list $BROKER_CONNECT --topic kafka_topic
hello,world

# bin/kafka-console-consumer.sh --zookeeper [host:port] --topic [topic_name]
[root@poseidonM2 kafka_2.10-0.10.0.1]# bin/kafka-console-consumer.sh --zookeeper $ZK_CONNECT --topic kafka_topic
hello,world
```

## Java API操作

未完待续......