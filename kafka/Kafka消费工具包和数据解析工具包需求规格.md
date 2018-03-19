# Kafka消费工具包需求规格

标签（空格分隔）： Kafka

*@nukil V1.0*

[TOC]



## Kafka消费工具包

### 需求

- 纯`Java`重构，保证不再受`Scala`版本影响
- 兼容常见`Kafka`版本（0.8和0.10版本）

### 设计方案

#### High-Level API

针对高阶API，即`consumer.poll`，`Kafka` 0.8和0.10两个版本的API返回的数据结构并不相同，0.8版本返回`Map<String, ConsumerRecords<K, V>>`，而0.10版本返回`ConsumerRecords<K, V>`，也就是说，无法通过使用`consumer.poll`实现兼容0.8和0.10版本，`Kafka`提供了另一种通过建立数据流的方式拉取数据，这种API对0.8和0.10版本兼容，设计方案如下：

![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/TIM截图20171130170712.png)

#### Low-Level API

低阶API对0.8和0.10版本的`Kafka`兼容。

什么场景下会用到低阶API：

- 一个消息读取多次
- 在一个处理过程中只消费Partition其中的一部分消息
- 添加事务管理机制以保证消息被处理且仅被处理一次

使用SimpleConsumer有哪些弊端：

- 必须在程序中跟踪offset值
- 必须找出指定Topic Partition中的lead broker
- 必须处理broker的变动

使用SimpleConsumer的步骤：

1. 从所有活跃的broker中找出哪个是指定Topic Partition中的leader broker
2. 找出指定Topic Partition中的所有备份broker
3. 构造请求
4. 发送请求查询数据
5. 处理leader broker变更

使用低阶API目前重构起来较为麻烦，暂时会先选择高阶API重构消费工具包。

## Kafka数据解析工具包

### 需求

针对`IoD`将`PCC`识别完推送到Kafka的人脸人体数据，可以通过工具包直接提取出需要的字段值。

### 设计方案

针对数据解析工具包主要是针对特定的数据（人脸，人体），找到其中各中字段的位置，然后将其中的字段值解析出来即可。