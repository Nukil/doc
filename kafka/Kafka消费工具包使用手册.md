# Kafka消费工具包使用手册

标签（空格分隔）： Kafka

@nukil

[TOC]

## 安装

- 如果本地安装了`maven`，请执行`install.bat（windows）`或`install.sh（linux）`，脚本会把该Kafka消费工具包安装到`maven`本地仓库，以后`maven`工程使用`Kafka`时添加以下依赖来操作`Kafka`

  ```properties
  ## 引入0.8 kafka消费工具包示例
  <dependency>
      <groupId>com.netposa.poseidon</groupId>
      <artifactId>kafka-consumer</artifactId>
      <version>0.8-1.0.1</version>
  </dependency>
  ## 引入kafka 客户端依赖
  <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka-clients</artifactId>
      <version>0.8.2.1</version>
  </dependency>
  <dependency>
      <groupId>org.apache.kafka</groupId>
      <artifactId>kafka_2.10</artifactId>
      <version>0.8.2.1</version>
  </dependency>

  ```

- 如果本地没有安装`maven`或者工程为非`maven`工程，请直接将工具包导入工程

## 配置简介

### 0.8

- consumer

  ```properties
  ## 拉取数据线程数，服务内部默认为8
  fetch.thread.number=4
  ## 消费者group id
  group.id=test-140
  ## 工具包debug模式开关
  enable.kafka.consumer.debug=false
  ## topic name
  service.receive.topic.name=kafka_1000W_8
  ## 最大单次拉取数据量
  max.poll.records=100000
  ## 自动偏移设置值，largest可选
  auto.offset.reset=smallest
  ## 接收网络请求缓存大小
  socket.receive.buffer.bytes=1048576
  ## 在试图确定某个partition的leader是否失去他的leader地位之前，需要等待的backoff时间
  refresh.leader.backoff.ms=200
  ## zookeeper集群地址
  zookeeper.connect=192.168.62.200:2181,192.168.62.201:2181,192.168.62.202:2181
  ```

- producer

  ```properties
  ## 最大发送失败数据缓存量，发送失败数据量缓存大于该值时，调用发送接口将无效，需要先处理发送失败数据
  max.error.message.size=10000
  ## 发送工具包debug模式开关
  enable.kafka.producer.debug=false
  ## kafka集群地址
  bootstrap.servers=192.168.62.200:9092,192.168.62.201:9092,192.168.62.202:9092
  ## producer可以用来缓存数据的内存大小。如果数据产生速度大于向broker发送的速度，producer会阻塞或者抛出异常
  buffer.memory=33554432
  ```

### 0.10

- consumer

  ```properties
  ## kafka集群地址
  bootstrap.servers=192.168.62.200:9092,192.168.62.201:9092,192.168.62.202:9092
  ## 消费者group id
  group.id=test-102
  ## 工具包debug模式开关
  enable.kafka.consumer.debug=false
  ## topic name
  service.receive.topic.name=kafka_1000W_10
  ## 最大单次拉取数据量
  max.poll.records=100000
  ## 自动偏移设置值，latest可选
  auto.offset.reset=earliest
  ## 接收网络请求缓存大小
  receive.buffer.bytes=1048576
  ## 拉取数据超时时间
  poll.timeout.ms=3000
  ```

- producer

  与0.8版本保持一致。

## 交互接口说明

- 基本数据结构定义

```java
// consumer拉取数据时返回的基本数据结构
public class RowMessage {
  	// 该数据在kafka里所属的topic和partition信息
    private TopicAndPartition topicAndPartition;
  	// 该数据的消息体
    private byte[] message;
  	// 该数据在partition上的偏移
    private long offset;
}

// producer发送数据时输入的基本数据结构
public class SourceMessage {
  	// 指定发送的topic名字
    private String topic;
  	// 消息的key值
    private String key;
  	// 消息体
    private byte[] message;
}
```

- consumer

```java
public class DirectKafkaConsumer implements Runnable, ConsumerInterface {
  	/*
  	 * 构造函数，输入consumer properties，返回DirectKafkaConsumer类型对象
  	 */
    public DirectKafkaConsumer(Properties properties) {}

  	/*
  	 * 数据拉取函数，无输入，返回RowMessage类型的消息集合
  	 */
    @Override
    public List<RowMessage> fetchMessages() throws Exception {}

  	/*
  	 * 提交offsets函数，输入Map，返回true or false
  	 */
    @Override
    public boolean commitOffsets(Map<TopicAndPartition, Long> offsets) {}
	
  	/*
  	 * 用于关闭consumer
  	 */
    @Override
    public void shutDown() {}
}
```

- producer

