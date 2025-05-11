# Spark逻辑处理流程


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

```scala
val array = Array[String](
  "how do you do", "are you ok", "thanks", "bye bye", "I'm ok"
)
val inputRDD = sc.parallelize(
  ArraySeq.unsafeWrapArray(array)
  , 3)
val resultRDD = inputRDD.flatMap(x => x.split(" "))
```

对输入RDD中每个元素（如List）执行func操作，得到新元素，然后将所有新元素组合得到新RDD。例如输入RDD中某个分区包含两个元素List(1, 2)和List(3, 4)，func是对List中的每个元素加1，那么最后得到的新RDD中该分区的元素为(2, 3, 4, 5)，实例代码会做分词操作，组成新的RDD。

```scala
def flatMap[U: ClassTag](f: T => IterableOnce[U]): RDD[U] = withScope {
  val cleanF = sc.clean(f)
  new MapPartitionsRDD[U, T](this, (_, _, iter) => iter.flatMap(cleanF))
}
```

`flatMap`最终返回的也是`MapPartitionsRDD`，对每个分区的iter调用`flatMap`函数

### flatMapValues

```scala
val array = Array[(Int, String)](
  (1, "how do you do"), (2, "are you ok"), (4, "thanks"), (5, "bye bye"),
  (2, "I'm ok")
)
val inputRDD = sc.parallelize(
  ArraySeq.unsafeWrapArray(array)
  , 3)
val resultRDD = inputRDD.flatMapValues(x => x.split(" "))
```

与flatMap类似，但只对RDD中<K, V> record中Value进行操作。

```scala
def flatMapValues[U](f: V => IterableOnce[U]): RDD[(K, U)] = self.withScope {
  val cleanF = self.context.clean(f)
  new MapPartitionsRDD[(K, U), (K, V)](self,
    (context, pid, iter) => iter.flatMap { case (k, v) =>
      cleanF(v).iterator.map(x => (k, x))
    },
    preservesPartitioning = true)
}
```

`flatMapValues`同样属于`PairRDDFunction`类，通过flatMap操作<K, V> record中的value，但不改变key，flatMapValues保持原先RDD的分区特性。

### sample

```scala
val array = Array[(Int, Char)](
  (1, 'a'), (2, 'b'), (3, 'c'), (4, 'd'), (2, 'e'), (3, 'f'), (2, 'g'), (1, 'h'))
val inputRDD = sc.parallelize(
  ArraySeq.unsafeWrapArray(array)
  , 3)
val sampleRDD = inputRDD.sample(false, 0.5)
```

对RDD中的数据进行抽样。

```scala
def sample(
    withReplacement: Boolean,
    fraction: Double,
    seed: Long = Utils.random.nextLong): RDD[T] = {
  require(fraction >= 0,
    s"Fraction must be nonnegative, but got ${fraction}")

  withScope {
    if (withReplacement) {
      new PartitionwiseSampledRDD[T, T](this, new PoissonSampler[T](fraction), true, seed)
    } else {
      new PartitionwiseSampledRDD[T, T](this, new BernoulliSampler[T](fraction), true, seed)
    }
  }
}
```

`sample`函数用于对RDD抽样，`withRepalcement`表示抽样是否有放回，`fraction`表示抽样比例，在有放回抽样中表示每个元素期望被选中的次数，fraction >= 0，使用泊松抽样，在无放回抽样中表示每个元素被选中的概率，fraction 在0到1之间，使用伯努利抽样。不能保证抽样数量精确的等于给定RDD记录总数 * fraction。

```scala
private[spark] class PartitionwiseSampledRDD[T: ClassTag, U: ClassTag](
    prev: RDD[T],
    sampler: RandomSampler[T, U],
    preservesPartitioning: Boolean,
    @transient private val seed: Long = Utils.random.nextLong)
  extends RDD[U](prev) {
```

`PartitionwiseSampledRDD`表示从父 RDD 的各个分区分别进行抽样而生成的 RDD，对于父RDD的每个分区，一个RandomSampler实例被用于获得这个分区中记录的随机抽样结果。

### sampleByKey

```scala
val array = Array[(Int, Char)](
  (1, 'a'), (2, 'b'), (1, 'c'), (2, 'd'), (2, 'e'), (1, 'f'), (2, 'g'), (1, 'h'))
val inputRDD = sc.parallelize(
  ArraySeq.unsafeWrapArray(array)
  , 3)
val map = Map((1 -> 0.8), (2 -> 0.5))
val sampleRDD = inputRDD.sampleByKey(false, map)
```

