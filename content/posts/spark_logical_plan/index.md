---
title: "Spark逻辑处理流程"
author: "爱吃芒果"
description:
date: "2025-05-11T15:20:27+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
    - "Spark"
categories:
    - "Spark"
---

Spark应用程序需要先转化为逻辑处理流程，逻辑处理流程主要包括：

- RDD数据模型
- 数据操作
- 数据依赖关系

数据操作分为两种，`transformation`操作并不会触发job的实际执行，`action`操作创建job并立即执行。类似于java中的stream，采用懒加载的方式。

## RDD数据模型

RDD （Resilient Distributed DataSet)是spark对计算过程中输入输出数据以及中间数据的抽象，表示不可变、分区的集合数据，可以被并行处理。

```scala
abstract class RDD[T: ClassTag](
    @transient private var _sc: SparkContext,
    @transient private var deps: Seq[Dependency[_]]
  ) extends Serializable with Logging {
```

`RDD`类包含一些基础操作，比如map、filter和persist，另外

- `PairRDDFunctions`包含专门处理键值对RDD的操作，比如`groupByKey`和`join`

- `DoubleRDDFunctions`包含数据为Double类型的RDD的操作

- `SequenceFileRDDFunction`包含可以被保存为SequenceFiels的RDD的操作
- `OrderedRDDFunctions`键值对RDD，key通过隐式转换后支持排序

RDD主要有5种属性：

- 分区列表
- 计算每个分区的函数
- 对其他RDD的依赖组成的依赖链表
- 可选，键值对RDD进行分区的Partitioner 比如某个RDD是hash分区的
- 可选，计算每个分区的本地化偏好列表，比如依据hdfs文件的block位置给定偏好，降低网络传输开销

### RDD常用属性

- `SparkContext` RDD所属的上下文
- `Seq[Dependency[_]]` 当前RDD依赖的RDD列表
- `Option[Partitioner]` partitioner，可以被子类重写，表示RDD是如何分区的
- `Array[Partition]` RDD拥有的所有分区

### Partition

```scala
/**
 * An identifier for a partition in an RDD.
 */
trait Partition extends Serializable {
  /**
   * Get the partition's index within its parent RDD
   */
  def index: Int

  // A better default implementation of HashCode
  override def hashCode(): Int = index

  override def equals(other: Any): Boolean = super.equals(other)
}

```

`Partition`表示RDD中的一个分区

```scala
private[spark] class PartitionPruningRDDPartition(idx: Int, val parentSplit: Partition)
  extends Partition {
  override val index = idx
}
```

`PartitionPruningRDDPartition`表示父RDD被剪枝后生成的子RDD中的分区。`idx`表示子RDD中分区的partition Id，`parentsplit`表示对应的父RDD中的分区。

### Partitioner

```scala
abstract class Partitioner extends Serializable {
  def numPartitions: Int
  def getPartition(key: Any): Int
}
```

`Partitioner`定义了键值对RDD中的元素如何通过key进行分区，映射每个key到一个partition ID，从0到 `numPartitions - 1`。注意partitioner必须是确定性的，给定相同的partition key必须返回相同的分区。

### Dependency

```scala
@DeveloperApi
abstract class Dependency[T] extends Serializable {
  def rdd: RDD[T]
}
```

RDD依赖的基础类。

```scala
@DeveloperApi
abstract class NarrowDependency[T](_rdd: RDD[T]) extends Dependency[T] {
  /**
   * Get the parent partitions for a child partition.
   * @param partitionId a partition of the child RDD
   * @return the partitions of the parent RDD that the child partition depends upon
   */
  def getParents(partitionId: Int): Seq[Int]

  override def rdd: RDD[T] = _rdd
}
```

窄依赖`NarrowDependency`，子RDD的每个分区依赖于父RDD的一小部分分区，窄依赖允许流水线执行，`getParenets`返回子RDD分区依赖的所有父RDD分区。

```scala
private[spark] class PruneDependency[T](rdd: RDD[T], partitionFilterFunc: Int => Boolean)
  extends NarrowDependency[T](rdd) {

  @transient
  val partitions: Array[Partition] = rdd.partitions
    .filter(s => partitionFilterFunc(s.index)).zipWithIndex
    // idx是子RDD的partition Id，从0开始
    // split是对应的父RDD中的分区
    .map { case(split, idx) => new PartitionPruningRDDPartition(idx, split) : Partition }

  override def getParents(partitionId: Int): List[Int] = {
    List(partitions(partitionId).asInstanceOf[PartitionPruningRDDPartition].parentSplit.index)
  }
}
```

`PruneDependency`是窄依赖的一种，子RDD中的分区是父RDD中分区剪枝后的子集，子RDD中的分区唯一依赖于父RDD的中一个分区。

## 常用transformation数据操作

### map操作

