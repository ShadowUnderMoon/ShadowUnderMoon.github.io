---
title: "Spark基础知识"
author: "爱吃芒果"
description:
date: "2025-05-10T11:51:10+08:00"
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

#### HashPartitioner

```scala
class HashPartitioner(partitions: Int) extends Partitioner {
  require(partitions >= 0, s"Number of partitions ($partitions) cannot be negative.")

  def numPartitions: Int = partitions

  def getPartition(key: Any): Int = key match {
    case null => 0
    case _ => Utils.nonNegativeMod(key.hashCode, numPartitions)
  }

  override def equals(other: Any): Boolean = other match {
    case h: HashPartitioner =>
      h.numPartitions == numPartitions
    case _ =>
      false
  }

  override def hashCode: Int = numPartitions
}
```

`HashPartitioner`使用java的`Object.hashCode`实现了基于hash的分区，java数据的hashCode基于数据的identity而不是他们的内容，所以尝试对`RDD[Array[_]]`或者`RDD[(Array[_], _)]`使用HashPartitioner将产生非预期效果。

#### RangePartitioner

```scala
class RangePartitioner[K : Ordering : ClassTag, V](
    partitions: Int,
    rdd: RDD[_ <: Product2[K, V]],
    private var ascending: Boolean = true,
    val samplePointsPerPartitionHint: Int = 20)
  extends Partitioner {
```

`RangePartitioner`将可排序的几率按范围划分成大致相等的区间，范围是通过对传入的RDD进行采样确定的。分区的实际数量可能和`partitions`参数不一致，比如当采样的记录少于partitions时。

```scala
def getPartition(key: Any): Int = {
  val k = key.asInstanceOf[K]
  var partition = 0
  if (rangeBounds.length <= 128) {
    // 分区个数很少，没有必要走二分查找
    while (partition < rangeBounds.length && ordering.gt(k, rangeBounds(partition))) {
      partition += 1
    }
  } else {
    // Determine which binary search method to use only once.
    partition = binarySearch(rangeBounds, k)
    // binarySearch either returns the match location or -[insertion point]-1
    if (partition < 0) {
      partition = -partition-1
    }
    if (partition > rangeBounds.length) {
      partition = rangeBounds.length
    }
  }
  if (ascending) {
    partition
  } else {
    rangeBounds.length - partition
  }
}
```

partitioner最重要的函数`getPartition`，用于确定某个<K, V> record应该分到哪个partition。

```scala
// 前partitions - 1个分区的上边界
private var rangeBounds: Array[K] = {
  if (partitions <= 1) {
    Array.empty
  } else {
    // 为了使输出分区大致平衡所需要的采样数据量，最大上限为100万
    val sampleSize = math.min(samplePointsPerPartitionHint.toDouble * partitions, 1e6)
    // 假设输出的分区大致平衡，这里超采样一部分
    val sampleSizePerPartition = math.ceil(3.0 * sampleSize / rdd.partitions.length).toInt
    val (numItems, sketched) = RangePartitioner.sketch(rdd.map(_._1), sampleSizePerPartition)
    if (numItems == 0L) {
      Array.empty
    } else {
      // 如果某个分区包含的元素数量远多余平均值，将对该分区重新采样，以确保从该分区中收集到足够的样本
      // fraction表示样本数量和数据总量的比值
      val fraction = math.min(sampleSize / math.max(numItems, 1L), 1.0)
      val candidates = ArrayBuffer.empty[(K, Float)]
      val imbalancedPartitions = mutable.Set.empty[Int]
      sketched.foreach { case (idx, n, sample) =>
        // 按照比例当前分区应该抽样的平均数量高于实际采样数量，认为当前分区需要重采样
        if (fraction * n > sampleSizePerPartition) {
          imbalancedPartitions += idx
        } else {
          // weight是采样概率的倒数，举个例子，假设有两个分区，都采样了30个样本
          // 但a分区大小为300，b分区大小为60，显然a和b分区采样的每个样本应该占的权重不同
          // weight的作用就在于此
          val weight = (n.toDouble / sample.length).toFloat
          for (key <- sample) {
            candidates += ((key, weight))
          }
        }
      }
      if (imbalancedPartitions.nonEmpty) {
        // 仅对需要重新抽样的分区进行操作
        val imbalanced = new PartitionPruningRDD(rdd.map(_._1), imbalancedPartitions.contains)
        val seed = byteswap32(-rdd.id - 1)
        // 使用sample进行抽样, 抽样的比例为fraction
        // 假设第一次抽样，总数为3000，抽样大小为30，平均抽样比例为0.1，所以进行重抽样，这次抽样占比为0.1，也就是300
        val reSampled = imbalanced.sample(withReplacement = false, fraction, seed).collect()
        val weight = (1.0 / fraction).toFloat
        candidates ++= reSampled.map(x => (x, weight))
      }
      // 如果采样的记录少于partitions，则最终的分区数量也会少于partitions
      RangePartitioner.determineBounds(candidates, math.min(partitions, candidates.size))
    }
  }
}
```



