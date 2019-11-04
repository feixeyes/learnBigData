---
title: Spark0.1源码学习之-编程模型和任务调度 
tags:
  - spark
  - 源码
date: 2019-11-04 17:59:00
categories: 技术原理
---

## 从大数据的Hello World说起

Spark 是一个大数据（分布式）计算框架，我们从大数据计算的hello world （word count）来看一下spark 的基本思想。

**Word Count问题，即输入一份文档文件，计算文档中各单词的个数。**

这是一个简单的计数问题，不考虑数据量，一种简单的解决方法是利用一个Map结构，各单词作key，各单词出现的次数作为value，逐个处理各单词，对各单词进行计数。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8iwbamz5wj30y60p079f.jpg)

伪代码基本是：

```shell
resMap = Map[String,Int]

for line in file:
  for word in line.split(' '):
    resMap[word]++
```

当数据量增大之后（比如上G），逐个单词计数会很低效，这时候可以考虑使用多进程将输入数据拆分成多个小的文件，统计过程划分为两个阶段。 1）统计各个文件中各单词的个数；2）汇总各文件的结果。过程如下图所示：

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8iif3m44rj319i0u0tc6.jpg)

两个阶段的程序，有不同的职责：

1. 第一个阶段的程序，负责读取数据，对数据进行解析（上例中文档解析出单词），产生中间数据；
2. 第二阶段的程序，负责汇总最终结果。 

我们将上述两个阶段分别称为map（和上述Map数据结构不一个概念） 和 reduce. 

## Hadoop

上述多线程版本的程序，可以拆解成“单词计数相关的逻辑”以及“解决分治/汇总类问题的通用框架”，比如换成求“全国当前人口的平均年龄”，在单词计数中用到的解析单词之类的逻辑用不到了，但多个map分别读取数据进行计算产生中间结果；将中间结果，发送到reduce，这些通用功能是可以复用的。

当数据量进一步增大，上述计算可以扩展到分布式环境中,即从多进程变成多台机器并行处理。 

第一个阶段，map的过程由不同的机器完成，将中间结果发送到一台reduce所在的机器。

第二各阶段，reduce的过程和单机类似，事实上如果最终结果数据量很大，用单一的一个reduce将会成为性能瓶颈。 比如上述Word Count问题，我们想象结果是一个超级大的单词表，虽然单词表很大，但每个单词其实是唯一的。可以有多个reduce，每个reduce只处理特定几个单词的计数。

这样又引入一个新的问题，每个map处理的中间结果，要以特定的规则来分发到reduce上，使得相同的单词的中间结果可以分发到同一个reduce上。这个过程称为 **shuffle**。

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8iw39nhvfj31140ikdkr.jpg)



Hadoop的本质就是将上述，Map/Reduce模型以及其依赖的shuffle过程等形成框架。

在大规模分布式环境中，除了编程模型，还需要花费大量的工作在资源分配（哪台机器处理map哪台处理reduce）和任务调度上（还有多少map任务没有结束，reduce任务是否可以开始启动了）。

Hadoop v2将Map/Reduce模型（含任务调度）和资源管理拆分成了两个独立部分，独立出来的资源管理模块就是YARN。

*说明：Hadoop相关知识不是本位重点，就像从单机程序讲都只是为了更好的理解spark，所以不会对hadoop做过多的讨论*

## Spark

Hadoop将解决问题的模式限定在Map和Reduce两个阶段，Spark提供了更灵活的问题模式的支持。

首先，spark对数据操作做了更多形式的抽象，比如对应于Map阶段细分出flapMap用来表示，将一条转成多条数据。 将reduce阶段细分出 groupby 、reduceByKey等阶段。

其次，M/R的两阶段在Spark中扩展成了，对数据一系列类似map、flatMap等转换的序列。

在Spark中对数据的抽象是**RDD（Resilient Distributed Datasets)**，百度百科对RDD介绍如下：

>  RDD(Resilient Distributed Datasets)，弹性分布式数据集，是分布式内存的一个抽象概念。RDD提供了一种高度受限的共享内存模型，即RDD是只读的记录分区的集合，只能通过在其他RDD执行确定的转换操作（如map、join和group by）而创建，然而这些限制使得实现容错的开销很低。

即我们可以将RDD看成一个链表，链表的每个节点在上一个节点数据的基础上增加了一些操作而生成。计算最终数据时，只需按照链表上的操作顺序对数据进行计算即可，

**data ->f1(data)->f2(f1(data))……->fn(…f1(data)…)**

对于word count 问题，spark代码流程如下：

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8luk1j704j30sd0nvq4r.jpg)

## RDD的核心思想

1. 对一个RDD进行操作后形成新的RDD； 

