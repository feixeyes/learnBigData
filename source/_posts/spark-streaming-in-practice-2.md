---
title: Spark Streaming in Practice（二）——读写kafka之读Kafka
date: 2019-05-21 19:12:39
tags: 
	- spark
	- streaming
	- 实践
categories: 技术实践
---

上篇文章我们一起研究了Kafka的特性，kafka Producer API 以及Spark写API的方案，本文我们继续研究Spark 读取Kafka的方案。ps: 读比较复杂所以放在后面研究。同样的我们从Kakfa 的Consumer API开始。

<!-- more -->

## Kafka Consumer APIs

Kafka 在0.8版本中有两层Consumer API，low-level的"simple"API和 high-level API。简单的理解 **simple API 暴露了更多细节(offset)，用户可以做更精细的控制**；**而High-leve API因为封装了底层细节（offset）使用起来更方便**。

### Simple API（0.8）

Simple API 维护了对单个broker的连接，每次请求都有闭合的响应。该API是无状态的即每次请求都是独立的，因此每次请求都需要指定要读取的offset，用户需要自己维护已经处理的offset信息。



该接口是线程安全的。



应用场景主要为：

1. 想要多次读取一条信息；
2. 只消费一部分消息；
3. 用户来确保 exact once 语义。

而使用该API就需要用户有更多的工作：

1. 用户需要自己维护已经处理的offset信息，以知道每次请求要从哪个offset开始；
2. 用户需要自己指定各partition的leader broker；
3. 用户需要处理leader的变更情况。



API原型如下：



```java
class kafka.javaapi.consumer.SimpleConsumer {
  /**
   *  Fetch a set of messages from a topic.
   *
   *  @param request specifies the topic name, topic partition, starting byte offset, maximum bytes to be fetched.
   *  @return a set of fetched messages
   */
  public FetchResponse fetch(request: kafka.javaapi.FetchRequest);
```

调用样例如下，可以看到请求中需要包含其实的 offset。

```java
            FetchRequest req = new FetchRequestBuilder()
                    .clientId(clientName)
                    .addFetch(a_topic, a_partition, readOffset, 100000)
                    .build();
            FetchResponse fetchResponse = consumer.fetch(req);
```



### High-Level API(0.8)

High-level API，隐藏了brokers 连接情况的细节，并且是有状态的即维护了已经消费的offset的情况。

该API将已消费的offset存储在zookeeper上。ps:会以Consumer group_id作为子目录，即以Consumer Group来消费数据。

API 原型如下，我们看到该API会对每个Topic建立一个KafkaStream（迭代器）：



```java
  /**
   *  Create a list of message streams of type T for each topic.
   *
   *  @param topicCountMap  a map of (topic, #streams) pair
   *  @param decoder a decoder that converts from Message to T
   *  @return a map of (topic, list of  KafkaStream) pairs.
   *          The number of items in the list is #streams. Each stream supports
   *          an iterator over message/metadata pairs.
   */
  public <K,V> Map<String, List<KafkaStream<K,V>>> 
    createMessageStreams(Map<String, Integer> topicCountMap, Decoder<K> keyDecoder, Decoder<V> valueDecoder);
  
```

样例如下：

该API会将消费的offset存储在zookeeper中，所以需要给出zk的配置：

```java
private static ConsumerConfig createConsumerConfig(String a_zookeeper, String a_groupId) {
        Properties props = new Properties();
        props.put("zookeeper.connect", a_zookeeper);
        props.put("group.id", a_groupId);
        props.put("zookeeper.session.timeout.ms", "400");
        props.put("zookeeper.sync.time.ms", "200");
        props.put("auto.commit.interval.ms", "1000");
        return new ConsumerConfig(props);
    }
```



第二步即获取KafkaStream 实例：

```java
Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
        topicCountMap.put(topic, new Integer(a_numThreads));
        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
        List<KafkaStream<byte[], byte[]>> streams = consumerMap.get(topic);
```



第三步，用户可以根据迭代器逐个消费数据：