```scala

// scalastyle:off println
package org.apache.spark.examples

import scala.collection.compat.immutable.ArraySeq

import org.apache.spark.SparkContext
import org.apache.spark.sql.SparkSession

object MapDemo {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder()
      .appName("MapDemo")
      .master("local")
      .getOrCreate()
    val sc = spark.sparkContext.asInstanceOf[SparkContext]
    val array = Array[(Int, Char)](
      (1, 'a'), (2, 'b'), (3, 'c'), (4, 'd'), (2, 'e'), (3, 'f'), (2, 'g'), (1, 'h'))
    val inputRDD = sc.parallelize(
      ArraySeq.unsafeWrapArray(array)
      , 3)
    val resultRDD = inputRDD.map(r => s"${r._1}_${r._2}")
    resultRDD.foreach(println)
    spark.stop()
  }
}
```

这里给出了一个简单的例子，通过map函数将key和value拼接起来。

```scala
def parallelize[T: ClassTag](
    seq: Seq[T],
    numSlices: Int = defaultParallelism): RDD[T] = withScope {
  assertNotStopped()
  new ParallelCollectionRDD[T](this, seq, numSlices, Map[Int, Seq[String]]())
}
private[spark] class ParallelCollectionRDD[T: ClassTag](
    sc: SparkContext,
    @transient private val data: Seq[T],
    numSlices: Int,
    locationPrefs: Map[Int, Seq[String]])
    extends RDD[T](sc, Nil) {
```

`parallelize`将一个局地的scala集合分布式化成RDD，但实际上仅仅是构建`ParallelCollectionRDD`而已，没有依赖于其他RDD，所以传入的为`Nil`。

```scala
def map[U: ClassTag](f: T => U): RDD[U] = withScope {
  val cleanF = sc.clean(f)
  new MapPartitionsRDD[U, T](this, (_, _, iter) => iter.map(cleanF))
}
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false,
    isFromBarrier: Boolean = false,
    isOrderSensitive: Boolean = false)
  extends RDD[U](prev) {
```

`map`函数对输出的RDD的每条记录应用目标函数，获得新的`MapPartitionRDD`，依赖于之前的RDD prev。

### mapValues

```scala
val resultRDD = inputRDD.mapValues(x => s"${x} + 1")
```

对之前的map例子稍作调整，调用`mapValues`而不是`map`函数，不改变key，只对value进行转换。

```scala
def mapValues[U](f: V => U): RDD[(K, U)] = self.withScope {
  val cleanF = self.context.clean(f)
  new MapPartitionsRDD[(K, U), (K, V)](self,
    (context, pid, iter) => iter.map { case (k, v) => (k, cleanF(v)) },
    preservesPartitioning = true)
}
```

实际调用了`PairRDDFunctions`中的`mapValue`方法，最终生成的依然是`MapPartitionsRDD`，但有两点不同，一是目标函数只对value进行转换，二是`preservePartitioning`为true。这里很好理解，`map`函数会对键值对进行操作，partitioner可能会失效，而`mapVlaues`只对value进行操作，不影响key，所以partitioner依然保持。

### filter

```scala
val resultRDD = inputRDD.filter(r => r._1 % 2 == 0)
```

`filter`对输入RDD中的每条记录进行func操作，如果结果为true，则保留这条记录，所有保留的记录形成新的RDD

```scala
def filter(f: T => Boolean): RDD[T] = withScope {
  val cleanF = sc.clean(f)
  new MapPartitionsRDD[T, T](
    this,
    (_, _, iter) => iter.filter(cleanF),
    preservesPartitioning = true)
}
```

`filter`操作后生成的RDD依然是`MapPartitionsRDD`，没有修改键值对，`preservesPartitioning`为true。

### filterByRange

```scala
val resultRDD = inputRDD.filterByRange(2, 4)
```

`filterByRange`对输入RDD中的数据进行过滤，只保留[lower, upper]之间的记录。

```scala
def filterByRange(lower: K, upper: K): RDD[P] = self.withScope {

  def inRange(k: K): Boolean = ordering.gteq(k, lower) && ordering.lteq(k, upper)

  val rddToFilter: RDD[P] = self.partitioner match {
    case Some(rp: RangePartitioner[_, _]) =>
    // getPartition获取分区号，partitionIndices表示可能包含目标记录的分区id Range
      val partitionIndices = (rp.getPartition(lower), rp.getPartition(upper)) match {
        case (l, u) => Math.min(l, u) to Math.max(l, u)
      }
      PartitionPruningRDD.create(self, partitionIndices.contains)
    case _ =>
      self
  }
  rddToFilter.filter { case (k, v) => inRange(k) }
}
```

`filterByRange`操作属于`OrderedRDDFunctions`，如果RDD通过`RangePartitioner`分区，这个操作可以执行的更加高效，仅需要对可能包含匹配元素的分区进行扫描，否则需要对所有分区应用filter。

```scala
@DeveloperApi
object PartitionPruningRDD {
  def create[T](rdd: RDD[T], partitionFilterFunc: Int => Boolean): PartitionPruningRDD[T] = {
    new PartitionPruningRDD[T](rdd, partitionFilterFunc)(rdd.elementClassTag)
  }
}
@DeveloperApi
class PartitionPruningRDD[T: ClassTag](
    prev: RDD[T],
    partitionFilterFunc: Int => Boolean)
  extends RDD[T](prev.context, List(new PruneDependency(prev, partitionFilterFunc))) {
```

`PartitionPruningRDD`用于RDD分区的剪枝，避免对所有分区进行操作。

### flatMap