对输入RDD中的数据进行抽样，为每个Key设置抽样比例。

```scala
def sampleByKey(withReplacement: Boolean,
    fractions: Map[K, Double],
    seed: Long = Utils.random.nextLong): RDD[(K, V)] = self.withScope {

  require(fractions.values.forall(v => v >= 0.0), "Negative sampling rates.")

  val samplingFunc = if (withReplacement) {
    StratifiedSamplingUtils.getPoissonSamplingFunction(self, fractions, false, seed)
  } else {
    StratifiedSamplingUtils.getBernoulliSamplingFunction(self, fractions, false, seed)
  }
  self.mapPartitionsWithIndex(samplingFunc, preservesPartitioning = true, isOrderSensitive = true)
}
```

使用简单随机抽样并仅遍历一次RDD，根据fractions为不同的键指定不同的采样率，从该RDD创建一个样本，所生成的样本大小大致等于对所有剪枝执行math.ceil(numItems * samplingRate)的总和。

`mapPartitionsWithIndex`通过对RDD的每个分区应用目标函数得到新的RDD，同时跟踪原来分区的index。应该是将index传入，来保证不同分区获得不同的随机性（只是猜测）。

### mapPartitions

```scala
def mapPartitions[U: ClassTag](
    f: Iterator[T] => Iterator[U],
    preservesPartitioning: Boolean = false): RDD[U] = withScope {
  val cleanedF = sc.clean(f)
  new MapPartitionsRDD(
    this,
    (_: TaskContext, _: Int, iter: Iterator[T]) => cleanedF(iter),
    preservesPartitioning)
}
```

`mapPartitions`对输入RDD中的每个分区进行func处理，输出新的一组数据，相较于`map`操作，具有更大的自由度，可以以任意方式处理整个分区的数据，而不是只能逐条遍历分区中的记录。

### mapPartitionsWithIndex

```scala
private[spark] def mapPartitionsWithIndex[U: ClassTag](
    f: (Int, Iterator[T]) => Iterator[U],
    preservesPartitioning: Boolean,
    isOrderSensitive: Boolean): RDD[U] = withScope {
  val cleanedF = sc.clean(f)
  new MapPartitionsRDD(
    this,
    (_: TaskContext, index: Int, iter: Iterator[T]) => cleanedF(index, iter),
    preservesPartitioning,
    isOrderSensitive = isOrderSensitive)
}
```

`mapPartitionsWithIndex`和`mapPartitions`语义类似，只是多传入了partition Id，利用这个id，可以实现对不同分区分别处理，比如之前`sampleByKey`操作就利用了partition Id。

```scala
val list = List(1, 2, 3, 4, 5, 6, 7, 8, 9)
val inputRDD = sc.parallelize(
  list,
  3
)
val resultRDD = inputRDD.mapPartitionsWithIndex((pid, iter) => {
  iter.map(Value => s"Pid: ${pid}, Value: ${Value}")
})

resultRDD.foreach(println)
```

比如可以利用这个函数打印RDD的内容，了解每个分区中有哪些数据。

### partitionBy

```scala
val array = Array[(Int, Char)](
  (1, 'a'), (2, 'b'), (1, 'c'), (2, 'd'), (2, 'e'), (1, 'f'), (2, 'g'), (1, 'h'))
val inputRDD = sc.parallelize(
  ArraySeq.unsafeWrapArray(array)
  , 3)

val resultRDD = inputRDD.partitionBy(new HashPartitioner(2))
val resultRDD2 = inputRDD.partitionBy(new RangePartitioner(2, inputRDD))
```

`partitionBy`使用新的partitioner对RDD进行分区，要求RDD是<K, V>类型。

```scala
def partitionBy(partitioner: Partitioner): RDD[(K, V)] = self.withScope {
  if (keyClass.isArray && partitioner.isInstanceOf[HashPartitioner]) {
    throw SparkCoreErrors.hashPartitionerCannotPartitionArrayKeyError()
  }
  if (self.partitioner == Some(partitioner)) {
    self
  } else {
    new ShuffledRDD[K, V, V](self, partitioner)
  }
}
```

`partitionBy`如果提供的partitioner和RDD原先的partitioner相同，则返回原来的RDD，否则返回`ShuffledRDD`。

```scala
@DeveloperApi
class ShuffledRDD[K: ClassTag, V: ClassTag, C: ClassTag](
    @transient var prev: RDD[_ <: Product2[K, V]],
    part: Partitioner)
  extends RDD[(K, C)](prev.context, Nil) {
```

`ShuffledRDD`表示shuffle后的RDD，即重新分区后的数据。