```java
ConsumerIterator<byte[], byte[]> it = m_stream.iterator();
        while (it.hasNext())
            System.out.println("Thread " + m_threadNumber + ": " + new String(it.next().message()));
        System.out.println("Shutting down Thread: " + m_threadNumber);
```

完整样例见：[0.8 Using the High Level Consumer](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example)

### Consumer API(1.0)

Kafka1.0版本中 Consumer API做了较大升级，主要特点如下：

1. 合并了0.8版本中 simple API和High-level API的功能；
2. 维护了一组连接所需broker是的TCP连接；
3. 该API**不是线程安全的**；
4. 可以根据参数来确定是自动更新已消费的offset还是手动更新。



自动更新（commit）offset：

```java
     Properties props = new Properties();
     props.put("bootstrap.servers", "localhost:9092");
     props.put("group.id", "test");
     props.put("enable.auto.commit", "true");
     props.put("auto.commit.interval.ms", "1000");
     props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
     consumer.subscribe(Arrays.asList("foo", "bar"));
     while (true) {
         ConsumerRecords<String, String> records = consumer.poll(100);
         for (ConsumerRecord<String, String> record : records)
             System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
     }
```



手动更新（commit）offset：



```java
     Properties props = new Properties();
     props.put("bootstrap.servers", "localhost:9092");
     props.put("group.id", "test");
     props.put("enable.auto.commit", "false");
     props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
     consumer.subscribe(Arrays.asList("foo", "bar"));
     final int minBatchSize = 200;
     List<ConsumerRecord<String, String>> buffer = new ArrayList<>();
     while (true) {
         ConsumerRecords<String, String> records = consumer.poll(100);
         for (ConsumerRecord<String, String> record : records) {
             buffer.add(record);
         }
         if (buffer.size() >= minBatchSize) {
             insertIntoDb(buffer);
             consumer.commitSync();
             buffer.clear();
         }
     }
```

## Spark 读Kakfa0.8与0.10对比



note that the 0.8 integration is compatible with later 0.9 and 0.10 brokers, but the 0.10 integration is not compatible with earlier brokers.

| [ spark-streaming-kafka-0-8](https://spark.apache.org/docs/2.2.0/streaming-kafka-0-8-integration.html) | [spark-streaming-kafka-0-10](https://spark.apache.org/docs/2.2.0/streaming-kafka-0-10-integration.html) |                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- | ---------------- |
| Broker Version                                               | 0.8.2.1 or higher                                            | 0.10.0 or higher |
| Api Stability                                                | Stable                                                       | Experimental     |
| Language Support                                             | Scala, Java, Python                                          | Scala, Java      |
| Receiver DStream                                             | Yes                                                          | No               |
| Direct DStream                                               | Yes                                                          | Yes              |
| SSL / TLS Support                                            | No                                                           | Yes              |
| Offset Commit Api                                            | No                                                           | Yes              |
| Dynamic Topic Subscription                                   | No                                                           | Yes              |



- Receiver DStream：
- Direct DStream：
- SSL / TLS Support：
- Offset Commit Api：
- Dynamic Topic Subscription：



![image-20190522151509565](/Users/admin/Library/Application Support/typora-user-images/image-20190522151509565.png)



![image-20190522151116173](/Users/admin/Library/Application Support/typora-user-images/image-20190522151116173.png)



![image-20190522151230008](/Users/admin/Library/Application Support/typora-user-images/image-20190522151230008.png)



修改了parition  重启





![image-20190522151622748](/Users/admin/Library/Application Support/typora-user-images/image-20190522151622748.png)



## Spark 读Kafka 0.8



## Spark 读Kafka 0.10



- Spark 读kafka的两（？）种方式
- DrictStream API
- High Level API 
- Spark 监控—— Spark Master UI
- Spark 监控—— Lag监控









- 0.8 Using the High Level Consumer](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example)
- [kakfa0.8 官网文档](https://kafka.apache.org/08/documentation.html)
- [0.8.0 SimpleConsumer Example](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example)
- [Spark Streaming + Kafka Integration Guide](https://spark.apache.org/docs/2.1.0/streaming-kafka-integration.html#spark-streaming-kafka-integration-guide)