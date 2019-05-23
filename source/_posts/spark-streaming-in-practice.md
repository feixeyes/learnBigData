---
title: Spark Streaming in Practice（一）——读写kafka之写Kafka
date: 2019-05-21 08:32:49
tags:
	- spark 
	- streaming
	- 实践
categories: 技术实践
---

记录Spark Streaming开发中的一些经验，有些是在当前理解下的个人总结，可能有失偏颇。无特别说明，下文中Spark 指Spark Streaming 应用。spark Streaming支持很多数据源，如file、socket但无疑Kakfa是其中最重要的streaming数据源，本文会对其重点研究研究。这一块基本思路是，首先看一下Kafka提供的基本的数据读写的API，然后再看一下Spark对相关API进一步封装提供了哪些功能。

<!-- more -->

## Kafka 基本概念



kafka是一个分布式的消息队列，即可以有生产者和消费者分别发布和订阅消息，kafka对消息进行持久化缓存，以使得生产者和消费者可以异步通信，并支持多个消费者复用消息。就像一个水池，可以有多个上游往水池中注水，也可以有多个下游从水池中取水。



- 数据记录：每条数据记录由 key、 value和 timestamp组成。
- kafka Broker: kafka由一到多个server组成，每个server称为Broker；每台机器上可以有一到多个broker，一组broker由zookeeper连接起来组成一个kafka集群。  Client(consumer或producer)和server(broker)通过**TCP**协议通信。
- Topic：kafka定义Topic的概念来提升消息队列的并行度，即从应用上可以认为每个Topic是一个单独的消息队列。
- Partition：像大多数分布式系统一样，kafka引入Partition(分区)来通过并行执行来提升性能（吞吐等），每个Topic会划分为多个partition，不同的partition分配给不同的broker来处理，所以Broker和Partition是多对多的关系。**Produer 可以决定消息的分区规则，来分配到指定的partition**。 注意：consumer的数量不能比partitions多（相同的consumer group），最多只会有partitions个consumer消费数据。
- Replica（副本）: 像大多数分布式系统一样，kafka引入多副本(Replica)来提升系统容错性，即每个partition会有多个副本，遇到故障时保证还有备份数据可用。**每个partition有一个broker作为leader还有0到多个broker作为follower。此处，kafka采用冷备机制，即leader提供数据的读写，follower只是被动的备份数据，只在leader出现故障时，从多个follower中选出新的leader。注意：因为对于不同的broker可以是不同的partition的leader，所以从整体上达到负载均衡的作用，即每个broker都有成为leader提供数据读写的机会**
- offset：Partition中的每条Message由offset来表示它在这个partition中的偏移量，这个offset不是该Message在partition数据文件中的实际存储位置，而是逻辑上一个值，它唯一确定了partition中的一条Message。因此，可以认为offset是partition中Message的id。实际上，offset是kafka中唯一的元数据。
- 关于数据顺序：kafka只能保证数据在单个partition内是有序的。若只有一个consumer，则能保证Topic级别的有序。
- 持久化：kafka会按照设定的生命期缓存所有的消息，其使用文件存储消息(append only log)，为了减少磁盘写入的次数，broker会将消息暂时缓存在内存中，当消息的个数(或尺寸)达到一定阀值时，再flush到磁盘。
- Segment：为了提升消息随机读的性能，kafka将数据文件分割为较小的segment。并通过索引来提升数据查询的性能，索引文件记录了offset对应的文件及具体位置。为了减少存储，offset存储的是相对于第一个offset的相对便宜，并没有存储所有offset的索引，而是有一定间隔的稀疏索引。
- Consumer Group：在kafka中实际的订阅单位其实是consumer group，即每个Topic中的每条记录只会被一个consumer group消费一次。ps:同一个group中的多个consumer可以在不同的机器上。

![img](http://kafka.apache.org/images/log_anatomy.png)

## Kafka 0.8 Vs. 0.10



0.10 adds the following over 0.8

- Zookeeper connections are discouraged . From 0.10 there won't be any zookeeper connections required . All connections for consuming data will be maintained by consumer API
- new unified Consumer API
- reduced client dependence on zookeeper (offsets stored in Kafka topic)
- Kafka Streams API
- Kafka Connect API
- transport encryption using TLS/SSL
- Kerberos/SASL Authentication support
- Access Control Lists
- timestamps on messages
- client interceptors
- lots and lots of bug fixes and improvements



从上，我们可以看出，0.10版本在0.8版本的基础上除了扩充功能，**主要是对consumer API的重构**。

## Kakfa Producer API

Producer API有如下特性：

1. 有同步和异步两种数据处理方式，用配置producer.type=async/sync来区分，默认是sync。其中异步方式会将数据进行缓存，直到达到缓存时间阈值或batch大小阈值才会进行发送；
2. 用户自定义数据分区函数；
3. 用户自定义数据序列化接口；



在**Kafka 0.8.0 官网文档**上说 可以使用zookeeper来支持 broker的发现，但是在**0.8.0 Producer Example** 文档中，又推荐使用的是**metadata.broker.list** 参数。

{% asset_img image-20190521185118875.png %}

经测试验证，应该使用**metadata.broker.list** 否则会报缺少参数。 

配置的时候不需要配置所有的brokerlist，只需要配置两个(容错)就可以了，kafka会自动做负载均衡，选出合适的leader。





## Spark 写Kafka

spark streaming/spark写Kafka的一个可行方法是，对每个executor 调用创建Kafka Producer 客户端，各分区单独发送数据，样例大体结构如下：

```scala
result.foreachRDD{rdd=>
  rdd.foreachPartition{ iter=>
    // 调用kafka Producer 发送数据
  }
}
```

但需要注意Producer 实例尽量复用，以提升性能。

Cloudera封装好了一个工具帮我们完成该过程，(原始代码已经找不到，是否复用Producer实例未验证)，该工具其实是从 [Jira-4-22](https://issues.apache.org/jira/browse/SPARK-4122) 拆分出来的。Maven坐标如下：

```xml
<groupId>org.cloudera.spark.streaming.kafka</groupId>
<artifactId>spark-kafka-writer</artifactId>
<version>0.1.0</version>
```

从[MavenPository](https://mvnrepository.com/artifact/org.cloudera.spark.streaming.kafka/spark-kafka-writer) 中看到，最新的版本还是0.1.0，更新日期是 2015年，不确定是有更好的方案还是什么原因，造成该工具一直没有升级。

{% asset_img image-20190521162901199.png %}



源码：https://github.com/harishreedharan/spark-streaming-kafka-output

ps: 该项目描述中标注了 "Move to org.cloudera "，所以这很大的可能性是该工具的源码。从代码可以看到，其实现跟上述分析的方案一致。



如下代码样例，该工具封装了隐式转换，导入后DStream类型会添加 *writeToKafka* 函数，并且用户自己添加自定义的**分区函数**。

```scala
import org.cloudera.spark.streaming.kafka.KafkaWriter._

stream.writeToKafka(producerConf,
            (x: String) => new KeyedMessage[String, String](topic,DigestUtils.md5Hex(x), x))
```



kafka的config会透传下去，到kafka原生Producer API。

## 参考资料

* [Kafka：架构简介【转】](https://www.cnblogs.com/seaspring/p/6138080.html)

* [Kafka Topic Partition Replica Assignment实现原理及资源隔离方案](https://www.cnblogs.com/yurunmiao/p/5550906.html)

* [Kafka 0.8.0 官网文档](http://kafka.apache.org/08/documentation.html)
* [0.8.0 Producer Example](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+Producer+Example)

  