```java
public class DirectKafkaProducer implements ProducerInterface {
   	/*
  	 * 构造函数，输入producer properties，返回DirectKafkaProducer类型对象
  	 */
    public DirectKafkaProducer(Properties properties) {}

  	/*
  	 * 发送一条数据，输入SourceMessage消息体，enableConfirm是否需要确认消息发送成功，返回	 List<SourceMessage>数据集合，在开启确认消息发送成功与否的情况下，该返回值如果不为null，则该返回值为前面发送失败的消息集合，客户端需要对这些数据进行手动处理。
  	 */
    @Override
    public List<SourceMessage> sendMessage(SourceMessage message, boolean enableConfirm) {}

  	/*
  	 * 批量发送数据，输入List<SourceMessage>消息体集合，enableConfirm是否需要确认消息发送成功，返回值同上。
  	 */
    @Override
    public List<SourceMessage> sendBatchMessage(List<SourceMessage> messages, boolean enableConfirm) {}

  	/*
  	 * 用于关闭producer
  	 */
    @Override
    public void shutDown() {}
}
```



## example

- 0.8 consumer

  ```java
  public class DirectKafkaConsumerTest {

      public static void main(String[] args) {
          Properties properties = LoadPropers.getSingleInstance().getProperties("kafka-consumer");
          int totalSize = 0;
          Map<TopicAndPartition, Long> offsets = new HashMap<>();
          boolean stopped = false;
          try {
            	// 创建 DirectKafkaConsumer对象
              DirectKafkaConsumer consumer = new DirectKafkaConsumer(properties);
            	// 开启数据拉取线程，与0.10 consumer不同之处
              new Thread(consumer).start();
              while (!stopped) {
                	// 拉取数据
                  List<RowMessage> list = consumer.fetchMessages();
                  System.out.println(String.format("fetch message size is %d, total : %d", list.size(), totalSize+=list.size()));
                	// 解析拉取数据中的topicAndPartition和offset信息
                  for(RowMessage message : messages) {
                      if (offsets.containsKey(message.topicAndPartition())) {
                          if (offsets.get(message.topicAndPartition()) < message.offset()) {
                          	offsets.put(message.topicAndPartition(), message.offset());
                      	}
                  	} else {
                      	offsets.put(message.topicAndPartition(), message.offset());
                  	}
              	}
                	// 提交offsets
                  consumer.commitOffsets(offsets);
                  if (list.size() < 1000) {
                      Thread.sleep(1000);
                  }
              }
            	// 关闭kafka consumer
              consumer.shutDown();
          } catch (Exception e) {
              System.out.println(e.getMessage() + e);
          }
      }

  }
  ```

- 0.10 consumer

  ```java
  public class DirectKafkaConsumerTest {
      public static void main(String[] args) {
          Properties propers = LoadPropers.getSingleInstance().getProperties("kafka-consumer");
        	// 创建 DirectKafkaConsumer对象
          DirectKafkaConsumer consumer = new DirectKafkaConsumer(propers);
          Map<TopicAndPartition, Long> offsets = new HashMap<>();
          int total = 0;
          boolean stopped = false;
          while (!stopped) {
            	// 拉取数据
              List<RowMessage> messages = consumer.fetchMessages();
            	// 解析拉取数据中的topicAndPartition和offset信息
              for(RowMessage message : messages) {
                  if (offsets.containsKey(message.topicAndPartition())) {
                      if (offsets.get(message.topicAndPartition()) < message.offset()) {
                          offsets.put(message.topicAndPartition(), message.offset());
                      }
                  } else {
                      offsets.put(message.topicAndPartition(), message.offset());
                  }
              }
              System.out.println(String.format("fetch message size is %d, total size is %d", messages.size(), total+=messages.size()));
              // 提交offsets
              consumer.commitOffsets(offsets);
              if (messages.size() < 1000) {
                  try {
                      TimeUnit.SECONDS.sleep(1);
                  } catch (Exception e) {
  			   	   System.out.println(e.getMessage() + e);
                  }
              }
          }
        	// 关闭kafka consumer
        	consumer.shutDown();
      }
  }
  ```

  ​

- producer

  ```java
  public class DirectKafkaProducerTest {
      public static void main(String[] args) {
        	// 读取kafka producer properties
          Properties properties = LoadPropers.getSingleInstance().getProperties("kafka-producer");
        	// 创建DirectKafkaProducer对象
          DirectKafkaProducer producer = new DirectKafkaProducer(properties);
          for (int i = 1; i <= 10000000; ++i) {
            	// 打印发送数据量
              if (i % 1000 == 0) {
                  System.out.println(i);
              }
            	// 发送单条数据，开启消息发送确认
              List<SourceMessage> list = producer.sendMessage(new SourceMessage("kafka_1000W_8", "" + i, "hello".getBytes()), true);
              if (list.size() > 0) {
                  System.out.println("send message error, now to resend");
                  while (list.size() > 0) {
                    	// 调用批量接口，补发发送失败数据
                      list = producer.sendBatchMessage(list, true);
                      if (list.size() > 0) {
                          try {
                              TimeUnit.SECONDS.sleep(3);
                          } catch (Exception e) {
                              System.out.println(e.getLocalizedMessage());
                          }
                      }
                  }
              }
          }
      }
  }
  ```

  ​