```scala
def sketch[K : ClassTag](
    rdd: RDD[K],
    sampleSizePerPartition: Int): (Long, Array[(Int, Long, Array[K])]) = {
  val shift = rdd.id
  // val classTagK = classTag[K] // to avoid serializing the entire partitioner object
  val sketched = rdd.mapPartitionsWithIndex { (idx, iter) =>
    val seed = byteswap32(idx ^ (shift << 16))
    val (sample, n) = SamplingUtils.reservoirSampleAndCount(
      iter, sampleSizePerPartition, seed)
    Iterator((idx, n, sample))
  }.collect()
  val numItems = sketched.map(_._2).sum
  (numItems, sketched)
}
```

`sketch`函数通过蓄水池抽样法从每个分区中抽样指定数量的样本，蓄水池抽样实现了流式的均匀抽样，不需要所有数据都加载在内存中。通过`collect`函数将所有数据收集到driver端，`numItems`是样本总数而不是抽样结果的总数，`sketched`是抽样列表，其中的每个元素包含抽样的partition Id，分区的大小以及抽样样本数组。

```scala
def determineBounds[K : Ordering : ClassTag](
    candidates: ArrayBuffer[(K, Float)],
    partitions: Int): Array[K] = {
  val ordering = implicitly[Ordering[K]]
  // 按照Key进行排序
  val ordered = candidates.sortBy(_._1)
  val numCandidates = ordered.size
  // 计算总权重
  val sumWeights = ordered.map(_._2.toDouble).sum
  // 类似于百分位数，每个区间应该具有的权重
  val step = sumWeights / partitions
  var cumWeight = 0.0
  var target = step
  val bounds = ArrayBuffer.empty[K]
  var i = 0
  var j = 0
  var previousBound = Option.empty[K]
  while ((i < numCandidates) && (j < partitions - 1)) {
    val (key, weight) = ordered(i)
    cumWeight += weight
    if (cumWeight >= target) {
      // 跳过重复的值
      if (previousBound.isEmpty || ordering.gt(key, previousBound.get)) {
        // bounds中的每个元素表示区间的上边界
        bounds += key
        target += step
        j += 1
        previousBound = Some(key)
      }
    }
    i += 1
  }
  bounds.toArray
}

```

`determineBounds`为range partition确定范围边界，返回的结果中的每个元素表示区间的上边界。

### Dependency

```scala
@DeveloperApi
abstract class Dependency[T] extends Serializable {
  def rdd: RDD[T]
}
```

RDD依赖的基础类。

#### NarrowDependency

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

#### PruneDependency

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

`PruneDependency`是窄依赖的一种，子RDD中的分区是父RDD中分区剪枝后的子集，子RDD中的每个分区唯一依赖于父RDD的对应分区。

#### OneToOneDependency

```scala
@DeveloperApi
class OneToOneDependency[T](rdd: RDD[T]) extends NarrowDependency[T](rdd) {
  override def getParents(partitionId: Int): List[Int] = List(partitionId)
}
```

OneToOneDependency表示父rdd和子rdd的分区之间是一一映射关系。

#### RangeDependency

```scala
/**
 * :: DeveloperApi ::
 * Represents a one-to-one dependency between ranges of partitions in the parent and child RDDs.
 * @param rdd the parent RDD
 * @param inStart the start of the range in the parent RDD
 * @param outStart the start of the range in the child RDD
 * @param length the length of the range
 */
@DeveloperApi
class RangeDependency[T](rdd: RDD[T], inStart: Int, outStart: Int, length: Int)
  extends NarrowDependency[T](rdd) {

  override def getParents(partitionId: Int): List[Int] = {
    if (partitionId >= outStart && partitionId < outStart + length) {
      List(partitionId - outStart + inStart)
    } else {
      Nil
    }
  }
}
```

