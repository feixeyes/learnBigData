---
title: SparkStreaming反压机制详解
date: 2019-01-04 12:24:03
tags: spark
categories: 技术原理
---

# 背景概念

* DStream： 表示一系列时间序列上连续的RDDs。
* Batch Duration：spark streaming的核心参数，设置流数据被分成多个batch的时间间隔，每个spark引擎处理的就是这个时间间隔内的数据。
* InputDStream：InputDStream继承自DStream，是所有输入流的基类，代表从源接收到的原始数据流DStreams，每一个InputDStream关联到单个Receiver对象，从源数据接收数据并存储到spark内存，等待处理。

<!-- more -->

# 反压是什么
反压可以限制每个batch接收到的消息量

	Spark Streaming 从v1.5开始引入反压机制（back-pressure）,通过动态控制数据接收速率来适配集群数据处理能力。

# 为什么设置反压
如果在一个batch内收到的消息过多，这就需要为executor分配更多内存，可能会导致其他spark streaming应用程序资源分配不足，甚至有OOM的风险。反压机制就可以动态控制batch接收消息速率，来适配集群处理能力。



# 设置

* 开启反压

SparkConf.set("spark.streaming.backpressure.enabled", "true")

* 设置每个kafka partition读取消息的最大速率:

SparkConf.set("spark.streaming.kafka.maxRatePerPartition", "spark.streaming.kafka.maxRatePerPartition")

	这个值要结合spark Streaming处理消息的速率和batchDuration，
	尽量保证读取的每个partition数据在batchDuration时间内处理完，
	这个参数需要不断调整，以做到尽可能高的吞吐量.


# 基本原理



## 速率预估
1. rateController （in DirectKafkaInputDStream）
2. RateController : 继承自StreamingListener. 用于处理BatchCompleted事件(见下图)。
3. RateEstimator
4. PIDRateEstimator



{% asset_img RateController.png %}

## 限流
maxMessagesPerPartition (in DirectKafkaInputDStream.scala)


{% asset_img maxMessagesPerPartition.png %}

com.google.common.util.concurrent.RateLimiter

RateLimiter是guava提供的基于令牌桶算法的实现类，可以非常简单的完成限流特技，并且根据系统的实际情况来调整生成token的速率。



# 需要注意的坑
## 从多个Topic创建Stream
根据 ***maxMessagesPerPartition*** 代码可知，每次根据上一个速率来预估速率，如果多个topic速率相差过大，会造成预估的速率忽大忽小(如下图)，速率很不稳定，整体上接近多个速率的平均值。
{% asset_img inputRate.png %}

## 一个Job多个Stream
由于只能配置一个参数，所以只能配置一个各topic对应partion速率的折中值，若不同topic的partition速率差别太大，则很难两全。

	ps:按小的设，大的会频繁甚至始终命中背压。按大的设，小的起不到背压效果而OOM。

## maxRatePerPartition参数设置

这个是速率，不需要乘以batch duration了。


# 参考文献
1. 再谈Spark Streaming Kafka反压 https://www.jianshu.com/p/c0b724137416
2. Spark Streaming性能优化: 如何在生产环境下应对流数据峰值巨变 https://www.cnblogs.com/itboys/p/6486089.html
3. 
