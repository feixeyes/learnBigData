---
title: 初探Flink之编程模型
date: 2019-05-13 10:03:53
tags: flink

---

我们从官网文档开始，一窥Flink初貌。(https://ci.apache.org/projects/flink/flink-docs-release-1.8/)

在学习Flink时，我们以和spark的对比，作为很重要的一条线来贯穿始终，通过对比两者的异同，来进一步理解分布式计算中需要解决的问题，及其解决方案。

Flink官网对其定义如下：		

Apache Flink is an open source platform for distributed stream and batch data processing.Flink’s core is a streaming dataflow engine that provides data distribution, communication, and fault tolerance for distributed computations over data streams. Flink builds batch processing on top of the streaming engine, overlaying native iteration support, managed memory, and program optimization.



从上述定义，我们了解到：

1. Flink是一个开源的流处理和批处理平台；
2. Flink的核心是流处理引擎；
3. 批处理功能建立在流引擎之上。



**与Spark异同**

1. 从功能定位来看，两者非常非常相似，都同时支持流计算和批处理；
2. Spark 把流处理建立在批处理引擎上，流处理是微批处理计算；
3. Flink把批处理建立在流处理引擎上，批处理是一个有限流。



官网文档给出了学习路径建议：

1. 基础概念学习 ([Dataflow Programming Model](https://ci.apache.org/projects/flink/flink-docs-release-1.8/concepts/programming-model.html)  和 [Distributed Runtime Environment](https://ci.apache.org/projects/flink/flink-docs-release-1.8/concepts/runtime.html). )
2. 学习导引
   - [Implement and run a DataStream application](https://ci.apache.org/projects/flink/flink-docs-release-1.8/tutorials/datastream_api.html)
   - [Setup a local Flink cluster](https://ci.apache.org/projects/flink/flink-docs-release-1.8/tutorials/local_setup.html)

下面我们就按照这个顺利来逐步深入，一窥究竟。

## 整体架构分层



Flink 将整个架构分成如下四层：

* Stateful Stream Processing:  定义了最核心的任务模型和最底层处理函数；
* Core APIs : 定义了批处理和流处理相关的API；
* Table API：定义了一个更高层的面向Table 抽象的API，即数据有schema ，定义了Table上常用的各种操作(select，filter，group by)等；
* SQL ：SQL接口。



![Programming levels of abstraction](https://ci.apache.org/projects/flink/flink-docs-release-1.8/fig/levels_of_abstraction.svg)



与Spark异同

1. 整体结构两者非常类似；

2. Flink中 流处理和批处理更加一致）;

   * Spark中 基于RDD(最核心模型、批处理)创建了DataFrame（Table API）和Streaming(流处理)又分别在DataFrame和Streaming之上创建了 spark sql 和 struct streaming接口

   * 而从上图可以很容易的看出flink对待流处理和批处理的一致性，ps: 阿里在致力于让两者更加一致。

3. Flink各层概念及定位更加清晰，可能更两者的发展有关，Spark是慢慢生长出来的，而Flink是阿里在其幼小时（2015年），大刀阔斧的改出来的，这个时期Spark已经是1.4版本（Spark和Flink 各 release版本见附录），其批处理、流处理及DataFrame已经比较完善。

ps: 只是初步印象，待深入研究结论未必如此。



## 编程模型

![A DataStream program, and its dataflow.](https://ci.apache.org/projects/flink/flink-docs-release-1.8/fig/program_dataflow.svg)



![A parallel dataflow](https://ci.apache.org/projects/flink/flink-docs-release-1.8/fig/parallel_dataflow.svg)





Flink的编程模型，主要是 **Stream** 及其上的 **Transformations**， 其中Stream 定义了分布式数据模型，Transformations 定义了 不同的Stream 所支持的各种操作（从上图也可以看出与Spark代码惊人的相似）。  

**与Spark异同**

1. 两者及其相似，核心都是分布式数据模型，及其上的操作组成；

2. Spark 将操作分为 Transform(Lazy 操作，定义了数据模型的各种变换) 和 Action(真正触发 Job的执行)；

3. 正如两者的核心理念不同，Spark基于批处理RDD模型，Flink 基于流处理 Stream模型。

4. 执行时都会将代码生成 DAG（**directed acyclic graphs**）；

5. 区分不同的操作依赖上游一个分区或多个分区的情况（Spark中窄依赖和宽依赖，Flink中**One-to-one**和**Redistributing**）。

    

   思考：两者一个以流计算为核心，一个以批处理为核心，会是两者不可逾越的鸿沟吗？还是最终会渐渐一致？ 



## 其他概念

### Windows



和批处理不同的是，流处理面对的数据是无限的，所以不能对一个流的所有数据进行处理，因此需要将一个无限流划分出一些有限的窗口，对每个窗口内的数据进行计算。

![Time- and Count Windows](https://ci.apache.org/projects/flink/flink-docs-release-1.8/fig/windows.svg)

* 可以按照时间(如每5秒)也可以按照数据量(如每500条数据)来划分窗口；
* Flink 定义了多种窗口类型以应对不同的业务需求，如 *tumbling windows* (无重叠), *sliding windows* (窗口间有重叠), and *session windows* (定义不活跃时间间隔来划分窗口).



### Time

![Event Time, Ingestion Time, and Processing Time](https://ci.apache.org/projects/flink/flink-docs-release-1.8/fig/event_ingestion_processing_time.svg)



Flink中可以按照以下三种时间来进行数据对齐处理：

* Event Time： 事件时间，通常是日志中打的行为发生时的时间戳；
* Ingestion Time： 事件进入Flink 的时间；
* Processing Time： Flink任务执行时的本地时间。



### Stateful Operations

有些业务场景中，需要对多条Event记录进行汇总统计，而不是逐条记录单独处理。这种情况下的操作成为 Stateful(有状态)的。

在实现上，其实Flink维护了一个类似Key-Value的结构来记录各state。



![State and Partitioning](https://ci.apache.org/projects/flink/flink-docs-release-1.8/fig/state_partitioning.svg)



### Checkpoints 容错

Flink 使用checkpoint和回放（stream replay）机制来做容错机制。

所谓checkpoint即定时的将数据的镜像进行持久化存储。当遇到故障时，从镜像点开始从新计算数据。

需要在checkpoint 保存周期和 故障时数据恢复时间之间做一个权衡。



### 批处理

Flink 将批处理当成一种特殊的流(有限的流)，但Flink也有针对批处理的离线特性做一些优化：

* 批处理不适用 checkpoint 进行容错，遇到故障时从头（对应分区）计算数据；
* Stateful操作使用 in-memory/out-of-core数据结构，而不是使用 k-v 索引；(需要进一步研究)
* 有一些批处理独有的异步API。

## 附录一：Spark各版本发布时间

| Version | Original release date | Latest version |                         Release date                         |
| :-----: | :-------------------: | :------------: | :----------------------------------------------------------: |
|   0.5   |      2012-06-12       |     0.5.1      |                          2012-10-07                          |
|   0.6   |      2012-10-14       |     0.6.2      | 2013-02-07[[36\]](https://en.wikipedia.org/wiki/Apache_Spark#cite_note-37) |
|   0.7   |      2013-02-27       |     0.7.3      |                          2013-07-16                          |
|   0.8   |      2013-09-25       |     0.8.1      |                          2013-12-19                          |
|   0.9   |      2014-02-02       |     0.9.2      |                          2014-07-23                          |
|   1.0   |      2014-05-26       |     1.0.2      |                          2014-08-05                          |
|   1.1   |      2014-09-11       |     1.1.1      |                          2014-11-26                          |
|   1.2   |      2014-12-18       |     1.2.2      |                          2015-04-17                          |
|   1.3   |      2015-03-13       |     1.3.1      |                          2015-04-17                          |
|   1.4   |      2015-06-11       |     1.4.1      |                          2015-07-15                          |
|   1.5   |      2015-09-09       |     1.5.2      |                          2015-11-09                          |
|   1.6   |      2016-01-04       |     1.6.3      |                          2016-11-07                          |





## 附录二： Flink 各版本发布时间

| Version | Original release date | Latest version | Release date |
| :-----: | :-------------------: | :------------: | :----------: |
|   0.9   |      2015-06-24       |     0.9.1      |  2015-09-01  |
|  0.10   |      2015-11-16       |     0.10.2     |  2016-02-11  |
|   1.0   |      2016-03-08       |     1.0.3      |  2016-05-11  |
|   1.1   |      2016-08-08       |     1.1.5      |  2017-03-22  |
|   1.2   |      2017-02-06       |     1.2.1      |  2017-04-26  |
|   1.3   |      2017-06-01       |     1.3.3      |  2018-03-15  |
|   1.4   |      2017-12-12       |     1.4.2      |  2018-03-08  |
|   1.5   |      2018-05-25       |     1.5.6      |  2018-12-26  |
|   1.6   |      2018-08-08       |     1.6.3      |  2018-12-22  |
|   1.7   |      2018-11-30       |     1.7.2      |  2019-02-15  |
| **1.8** |      2019-04-09       |     1.8.0      |  2019-04-09  |