2. 每个RDD可以定义为 上游数据集及其上的一个操作 **RDD**{parent,function} 

   ps: 操作分为两个部分，一个是抽象出来通用的操作类型，如对数据逐个操作、过滤、抽样； 另一部分是更为灵活的逻辑，如 对数据逐个操作中的具体操作逻辑。 前一部分，Spark定义成通用的API，后一部分，用户以函数参数的形式传入。

3. RDD的起源为使用API由外部数据（文件等）创建得到。   

4. RDD的转换由多个称为Transform操作的API组成  ps:及各种操作对应的 RDD子类

5. 考虑并行处理，每个数据集可以表示成多个 **split**的组合，及多个子数据集，RDD扩展为 一组数据集 及 其上的一个操作 形成新的一组数据集 

6. 由称为Action的多个API生成最终数据。

以 alpha-0.1版本 HdfsTest 为例，代码如下：

```scala
import spark._

object HdfsTest {
  def main(args: Array[String]) {
    val sc = new SparkContext(args(0), "HdfsTest")
    val file = sc.textFile(args(1))
    val mapped = file.map(s => s.length).cache()
    for (iter <- 1 to 10) {
      val start = System.currentTimeMillis()
      for (x <- mapped) { x + 2 }
      //  println("Processing: " + x)
      val end = System.currentTimeMillis()
      println("Iteration " + iter + " took " + (end-start) + " ms")
    }
  }
}
```

上述代码中，变量file 和 mapped都是 RDD实例，其中，file由SparkContext的API直接从外部文件穿件得到；mapped 根据RDD的转换函数生成。代码中的其他的内容这里不比关注。 

## RDD类定义

在alpha-0.1版本中，RDD定义如下：

