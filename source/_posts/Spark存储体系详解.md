---
title: Spark存储体系详解
date: 2019-05-15 10:08:16
tags: spark
categories: 技术原理
---

## 存储体系的职责

在研究Spark存储体系之前，我们先搞清楚一个重要的问题：**对于一个数据计算引擎，存储体系在其中的职责是什么？或者说功能定位是什么？** 而子模块肯定是为整个系统服务的，所以我们可以从 计算引擎(Spark)本身功能来一探究竟。


一个数据计算引擎，最主要的是对数据进行各种加工（转换、过滤、聚合、合并、统计等），我们可以将对数据的加工当成数据状态的转换。所以计算引擎的核心功能就是 定义**数据状态** 及定义**数据状态之上的各种操作**，对应于Spark即是，RDD 及之上的各种Transform和Action操作，所以Spark存储体系应该包括对RDD的存储。

<!-- more -->

输入是外部系统不算Spark本身的存储体系，Spark执行时会将整个作业按照数据依赖情况构建成DAG划分成多个Stage，Stage间涉及到数据的生成和传输(Shuffle阶段) 此处涉及到存储体系，涉及到的功能有数据存储、数据寻址，数据读取和数据远程传输。

在数据处理中，涉及到一个数据对应多路处理的情况，无论用户是否显式的调用Cache，这都涉及到RDD**复用**，即涉及到中间结果的存储，此处会用到存储体系，涉及到的功能也是数据的存储、寻址和读取。

Spark 提供了Broadcast功能可以将小的数据集同步到多个节点上，该功能也涉及数据的存储、寻址、读取和远程传输。

## Spark存储体系主要功能

根据以上分析，我们知道Spark存储体系主要功能为：

* 资源的申请：对于磁盘即创建目录、文件；对于内存即变量的创建、销毁。磁盘资源相对比较丰富，在需要时再去创建目录、文件是完全没问题的，所以不需要太复杂的资源管理。而内存资源是很稀缺的，如果在需要时再去申请很可能出现系统没有足够的内存分配的情况，而如果不加节制的申请也可能自己把系统内存占光，造成其他任务不能执行，这样就会出现很大的不确定性。Spark采用预申请内存资源，自己管理内存资源的方式来确保一个更稳定的内存环境。
* 数据存储、读取：具体数据的读写。ps: Spark定义数据存储的最小单元为Block。
* **数据寻址(单节点、集群中)**：为了区分各Block，每个Block有唯一的标识Id(BlockId)。对于数据的查询，一方面，需要确定在具体哪个节点上（**集群中寻址**）；另一方面，需要确定在具体节点的内存中还是磁盘中，具体路径或引用是什么（**节点中具体数据寻址**）。
* 数据远程传输：Broadcast、Shuffle之类的操作涉及到数据的跨节点传输，所以需要有数据远程传输功能。



## Spark中存储体系相关的类

确定了Spark存储体系所提供的功能，我们再来看看Spark中存储体系相对应的具体的实现类。



* BlockId:  Block是Spark存储体系中，数据管理的基本单位；BlockId是数据块的唯一标识。

* BlockInfo：Block的元数据信息，如存储Level(StorageLevel)等。

* StorageLevel：定义了存储级别，内存还是磁盘，序列化还是非序列化，几个副本。

*  **BlockInfoManager**：使用一个Map来映射BlockId和其对应的BlockInfo; 另外维护了每个Block的读锁和写锁。

* BlockResult：Block的数据结果，包括数据读取接口。

* **BlockManager**：每个节点(driver、executor)中管理数据的接口，屏蔽了底层存储细节（内存、磁盘、序列化方式）。

* BlockManagerId：Blockmanager的唯一标识。

* **BlockMangerMasterEndpoint**：维护集群中所有的BlockManager及其维护的Block
* BlockManagerMaster：各BlockManager与 BlockMangerMasterEndpoint 通信的代理（RPC客户端）。

* **MapOutTracher**：用于找到某Reduce对应的上游Block所在的位置。

* BlockManagerSlaveEndpoint：各BlockManager中对外提供服务的RPC服务端，用于接收对该节点数据处理的请求（主要是删除数据）。

* DiskStore：文件读写。

* MemoryStore：内存读写管理，并根据存储级别将数据转存到磁盘等。

* DiskBlockManager：资源申请，创建目录、文件等。

* MemoryPool：预申请的内存空间，主要有四块，堆内执行内存、堆内存储内存、堆外执行内存和堆外存储内存。

* MemoryManager：对预申请的内存进行分配与回收。目前有[StaticMemoryManager](https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/memory/StaticMemoryManager.scala) 和 [UnifiedMemoryManager](https://github.com/apache/spark/blob/branch-1.6/core/src/main/scala/org/apache/spark/memory/UnifiedMemoryManager.scala) 两种内存管理器，这里Static和Unified是相对执行内存和存储内存的关系来说的。Static即执行内存和存储内存各自空间相对是静态的、固定的；而Unified管理方式下对执行内存和存储内存统一管理，两者可以相互借用。

* ShuffleClient：用于Block上传、下载的接口。

* BlockTransferService：ShuffleClient底层的面向网络RPC的数据传输服务。


对这些类有个整体的认识，再去看源码应该会容易很多。



下面有两张图，比便于对Spark存储体系的整体认识，从网上直接粘过来的，把原文链接放在了参考资料里。



这张图跟《Spark内核设计的艺术》的一样，姑且作为出处：

{% asset_img spark-storage-system.png %}



这张图见参考资料：

{% asset_img spark-store.png %}



## 参考资料：

 [Spark存储体系](https://www.cnblogs.com/cenglinjinran/p/8476199.html)





