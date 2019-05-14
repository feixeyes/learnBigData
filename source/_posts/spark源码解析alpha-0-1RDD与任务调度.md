---
title: spark源码解析alpha-0.1
date: 2019-02-15 10:26:07
tags: spark,原理,源码
---



Spark源码：https://github.com/apache/spark.git 

版本号：alpha-0.1



Spark最核心的概念是RDD（分布式弹性数据集）数据模型，在alpha-0.1版本中实现了RDD数据模型及在其上的任务调度系统，另外还实现了Broadcast和Accumulator工具以及SparkContext接口和spark-shell接口，下面对其分别做介绍。

# RDD与任务调度

​	百度百科对RDD介绍如下：*RDD(Resilient Distributed Datasets)，弹性分布式数据集，是分布式内存的一个抽象概念。RDD提供了一种高度受限的共享内存模型，即RDD是只读的记录分区的集合，只能通过在其他RDD执行确定的转换操作（如map、join和group by）而创建，然而这些限制使得实现容错的开销很低。​*

​        我们可以将RDD看成一个链表，链表的每个节点在上一个节点数据的基础上增加了一些操作而生成。计算最终数据时，只需按照链表上的操作顺序对数据进行计算即可。

data ->f1(data)->f2(f1(data))......->fn(...f1(data)...)

RDD将数据分成多个split，每个split可以单独形成操作链表，及整个RDD成为了一组链表。

不同的操作产生不同的RDD即形成了RDD的类体系。

## RDD的API


RDD的函数可以分为两类，一类是用户接口算子，包括对数据进行操作的transform算子，如map、filter、reduce等和触发任务执行的Action算子如 count、collect、foreach等；另一类是任务执行时需要的函数，如split、iterator等，子类通过复写这些函数来实现不同的子RDD。在alpha-0.1版本中RDD实现了以下函数:


**执行函数（子类重载）**

- def splits: Array[Split]    获取数据分片
- def iterator(split: Split): Iterator[T]   数据分片上的迭代器
- def preferredLocations(split: Split): Seq[String]  数据分片引用的数据地址
- def taskStarted(split: Split, slot: SlaveOffer) 任务是否启动


**Transform算子**

* def map(f: T => U):MappedRDD
* def filter(f: T => Boolean):FilteredRDD
* def aggregateSplit():SplitRDD
* def cache():CachedRDD
* def def sample(withReplacement: Boolean, frac: Double,seed: Int):SampledRDD
* def flatMap(f: T => Traversable[U]):FlatMappedRDD


**Action算子**

* def foreach(f: T => Unit):Unit
* def collect(): Array[T]
* def toArray(): Array[T]
* def reduce(f: (T, T) => T): T
* def take(num: Int): Array[T]
* def first: T
* def count(): Long
* def union(other: RDD[T]):UnionRDD
* def ++(other: RDD[T]):UnionRDD
* def cartesian(other: RDD[U]):CartesianRDD



## RDD类体系



RDD分为两大类，一类根据外部数据生成，是RDD的起点，在alpha-0.1版本中只有HdfsTextFile 和ParallelArray；另一类是在RDD增加操作产生，如 MappedRDD、FilteredRDD等，该类RDD和transform算子相对应。继承体系如下：

{% asset_img  RDDUml.png %}

{% asset_img  SplitUML.png %}

alpha-0.1中实现的RDD有：

* HdfsTextFile(sc: SparkContext, path: String)
* ParallelArray(sc: SparkContext, data: Seq[T], numSlices: Int)
* MappedRDD(prev: RDD[T], f: T => U)
* FilteredRDD(prev: RDD[T], f: T => Boolean)
* FlatMappedRDD(prev: RDD[T], f: T => Traversable[U])
* SplitRDD(prev: RDD[T])
* SampledRDD(prev: RDD[T], withReplacement: Boolean, frac: Double, seed: Int)
* CachedRDD(prev: RDD[T])
* UnionRDD(sc: SparkContext, rdd1: RDD[T], rdd2: RDD[T])
* CartesianRDD(sc: SparkContext, rdd1: RDD[T], rdd2: RDD[U])



## 任务调度

如上所述，Action算子会产生任务，并触发任务的提交。下面我们以foreach为例，追踪任务调度流程。



1. 在action算子（foreach）中，对每个分区(split)生成Task（ForeachTask）实例，并调用sc（SparkContext）中的 runTaskObjects函数来执行任务。

```scala
def foreach(f: T => Unit) {
  val cleanF = sc.clean(f)
  val tasks = splits.map(s => new ForeachTask(this, s, cleanF)).toArray
  sc.runTaskObjects(tasks)
}
```

2、在SparkContext的 runTaskObjects函数中，调用 Scheduler实例的 runTasks函数来执行任务。

```scala
class SparkContext(master: String, frameworkName: String) extends Logging {

  private[spark] def runTaskObjects[T: ClassManifest](tasks: Seq[Task[T]])
      : Array[T] = {
    logInfo("Running " + tasks.length + " tasks in parallel")
    val start = System.nanoTime
    val result = scheduler.runTasks(tasks.toArray)
    logInfo("Tasks finished in " + (System.nanoTime - start) / 1e9 + " s")
    return result
  }
}
```



## Scheduler与Task类体系

{% asset_img  SchedulerUML.png %}

# SparkContext

SparkContext有两大职责，一方面管理着spark运行所需的环境，在alpha-0.1中主要是 任务调度器Scheduler；另一方面向用户提供了编程API。主要函数如下：

**运行环境**

* scheduler: Scheduler   任务调度器 
* def runTasks(tasks: Array[() => T]): Array[T]   任务执行函数（由rdd的action算子调用）

**编程API**

* def textFile(path: String) : HdfsTextFile
* def parallelize(seq: Seq[T], numSlices: Int):ParallelArray[T]
* def parallelize(seq: Seq[T]):ParallelArray[T]
* def accumulator():Accumulator   
* def broadcast(value: T):CentralizedHDFSBroadcast



{% asset_img  SparkContextUML.png %}



# 参考文献

* Spark-alpha-0.1源码解读：https://www.jianshu.com/p/795302f94fa1
* 百度百科：https://baike.baidu.com/item/RDD/5840158

