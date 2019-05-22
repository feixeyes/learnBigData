---
title: Spark Streaming in Practice（二）——读写kafka之读Kafka
date: 2019-05-21 19:12:39
tags: 
	- spark
	- streaming
	- 实践
---





## Kafka Consumer APIs

We have 2 levels of consumer APIs. The low-level "simple" API maintains a connection to a single broker and has a close correspondence to the network requests sent to the server. This API is completely stateless, with the offset being passed in on every request, allowing the user to maintain this metadata however they choose.

The high-level API hides the details of brokers from the consumer and allows consuming off the cluster of machines without concern for the underlying topology. It also maintains the state of what has been consumed. The high-level API also provides the ability to subscribe to topics that match a filter expression (i.e., either a whitelist or a blacklist regular expression).

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



## Spark 读Kafka 0.8



## Spark 读Kafka 0.10



- Spark 读kafka的两（？）种方式
- DrictStream API
- High Level API 
- Spark 监控—— Spark Master UI
- Spark 监控—— Lag监控









- 0.8 Using the High Level Consumer](https://cwiki.apache.org/confluence/display/KAFKA/Consumer+Group+Example)
- [0.8.0 SimpleConsumer Example](https://cwiki.apache.org/confluence/display/KAFKA/0.8.0+SimpleConsumer+Example)
- 
- [Spark Streaming + Kafka Integration Guide](https://spark.apache.org/docs/2.1.0/streaming-kafka-integration.html#spark-streaming-kafka-integration-guide)