RangeDependency表示父 RDD 和子 RDD 中分区范围之间的一对一依赖关系。依然是一一对应关系，但分区号可能不相同。

#### ShuffleDependency

先通过一个例子来说明ShuffleDependency的用途

```scala
def join[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (V, W))] = self.withScope {
  this.cogroup(other, partitioner).flatMapValues( pair =>
    for (v <- pair._1.iterator; w <- pair._2.iterator) yield (v, w)
  )
}
def cogroup[W](other: RDD[(K, W)], partitioner: Partitioner)
    : RDD[(K, (Iterable[V], Iterable[W]))] = self.withScope {
  if (partitioner.isInstanceOf[HashPartitioner] && keyClass.isArray) {
    throw SparkCoreErrors.hashPartitionerCannotPartitionArrayKeyError()
  }
  val cg = new CoGroupedRDD[K](Seq(self, other), partitioner)
  cg.mapValues { case Array(vs, w1s) =>
    (vs.asInstanceOf[Iterable[V]], w1s.asInstanceOf[Iterable[W]])
  }
}
// CoGroupedRDD
override def getDependencies: Seq[Dependency[_]] = {
  rdds.map { rdd: RDD[_] =>
    if (rdd.partitioner == Some(part)) {
      logDebug("Adding one-to-one dependency with " + rdd)
      new OneToOneDependency(rdd)
    } else {
      logDebug("Adding shuffle dependency with " + rdd)
      new ShuffleDependency[K, Any, CoGroupCombiner](
        rdd.asInstanceOf[RDD[_ <: Product2[K, _]]], part, serializer)
    }
  }
}
```

假设有一个join操作，指定了结果RDD的Partitioner，内部调用了cogroup生成了CoGroupedRDD，并且将依赖的RDD都作为参数传入，如果依赖的RDD和指定的Partitioner相同，则是窄依赖，否则是宽依赖，生成ShufflDependency。

```scala
@DeveloperApi
class ShuffleDependency[K: ClassTag, V: ClassTag, C: ClassTag](
    @transient private val _rdd: RDD[_ <: Product2[K, V]],
    val partitioner: Partitioner,
    val serializer: Serializer = SparkEnv.get.serializer,
    val keyOrdering: Option[Ordering[K]] = None,
    val aggregator: Option[Aggregator[K, V, C]] = None,
    val mapSideCombine: Boolean = false,
    val shuffleWriterProcessor: ShuffleWriteProcessor = new ShuffleWriteProcessor)
  extends Dependency[Product2[K, V]] with Logging {

  if (mapSideCombine) {
    require(aggregator.isDefined, "Map-side combine without Aggregator specified!")
  }
  override def rdd: RDD[Product2[K, V]] = _rdd.asInstanceOf[RDD[Product2[K, V]]]

  private[spark] val keyClassName: String = reflect.classTag[K].runtimeClass.getName
  private[spark] val valueClassName: String = reflect.classTag[V].runtimeClass.getName
  // Note: It's possible that the combiner class tag is null, if the combineByKey
  // methods in PairRDDFunctions are used instead of combineByKeyWithClassTag.
  private[spark] val combinerClassName: Option[String] =
    Option(reflect.classTag[C]).map(_.runtimeClass.getName)

  val shuffleId: Int = _rdd.context.newShuffleId()

  val shuffleHandle: ShuffleHandle = _rdd.context.env.shuffleManager.registerShuffle(
    shuffleId, this)

  private[this] val numPartitions = rdd.partitions.length
```

ShuffleDependency的构造函数包括：

- RDD 需要shuffle操作的RDD
- Partitioner shuffle的分区器
- Aggregator map端、reduce端的聚合函数
- mapSideCombine 是否在map端进行聚合

ShuffleDependency构造时，会从SparkContext中获取应用唯一的Shuffle ID作为表示，ShuffleDependency将自己注册到ShuffleManager，并返回ShuffleHandle