![](https://tva1.sinaimg.cn/large/006y8mN6ly1g8l7vs4snlj30lo0o877w.jpg)



**RDD的函数可以分为两类，一类是用户接口算子，包括对数据进行操作的transform算子，如map、filter、reduce等和触发任务执行的Action算子如 count、collect、foreach等；另一类是任务执行时需要的函数，如split、iterator等，子类通过复写这些函数来实现不同的子RDD。**

在alpha-0.1版本中RDD实现了以下函数:

**执行函数（子类重载）**

- def splits: Array[Split] 获取数据分片

- def iterator(split: Split): Iterator[T] 数据分片上的迭代器 ps:每个分片也是一个数据集，需要提供迭代器来遍历。

- def preferredLocations(split: Split): Seq[String] 数据分片引用的数据地址

在RDD类中，上述三个函数只有定义并没有实现，在各子类中具体实现，以**HdfsTextFile**类为例，函数实现如下：

```scala
@transient val splits_ =
  inputFormat.getSplits(conf, sc.scheduler.numCores).map(new HdfsSplit(_)).toArray

override def splits = splits_.asInstanceOf[Array[Split]]

override def iterator(split_in: Split) = new Iterator[String] {
  val split = split_in.asInstanceOf[HdfsSplit]
  var reader: RecordReader[LongWritable, Text] = null
  ConfigureLock.synchronized {
    val conf = new JobConf()
    conf.set("io.file.buffer.size",
        System.getProperty("spark.buffer.size", "65536"))
    val tif = new TextInputFormat()
    tif.configure(conf) 
    reader = tif.getRecordReader(split.inputSplit.value, conf, Reporter.NULL)
  }
  val lineNum = new LongWritable()
  val text = new Text()
  var gotNext = false
  var finished = false

  override def hasNext: Boolean = {
    if (!gotNext) {
      try {
        finished = !reader.next(lineNum, text)
      } catch {
        case eofe: java.io.EOFException =>
          finished = true
      }
      gotNext = true
    }
    !finished
  }

  override def next: String = {
    if (!gotNext)
      finished = !reader.next(lineNum, text)
    if (finished)
      throw new java.util.NoSuchElementException("end of stream")
    gotNext = false
    text.toString
  }
}

override def preferredLocations(split: Split) = {
  // TODO: Filtering out "localhost" in case of file:// URLs
  split.asInstanceOf[HdfsSplit].inputSplit.value.getLocations().filter(_ != "localhost")
}
```

**Transform算子**

- def map(f: T => U):MappedRDD

- def flatMap(f: T => Traversable[U]):FlatMappedRDD

- def filter(f: T => Boolean):FilteredRDD

- def aggregateSplit():SplitRDD

- def cache():CachedRDD

- def def sample(withReplacement: Boolean, frac: Double,seed: Int):SampledRDD

- def union(other: RDD[T]):UnionRDD

- def ++(other: RDD[T]):UnionRDD

- def cartesian(other: RDD[U]):CartesianRDD

每个Transform算子会产生一个与其相对类型的RDD，如map算子的实现如下：

```scala
def map[U: ClassManifest](f: T => U) = new MappedRDD(this, sc.clean(f))
```

**Action算子**

* def foreach(f: T => Unit):Unit

- def collect(): Array[T]

- def toArray(): Array[T]

- def reduce(f: (T, T) => T): T

- def take(num: Int): Array[T]

- def first: T

- def count(): Long

每个Action算子会产生任务调度，如collect算子的实现如下：

```scala
def collect(): Array[T] = {
  val tasks = splits.map(s => new CollectTask(this, s))
  val results = sc.runTaskObjects(tasks)
  Array.concat(results: _*)
}
```

## RDD类体系

RDD分为两大类，一类根据外部数据生成，是RDD的起点，在alpha-0.1版本中只有HdfsTextFile 和ParallelArray；另一类是在RDD增加操作产生，如 MappedRDD、FilteredRDD等，该类RDD和transform算子相对应。继承体系如下：

![img](https://tva1.sinaimg.cn/large/006y8mN6ly1g8ly0b6kb7j30wt0bndgk.jpg)



![img](https://tva1.sinaimg.cn/large/006y8mN6ly1g8ly09rme9j30eq09mjrl.jpg)

alpha-0.1中实现的RDD有：

- HdfsTextFile(sc: SparkContext, path: String)
- ParallelArray(sc: SparkContext, data: Seq[T], numSlices: Int)
- MappedRDD(prev: RDD[T], f: T => U) 对应map操作
- FilteredRDD(prev: RDD[T], f: T => Boolean)  对应filter操作
- FlatMappedRDD(prev: RDD[T], f: T => Traversable[U]) 对应flatmap操作
- SplitRDD(prev: RDD[T])
- SampledRDD(prev: RDD[T], withReplacement: Boolean, frac: Double, seed: Int)   对应sample操作
- CachedRDD(prev: RDD[T])  对应cache操作
- UnionRDD(sc: SparkContext, rdd1: RDD[T], rdd2: RDD[T]) 把split 合并
- CartesianRDD(sc: SparkContext, rdd1: RDD[T], rdd2: RDD[U])

## 任务调度

如上所述，Action算子会产生任务，并触发任务的提交。下面我们以foreach为例，追踪任务调度流程。

第一步，在action算子（foreach）中，对每个分区(split)生成Task（ForeachTask）实例，并调用sc（SparkContext）中的 runTaskObjects函数来执行任务。

```scala
def foreach(f: T => Unit) {
  val cleanF = sc.clean(f)
  val tasks = splits.map(s => new ForeachTask(this, s, cleanF)).toArray
  sc.runTaskObjects(tasks)
}
```

第二步，在SparkContext的 runTaskObjects函数中，调用 Scheduler实例的 runTasks函数来执行任务。

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

![img](https://tva1.sinaimg.cn/large/006y8mN6ly1g8ly05fsgpj30iz09i3yu.jpg)

# SparkContext

SparkContext有两大职责，一方面管理着spark运行所需的环境，在alpha-0.1中主要是 任务调度器Scheduler；另一方面向用户提供了编程API。主要函数如下：

**运行环境**

- scheduler: Scheduler 任务调度器
- def runTasks(tasks: Array[() => T]): Array[T] 任务执行函数（由rdd的action算子调用）

**编程API**

- def textFile(path: String) : HdfsTextFile
- def parallelize(seq: Seq[T], numSlices: Int):ParallelArray[T]
- def parallelize(seq: Seq[T]):ParallelArray[T]
- def accumulator():Accumulator
- def broadcast(value: T):CentralizedHDFSBroadcast

![img](https://tva1.sinaimg.cn/large/006y8mN6ly1g8ly010hsmj30bg0bnmyl.jpg)



## 总结

在 alpha-0.1版本中，实现了基本分布式数据模型RDD的类体系和任务调度模型，但这个版本还比较简单，并没有涉及复杂的操作，比如并没有实现涉及到shuffle过程的操作。但该版本对于理解spark的基本思想还是有很大的帮助。

下一篇文章，我们来调试一下spark0.1版本中的编程模型和任务调度系统。

# 参考文献

- Spark-alpha-0.1源码解读：https://www.jianshu.com/p/795302f94fa1
- 百度百科：https://baike.baidu.com/item/RDD/5840158
- [spark源码解析alpha-0.1]([https://blog.guopengfei.top/2019/02/15/spark%E6%BA%90%E7%A0%81%E8%A7%A3%E6%9E%90alpha-0-1RDD%E4%B8%8E%E4%BB%BB%E5%8A%A1%E8%B0%83%E5%BA%A6/#more](https://blog.guopengfei.top/2019/02/15/spark源码解析alpha-0-1RDD与任务调度/#more))





