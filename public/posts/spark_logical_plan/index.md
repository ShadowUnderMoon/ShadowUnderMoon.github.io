# Spark逻辑处理流程


Spark应用程序需要先转化为逻辑处理流程，逻辑处理流程主要包括：

- RDD数据模型
- 数据操作
- 数据依赖关系

数据操作分为两种，`transformation`操作并不会触发job的实际执行，`action`操作创建job并立即执行。类似于java中的stream，采用懒加载的方式。





## 常用transformation数据操作

### map

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

### groupByKey

```scala
def groupByKey(numPartitions: Int): RDD[(K, Iterable[V])] = self.withScope {
  groupByKey(new HashPartitioner(numPartitions))
}
def groupByKey(partitioner: Partitioner): RDD[(K, Iterable[V])] = self.withScope {
  // groupByKey shouldn't use map side combine because map side combine does not
  // reduce the amount of data shuffled and requires all map side data be inserted
  // into a hash table, leading to more objects in the old gen.
  val createCombiner = (v: V) => CompactBuffer(v)
  val mergeValue = (buf: CompactBuffer[V], v: V) => buf += v
  val mergeCombiners = (c1: CompactBuffer[V], c2: CompactBuffer[V]) => c1 ++= c2
  val bufs = combineByKeyWithClassTag[CompactBuffer[V]](
    createCombiner, mergeValue, mergeCombiners, partitioner, mapSideCombine = false)
  bufs.asInstanceOf[RDD[(K, Iterable[V])]]
}
```

将RDD1中的<K, V> record按照key聚合在一起，形成`K, List<V>`，numPartitions表示生成的rdd2的分区个数。`groupByKey`的行为和父RDD的partitioner有关，如果父RDD和生成的子RDD的partitioiner相同，则不需要shuffle，否则需要进行shuffle。假如在这里指定分区数为`3`，子RDD的paritioner为`HashPartitioner(3)`，如果父RDD的partitioner相同，显然没有必要再进行一次shuffle。

```java
def combineByKeyWithClassTag[C](
    createCombiner: V => C,
    mergeValue: (C, V) => C,
    mergeCombiners: (C, C) => C,
    partitioner: Partitioner,
    mapSideCombine: Boolean = true,
    serializer: Serializer = null)(implicit ct: ClassTag[C]): RDD[(K, C)] = self.withScope {
  require(mergeCombiners != null, "mergeCombiners must be defined") // required as of Spark 0.9.0
  // 如果key的类型为数组，则不支持map端聚合以及hash分区
  if (keyClass.isArray) {
    if (mapSideCombine) {
      throw SparkCoreErrors.cannotUseMapSideCombiningWithArrayKeyError()
    }
    if (partitioner.isInstanceOf[HashPartitioner]) {
      throw SparkCoreErrors.hashPartitionerCannotPartitionArrayKeyError()
    }
  }
  val aggregator = new Aggregator[K, V, C](
    self.context.clean(createCombiner),
    self.context.clean(mergeValue),
    self.context.clean(mergeCombiners))
  // 如果partitioner相同
  if (self.partitioner == Some(partitioner)) {
    self.mapPartitions(iter => {
      // 访问ThreadLocal变量，获取当前的taskContext
      val context = TaskContext.get()
      // aggregator创建ExternalAppendOnlyMap，用于实现combiner
      new InterruptibleIterator(context, aggregator.combineValuesByKey(iter, context))
    }, preservesPartitioning = true)
  } else {
    // parttioner不相同，进行一次shuffle
    new ShuffledRDD[K, V, C](self, partitioner)
      .setSerializer(serializer)
      .setAggregator(aggregator)
      .setMapSideCombine(mapSideCombine)
  }
}
```

在paritioner相同的情况下，调用了`mapPartitions`方法，实际的操作由`aggregator.combineValuesByKey`实现。

```scala
@DeveloperApi
case class Aggregator[K, V, C] (
    createCombiner: V => C,
    mergeValue: (C, V) => C,
    mergeCombiners: (C, C) => C) {

  def combineValuesByKey(
      iter: Iterator[_ <: Product2[K, V]],
      context: TaskContext): Iterator[(K, C)] = {
    val combiners = new ExternalAppendOnlyMap[K, V, C](createCombiner, mergeValue, mergeCombiners)
    combiners.insertAll(iter)
    updateMetrics(context, combiners)
    combiners.iterator
  }

  def combineCombinersByKey(
      iter: Iterator[_ <: Product2[K, C]],
      context: TaskContext): Iterator[(K, C)] = {
    val combiners = new ExternalAppendOnlyMap[K, C, C](identity, mergeCombiners, mergeCombiners)
    combiners.insertAll(iter)
    updateMetrics(context, combiners)
    combiners.iterator
  }

  /** Update task metrics after populating the external map. */
  private def updateMetrics(context: TaskContext, map: ExternalAppendOnlyMap[_, _, _]): Unit = {
    Option(context).foreach { c =>
      c.taskMetrics().incMemoryBytesSpilled(map.memoryBytesSpilled)
      c.taskMetrics().incDiskBytesSpilled(map.diskBytesSpilled)
      c.taskMetrics().incPeakExecutionMemory(map.peakMemoryUsedBytes)
    }
  }
}
```

`Aggregator`这个类有三个参数：

- createCombiner 用于从初值创建聚合结果，比如 a -> list[a]
- mergeValue 将新的值加入聚合结果，比如 b -> list[a, b]
- mergeCombiners 将两个聚合结果再聚合，比如 [c, d] -> list[a, b, c, d]

可以看到`combineValuesByKey`操作创建了`ExternalAppendOnlyMap`，功能类似于hashmap，聚合操作使用传入的聚合函数，将分区中的所有数据插入map中聚合，`ExternalAppendOnlyMap`实现了吐磁盘，在完成插入后会更新内存的信息，并返回map的迭代器。

### reduceByKey

```scala
def reduceByKey(func: (V, V) => V, numPartitions: Int): RDD[(K, V)] = self.withScope {
  reduceByKey(new HashPartitioner(numPartitions), func)
}
def reduceByKey(partitioner: Partitioner, func: (V, V) => V): RDD[(K, V)] = self.withScope {
  combineByKeyWithClassTag[V]((v: V) => v, func, func, partitioner)
}
```

`reduceByKey`使用reduce函数按key聚合，在map端先局地combine然后再在reduce端聚合。

`groupByKey`没有map端聚合的原因是即使聚合也不能减少传输的数据量和内存用量。

### aggregateByKey

```scala
def aggregateByKey[U: ClassTag](zeroValue: U, numPartitions: Int)(seqOp: (U, V) => U,
    combOp: (U, U) => U): RDD[(K, U)] = self.withScope {
  aggregateByKey(zeroValue, new HashPartitioner(numPartitions))(seqOp, combOp)
}
def aggregateByKey[U: ClassTag](zeroValue: U, partitioner: Partitioner)(seqOp: (U, V) => U,
    combOp: (U, U) => U): RDD[(K, U)] = self.withScope {
  // Serialize the zero value to a byte array so that we can get a new clone of it on each key
  val zeroBuffer = SparkEnv.get.serializer.newInstance().serialize(zeroValue)
  val zeroArray = new Array[Byte](zeroBuffer.limit)
  zeroBuffer.get(zeroArray)

  lazy val cachedSerializer = SparkEnv.get.serializer.newInstance()
  val createZero = () => cachedSerializer.deserialize[U](ByteBuffer.wrap(zeroArray))

  // We will clean the combiner closure later in `combineByKey`
  val cleanedSeqOp = self.context.clean(seqOp)
  combineByKeyWithClassTag[U]((v: V) => cleanedSeqOp(createZero(), v),
    cleanedSeqOp, combOp, partitioner)
}
```

`aggregateByKey`底层也是调用了`combineByKey`，可以看做是一个更加通用的`reduceByKey`，支持返回类型和value类型不一致，支持map端聚合函数和reduce聚合函数不相同。

### combineByKey

```scala
def combineByKeyWithClassTag[C](
    createCombiner: V => C,
    mergeValue: (C, V) => C,
    mergeCombiners: (C, C) => C,
    partitioner: Partitioner,
    mapSideCombine: Boolean = true,
    serializer: Serializer = null)(implicit ct: ClassTag[C]): RDD[(K, C)] = self.withScope {
```



前述的聚合函数都是基于combineByKey实现的，所以combineByKey也提供了最大的灵活性，比如`aggregateByKey`只能指定初始值，然而`combineByKey`可以通过函数为不同Key指定不同的初始值。

### foldByKey

```scala
def foldByKey(zeroValue: V, numPartitions: Int)(func: (V, V) => V): RDD[(K, V)] = self.withScope {
  foldByKey(zeroValue, new HashPartitioner(numPartitions))(func)
}
def foldByKey(
    zeroValue: V,
    partitioner: Partitioner)(func: (V, V) => V): RDD[(K, V)] = self.withScope {
  // Serialize the zero value to a byte array so that we can get a new clone of it on each key
  val zeroBuffer = SparkEnv.get.serializer.newInstance().serialize(zeroValue)
  val zeroArray = new Array[Byte](zeroBuffer.limit)
  zeroBuffer.get(zeroArray)

  // When deserializing, use a lazy val to create just one instance of the serializer per task
  lazy val cachedSerializer = SparkEnv.get.serializer.newInstance()
  val createZero = () => cachedSerializer.deserialize[V](ByteBuffer.wrap(zeroArray))

  val cleanedFunc = self.context.clean(func)
  combineByKeyWithClassTag[V]((v: V) => cleanedFunc(createZero(), v),
    cleanedFunc, cleanedFunc, partitioner)
}
```

foldByKey是一个简化的aggregateByKey，seqOp和combineOp共用一个func。

### cogroup/groupWith

```scala
def cogroup[W](other: RDD[(K, W)]): RDD[(K, (Iterable[V], Iterable[W]))] = self.withScope {
  cogroup(other, defaultPartitioner(self, other))
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
```

cogroup中文翻译成联合分组，将多个RDD中具有相同Key的Value聚合在一起，假设rdd1包含<K, V> record，rdd2包含<K, W> record，则两者聚合结果为`<K, (List<V>, List<W>)`。这个操作还有另一个名字groupwith。

cogroup操作实际生成了两个RDD，CoGroupedRDD将数据聚合在一起，MapPartitionsRDD仅对结果的数据类型进行转换。

### join

```scala
def join[W](other: RDD[(K, W)]): RDD[(K, (V, W))] = self.withScope {
  join(other, defaultPartitioner(self, other))
}
def join[W](other: RDD[(K, W)], partitioner: Partitioner): RDD[(K, (V, W))] = self.withScope {
  this.cogroup(other, partitioner).flatMapValues( pair =>
    for (v <- pair._1.iterator; w <- pair._2.iterator) yield (v, w)
  )
}
```

join和SQL中的join类似，将两个RDD中相同key的value联接起来，假设rdd1中的数据为<K, V>，rdd2中的数据为<K, W>，那么join之后的结果为<K, (V, W)>。在实现中，join首先调用了`cogroup`生成CoGroupedRDD和MapPartitionedRDD，然后使用flatMapValues计算相同key下value的笛卡尔积。

### cartesian

```scala
def cartesian[U: ClassTag](other: RDD[U]): RDD[(T, U)] = withScope {
  new CartesianRDD(sc, this, other)
}
```

cartesian操作生成两个RDD的笛卡尔积，假设RDD1中的分区个数为m，rdd2中的分区个数为n，cartesian操作会生成m * n个分区，rdd1和rdd2中的分区两两组合，组合后形成CartesianRDD中的一个分区，该分区中的数据是rdd1和rdd2相应的两个分区中数据的笛卡尔积。

### sortByKey

```scala
def sortByKey(ascending: Boolean = true, numPartitions: Int = self.partitions.length)
    : RDD[(K, V)] = self.withScope
{
  val part = new RangePartitioner(numPartitions, self, ascending)
  new ShuffledRDD[K, V, V](self, part)
    .setKeyOrdering(if (ascending) ordering else ordering.reverse)
}
```

sortByKey对rdd1中<K, V> record进行排序，注意只对key进行排序，在相同Key的情况相爱，并不对value进行排序。sortByKey首先通过range划分将数据分布到shuffledRDD的不同分区中，可以保证在生成的RDD中，partition1中的所有record的key小于（或大于）partition2中所有record的key。

### coalesce

```scala
def coalesce(numPartitions: Int, shuffle: Boolean = false,
             partitionCoalescer: Option[PartitionCoalescer] = Option.empty)
            (implicit ord: Ordering[T] = null)
    : RDD[T] = withScope {
  require(numPartitions > 0, s"Number of partitions ($numPartitions) must be positive.")
  if (shuffle) {
    /** Distributes elements evenly across output partitions, starting from a random partition. */
    val distributePartition = (index: Int, items: Iterator[T]) => {
      var position = new XORShiftRandom(index).nextInt(numPartitions)
      items.map { t =>
        // Note that the hash code of the key will just be the key itself. The HashPartitioner
        // will mod it with the number of total partitions.
        position = position + 1
        (position, t)
      }
    } : Iterator[(Int, T)]

    // include a shuffle step so that our upstream tasks are still distributed
    new CoalescedRDD(
      new ShuffledRDD[Int, T, T](
        mapPartitionsWithIndexInternal(distributePartition, isOrderSensitive = true),
        new HashPartitioner(numPartitions)),
      numPartitions,
      partitionCoalescer).values
  } else {
    new CoalescedRDD(this, numPartitions, partitionCoalescer)
  }
}
private[spark] def mapPartitionsWithIndexInternal[U: ClassTag](
    f: (Int, Iterator[T]) => Iterator[U],
    preservesPartitioning: Boolean = false,
    isOrderSensitive: Boolean = false): RDD[U] = withScope {
  new MapPartitionsRDD(
    this,
    (_: TaskContext, index: Int, iter: Iterator[T]) => f(index, iter),
    preservesPartitioning = preservesPartitioning,
    isOrderSensitive = isOrderSensitive)
}
```

coalesce用于将rdd的分区个数降低或者升高，在不使用shuffle的情况下，会直接生成CoalescedRDD，直接将相邻的分区合并，分区个数只能降低不能升高，当rdd中不同分区中的数据量差别较大时，直接合并容易造成数据倾斜（元素集中于少数分区中）。使用shffule直接解决数据倾斜问题，通过mapPartitionsWithIndex对输出RDD的每个分区进行操作，为原来的记录增加Key，Key是一个Int，对每个分区得到一个随机的起始位置，后续记录的Key是前一条记录的Key + 1，最后使用hash分组时相邻的记录会被分到不同的组。最终生成CoalescedRDD，并丢弃新生成的Key，通过map操作获取原来的记录。

### repartition

```scala
def repartition(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {
  coalesce(numPartitions, shuffle = true)
}
```

repartition操作底层使用了coalesce的shuffle版本。

### repartitionAndSortWithinPartitions

```scala
def repartitionAndSortWithinPartitions(partitioner: Partitioner): RDD[(K, V)] = self.withScope {
  if (self.partitioner == Some(partitioner)) {
    self.mapPartitions(iter => {
      val context = TaskContext.get()
      val sorter = new ExternalSorter[K, V, V](context, None, None, Some(ordering))
      new InterruptibleIterator(context,
        sorter.insertAllAndUpdateMetrics(iter).asInstanceOf[Iterator[(K, V)]])
    }, preservesPartitioning = true)
  } else {
    new ShuffledRDD[K, V, V](self, partitioner).setKeyOrdering(ordering)
  }
}
```

repartitionAndSortWithinPartitions可以灵活使用各种partitioner对数据进行分区，并且可以对输出RDD中的每个分区中的Key进行排序。这样相比于调用`repartition`然后在每个分区内排序效率更高，因为repartitionAndSortWithinPartitions可以将排序下推到shuffle机制中，注意结果只能保证是分区内有序，不能保证全局有序。

### intersection

```scala
def intersection(other: RDD[T]): RDD[T] = withScope {
  this.map(v => (v, null)).cogroup(other.map(v => (v, null)))
      .filter { case (_, (leftGroup, rightGroup)) => leftGroup.nonEmpty && rightGroup.nonEmpty }
      .keys
}
```

intersection求rdd1和rdd2的交集，输出RDD不包含任何重复的元素。从实现中可以看到，首先通过map函数将record转化为<K, V>类型，V为固定值null，然后通过cogroup将rdd1和rdd2中的record聚合在一起，过滤掉为空的record，最后只保留key，得到交集元素。

### distinct

```java
def distinct(numPartitions: Int)(implicit ord: Ordering[T] = null): RDD[T] = withScope {
  def removeDuplicatesInPartition(partition: Iterator[T]): Iterator[T] = {
    // Create an instance of external append only map which ignores values.
    val map = new ExternalAppendOnlyMap[T, Null, Null](
      createCombiner = _ => null,
      mergeValue = (a, b) => a,
      mergeCombiners = (a, b) => a)
    map.insertAll(partition.map(_ -> null))
    map.iterator.map(_._1)
  }
  partitioner match {
    case Some(_) if numPartitions == partitions.length =>
      mapPartitions(removeDuplicatesInPartition, preservesPartitioning = true)
    case _ => map(x => (x, null)).reduceByKey((x, _) => x, numPartitions).map(_._1)
  }
}
```

distinct是去重操作，对rdd中的数据进行去重，如果rdd已经有partitioner并且分区个数和预期分区个数相同，直接走分区内去重的逻辑，通过创建一个ExternalAppendOnlyMap，得到去重后的数据。其他情况下需要走shuffle逻辑，首先将record映射为<K, V>，V为固定值null，然后调用reduceByKey进行聚合，最终只保留key。

### union

```scala
def union(other: RDD[T]): RDD[T] = withScope {
  sc.union(this, other)
}
def union[T: ClassTag](first: RDD[T], rest: RDD[T]*): RDD[T] = withScope {
  union(Seq(first) ++ rest)
}
def union[T: ClassTag](rdds: Seq[RDD[T]]): RDD[T] = withScope {
  // 过滤空的RDD
  val nonEmptyRdds = rdds.filter(!_.partitions.isEmpty)
  val partitioners = nonEmptyRdds.flatMap(_.partitioner).toSet
  if (nonEmptyRdds.forall(_.partitioner.isDefined) && partitioners.size == 1) {
    new PartitionerAwareUnionRDD(this, nonEmptyRdds)
  } else {
    new UnionRDD(this, nonEmptyRdds)
  }
}
```

union表示将rdd1和rdd2中的元素合并到一起。如果所有RDD的partitioner都相同，则构造PartitionerAwareUnionRDD，分区个数与rdd1和rdd2的分区个数相同，且输出RDD中每个分区中的数据都是rdd1和rdd2对应分区合并的结果。如果rdd1和rdd2的partitioner不同，合并后的RDD为UnionRDD，分区个数是rdd1和rdd2的分区个数之和，输出RDD中的每个分区也一一对应rdd1或者rdd2中的相应的分区。

### zip

```scala
def zip[U: ClassTag](other: RDD[U]): RDD[(T, U)] = withScope {
  zipPartitions(other, preservesPartitioning = false) { (thisIter, otherIter) =>
    new Iterator[(T, U)] {
      def hasNext: Boolean = (thisIter.hasNext, otherIter.hasNext) match {
        case (true, true) => true
        case (false, false) => false
        case _ => throw SparkCoreErrors.canOnlyZipRDDsWithSamePartitionSizeError()
      }
      def next(): (T, U) = (thisIter.next(), otherIter.next())
    }
  }
}
```

将rdd1和rdd2中的元素按照一一对应关系连接在一起，构成<K, V> record。该操作要求rdd1和rdd2的分区个数相同，而且每个分区包含的元素个数相同。

### zipParitions

```scala
def zipPartitions[B: ClassTag, V: ClassTag]
    (rdd2: RDD[B], preservesPartitioning: Boolean)
    (f: (Iterator[T], Iterator[B]) => Iterator[V]): RDD[V] = withScope {
  new ZippedPartitionsRDD2(sc, sc.clean(f), this, rdd2, preservesPartitioning)
}
```

zipPartitions将rdd1和rdd2中的分区按照一一对应关系连接在一起，形成新的rdd。新的rdd中的每个分区的数据都通过对rdd1和rdd2中对应分区执行func函数得到，该操作要求rdd1和rdd2的分区个数相同，但不要求每个分区包含相同的元素个数。

### zipWithIndex

```scala
def zipWithIndex(): RDD[(T, Long)] = withScope {
  new ZippedWithIndexRDD(this)
}
```

对rdd1中的数据进行编号，编号方式是从0开始按序递增，直接返回`ZippedWithIndexRDD`

### zipWtihUniqueId

```scala
def zipWithUniqueId(): RDD[(T, Long)] = withScope {
  val n = this.partitions.length.toLong
  this.mapPartitionsWithIndex { case (k, iter) =>
    Utils.getIteratorZipWithIndex(iter, 0L).map { case (item, i) =>
      (item, i * n + k)
    }
  }
}
def getIteratorZipWithIndex[T](iter: Iterator[T], startIndex: Long): Iterator[(T, Long)] = {
  new Iterator[(T, Long)] {
    require(startIndex >= 0, "startIndex should be >= 0.")
    var index: Long = startIndex - 1L
    def hasNext: Boolean = iter.hasNext
    def next(): (T, Long) = {
      index += 1L
      (iter.next(), index)
    }
  }
}
```

对rdd1中的数据进行编号，编号方式为round-robin，就像给每个人轮流发扑克牌，如果某些分区比较小，原本应该分给这个分区的编号会轮空，而不是分配给另一个分区。zipWithUniqueId通过mapPartitionsWithIndex实现，返回MapPartitionsRDD

### subtractByKey

```scala
def subtractByKey[W: ClassTag](other: RDD[(K, W)]): RDD[(K, V)] = self.withScope {
  subtractByKey(other, self.partitioner.getOrElse(new HashPartitioner(self.partitions.length)))
}
def subtractByKey[W: ClassTag](other: RDD[(K, W)], p: Partitioner): RDD[(K, V)] = self.withScope {
  new SubtractedRDD[K, V, W](self, other, p)
}
```

subtractByKey计算出key在rdd1中而不在rdd2中的record，逻辑类似于cogroup，但实现比CoGroupedRDD更加高效，生成SubtractedRDD。

使用rdd1的paritioner或者分区个数，因为结果集不会大于rdd1

### subtract

```scala
def subtract(other: RDD[T]): RDD[T] = withScope {
  subtract(other, partitioner.getOrElse(new HashPartitioner(partitions.length)))
}
def subtract(
    other: RDD[T],
    p: Partitioner)(implicit ord: Ordering[T] = null): RDD[T] = withScope {
  if (partitioner == Some(p)) {
    // Our partitioner knows how to handle T (which, since we have a partitioner, is
    // really (K, V)) so make a new Partitioner that will de-tuple our fake tuples
    val p2 = new Partitioner() {
      override def numPartitions: Int = p.numPartitions
      override def getPartition(k: Any): Int = p.getPartition(k.asInstanceOf[(Any, _)]._1)
    }
    // Unfortunately, since we're making a new p2, we'll get ShuffleDependencies
    // anyway, and when calling .keys, will not have a partitioner set, even though
    // the SubtractedRDD will, thanks to p2's de-tupled partitioning, already be
    // partitioned by the right/real keys (e.g. p).
    this.map(x => (x, null)).subtractByKey(other.map((_, null)), p2).keys
  } else {
    this.map(x => (x, null)).subtractByKey(other.map((_, null)), p).keys
  }
}
```

将record映射为<K, V> record，V为null，是一个比较常见的思路，这样可以复用代码。

### sortBy

```scala
def sortBy[K](
    f: (T) => K,
    ascending: Boolean = true,
    numPartitions: Int = this.partitions.length)
    (implicit ord: Ordering[K], ctag: ClassTag[K]): RDD[T] = withScope {
  this.keyBy[K](f)
      .sortByKey(ascending, numPartitions)
      .values
}
def keyBy[K](f: T => K): RDD[(K, T)] = withScope {
  val cleanedF = sc.clean(f)
  map(x => (cleanedF(x), x))
}
```

sortBy基于func的计算结果对rdd1中的recorc进行排序，底层使用sortByKey实现。

### glom

```scala
def glom(): RDD[Array[T]] = withScope {
  new MapPartitionsRDD[Array[T], T](this, (_, _, iter) => Iterator(iter.toArray))
}
```

将rdd1中的每个分区的record合并到一个list中，底层通过MapPartitionsRDD实现。

## 常用action数据操作

action数据操作是用来对计算结果进行后处理，同时提交计算job。可以通过返回值区分一个操作是action还是transformation，transformation操作一般返回RDD类型，而action操作一般返回数值、数据结果（如Map）或者不返回任何值（比如写磁盘）。

### count

```scala
def count(): Long = sc.runJob(this, Utils.getIteratorSize _).sum
def getIteratorSize(iterator: Iterator[_]): Long = {
  if (iterator.knownSize >= 0) iterator.knownSize.toLong
  else {
    var count = 0L
    while (iterator.hasNext) {
      count += 1L
      iterator.next()
    }
    count
  }
}
```

count操作首先计算每个分区中record的数目，然后在Driver端进行累加操作，返回rdd中包含的record个数。

### countByKey

```scala
def countByKey(): Map[K, Long] = self.withScope {
  self.mapValues(_ => 1L).reduceByKey(_ + _).collect().toMap
}
```

countByKey统计rdd中每个key出现的次数，返回一个map，要求rdd是<K, V>类型。countByKey首先通过mapValues将<K, V> record中的Value设置为1，然后利用reduceByKey统计每个key出现的次数，最后使用`collect`方法将结果收集到Driver端。

### countByValue

```scala
def countByValue()(implicit ord: Ordering[T] = null): Map[T, Long] = withScope {
  map(value => (value, null)).countByKey()
}
```

countByValue并不是统计<K, V> record中每个Value出现的次数，而是统计每个record出现的次数。底层首先通过map函数将record转成<K, V> record，Value为null，然后调用countByKey统计Key的次数。

### collect

```scala
def collect(): Array[T] = withScope {
  val results = sc.runJob(this, (iter: Iterator[T]) => iter.toArray)
  import org.apache.spark.util.ArrayImplicits._
  Array.concat(results.toImmutableArraySeq: _*)
}
```

collect操作将rdd中的record收集到Driver端，返回类型为`Array[T]`

### collectAsMap

```scala
def collectAsMap(): Map[K, V] = self.withScope {
  val data = self.collect()
  val map = new mutable.HashMap[K, V]
  map.sizeHint(data.length)
  data.foreach { pair => map.put(pair._1, pair._2) }
  map
}
```

collectAsMap通过collect调用将<K, V> record收集到Driver端。

### foreach

```scala
def foreach(f: T => Unit): Unit = withScope {
  val cleanF = sc.clean(f)
  sc.runJob(this, (iter: Iterator[T]) => iter.foreach(cleanF))
}
```

将rdd中的每个record按照func进行处理，底层调用runJob。

### foreachPartitions

```scala
def foreachPartition(f: Iterator[T] => Unit): Unit = withScope {
  val cleanF = sc.clean(f)
  sc.runJob(this, (iter: Iterator[T]) => cleanF(iter))
}
```

将rdd中的每个分区中的数据按照func进行处理，底层调用runJob。

### fold

```scala
def fold(zeroValue: T)(op: (T, T) => T): T = withScope {
  // Clone the zero value since we will also be serializing it as part of tasks
  var jobResult = Utils.clone(zeroValue, sc.env.closureSerializer.newInstance())
  val cleanOp = sc.clean(op)
  val foldPartition = (iter: Iterator[T]) => iter.fold(zeroValue)(cleanOp)
  val mergeResult = (_: Int, taskResult: T) => jobResult = op(jobResult, taskResult)
  sc.runJob(this, foldPartition, mergeResult)
  jobResult
}
```

fold将rdd中的record按照func进行聚合，首先在rdd的每个分区中计算出局部结果即函数`foldPartition`，然后在Driver段将局部结果聚合成最终结果即函数`mergeResult`。需要注意的是，fold每次聚合是初始值zeroValue都会参与计算。

### reduce

```scala
def reduce(f: (T, T) => T): T = withScope {
  val cleanF = sc.clean(f)
  val reducePartition: Iterator[T] => Option[T] = iter => {
    if (iter.hasNext) {
      Some(iter.reduceLeft(cleanF))
    } else {
      None
    }
  }
  var jobResult: Option[T] = None
  val mergeResult = (_: Int, taskResult: Option[T]) => {
    if (taskResult.isDefined) {
      jobResult = jobResult match {
        case Some(value) => Some(f(value, taskResult.get))
        case None => taskResult
      }
    }
  }
  sc.runJob(this, reducePartition, mergeResult)
  // Get the final result out of our Option, or throw an exception if the RDD was empty
  jobResult.getOrElse(throw SparkCoreErrors.emptyCollectionError())
}
```

将rdd中的record按照func进行聚合，这里没有提供初始值，所以需要处理空值的情况。

### aggregate

```scala
def aggregate[U: ClassTag](zeroValue: U)(seqOp: (U, T) => U, combOp: (U, U) => U): U = withScope {
  // Clone the zero value since we will also be serializing it as part of tasks
  var jobResult = Utils.clone(zeroValue, sc.env.serializer.newInstance())
  val cleanSeqOp = sc.clean(seqOp)
  val aggregatePartition = (it: Iterator[T]) => it.foldLeft(zeroValue)(cleanSeqOp)
  val mergeResult = (_: Int, taskResult: U) => jobResult = combOp(jobResult, taskResult)
  sc.runJob(this, aggregatePartition, mergeResult)
  jobResult
}
```

将rdd中的record按照func进行聚合，这里提供了初始值，分区聚合和Driver端聚合都会使用初始值。

为什么已经有了reduceByKey、aggregateByKey等操作，还要定义aggreagte和reduce等操作呢？虽然reduceByKey、aggregateByKey等操作可以对每个分区中的record，以及跨分区且具有相同Key的record进行聚合，但这些聚合都是在部分数据上，类似于`<K, func(list(V))`，而不是针对所有record进行全局聚合，即`func(<K, list(V))`。

然而aggregate、reduce等操作存在相同的问题，当需要merge的部分结果很大时，数据传输量很大，而且Driver是单点merge，存在效率和内存空间限制的问题，为了解决这个问题，Spark对这些聚合操作进行了优化，提出了treeAggregate和treeReduce操作。

### treeAggregate

```scala
def treeAggregate[U: ClassTag](zeroValue: U)(
    seqOp: (U, T) => U,
    combOp: (U, U) => U,
    depth: Int = 2): U = withScope {
    treeAggregate(zeroValue, seqOp, combOp, depth, finalAggregateOnExecutor = false)
}
def treeAggregate[U: ClassTag](
    zeroValue: U,
    seqOp: (U, T) => U,
    combOp: (U, U) => U,
    depth: Int,
    finalAggregateOnExecutor: Boolean): U = withScope {
    require(depth >= 1, s"Depth must be greater than or equal to 1 but got $depth.")
  if (partitions.length == 0) {
    Utils.clone(zeroValue, context.env.closureSerializer.newInstance())
  } else {
    val cleanSeqOp = context.clean(seqOp)
    val cleanCombOp = context.clean(combOp)
    val aggregatePartition =
      (it: Iterator[T]) => it.foldLeft(zeroValue)(cleanSeqOp)
    var partiallyAggregated: RDD[U] = mapPartitions(it => Iterator(aggregatePartition(it)))
    var numPartitions = partiallyAggregated.partitions.length
    val scale = math.max(math.ceil(math.pow(numPartitions, 1.0 / depth)).toInt, 2)
    // If creating an extra level doesn't help reduce
    // the wall-clock time, we stop tree aggregation.

    // Don't trigger TreeAggregation when it doesn't save wall-clock time
    while (numPartitions > scale + math.ceil(numPartitions.toDouble / scale)) {
      numPartitions /= scale
      val curNumPartitions = numPartitions
      partiallyAggregated = partiallyAggregated.mapPartitionsWithIndex {
        (i, iter) => iter.map((i % curNumPartitions, _))
      }.foldByKey(zeroValue, new HashPartitioner(curNumPartitions))(cleanCombOp).values
    }
    if (finalAggregateOnExecutor && partiallyAggregated.partitions.length > 1) {
      // map the partially aggregated rdd into a key-value rdd
      // do the computation in the single executor with one partition
      // get the new RDD[U]
      partiallyAggregated = partiallyAggregated
        .map(v => (0.toByte, v))
        .foldByKey(zeroValue, new ConstantPartitioner)(cleanCombOp)
        .values
    }
    val copiedZeroValue = Utils.clone(zeroValue, sc.env.closureSerializer.newInstance())
    partiallyAggregated.fold(copiedZeroValue)(cleanCombOp)
  }
}

```

treeAggreagte是为了解决aggregate在Driver端聚合导致的数据传输量大、单点merge、内存空间限制等问题，思路类似于归并排序的层次归并，每层都将分区数目降低为原来的`1/scale`，也就是一颗近似完美的平衡树，让每层每个节点的负载都相对合理。我们可以在参数中指定depth，假设分区数量为N，则近似有`N / (scale^depth) = 1`。当然Spark在何时停止局部聚合做了优化，平衡效率和开销，选择在`numPartitions > scale + math.ceil(numPartitions.toDouble / scale`时停止局部聚合，`numPartitions`表示当前分区数，`numParttions/scale`表示如果继续局部聚合下一层的分区数，为什么会有一个额外的`scale`，我认为应该是为了避免极端情况，比如分区数为2，scale为2， 那么如果没有额外的scale作为成本，这里会继续局部聚合，然后有了额外的scale。

实现上局部聚合使用了foldByKey，尽管形式上使用了ShuffleDependency，但是由于每个分区中只有一条记录，实际数据传输时类似于多对一的NarrowDependency。

### treeReduce

treeReduce是reduce的优化版本。底层实际上调用了treeAggregate。

```scala
def treeReduce(f: (T, T) => T, depth: Int = 2): T = withScope {
  require(depth >= 1, s"Depth must be greater than or equal to 1 but got $depth.")
  val cleanF = context.clean(f)
  val reducePartition: Iterator[T] => Option[T] = iter => {
    if (iter.hasNext) {
      Some(iter.reduceLeft(cleanF))
    } else {
      None
    }
  }
  val partiallyReduced = mapPartitions(it => Iterator(reducePartition(it)))
  val op: (Option[T], Option[T]) => Option[T] = (c, x) => {
    if (c.isDefined && x.isDefined) {
      Some(cleanF(c.get, x.get))
    } else if (c.isDefined) {
      c
    } else if (x.isDefined) {
      x
    } else {
      None
    }
  }
  partiallyReduced.treeAggregate(Option.empty[T])(op, op, depth)
    .getOrElse(throw SparkCoreErrors.emptyCollectionError())
}
```

### reduceByKeyLocally

```scala
def reduceByKeyLocally(func: (V, V) => V): Map[K, V] = self.withScope {
  val cleanedF = self.sparkContext.clean(func)

  if (keyClass.isArray) {
    throw SparkCoreErrors.reduceByKeyLocallyNotSupportArrayKeysError()
  }

  val reducePartition = (iter: Iterator[(K, V)]) => {
    val map = new JHashMap[K, V]
    iter.foreach { pair =>
      val old = map.get(pair._1)
      map.put(pair._1, if (old == null) pair._2 else cleanedF(old, pair._2))
    }
    Iterator(map)
  } : Iterator[JHashMap[K, V]]

  val mergeMaps = (m1: JHashMap[K, V], m2: JHashMap[K, V]) => {
    m2.asScala.foreach { pair =>
      val old = m1.get(pair._1)
      m1.put(pair._1, if (old == null) pair._2 else cleanedF(old, pair._2))
    }
    m1
  } : JHashMap[K, V]

  self.mapPartitions(reducePartition).reduce(mergeMaps).asScala
}
```

reduceByKeyLocally首先在rdd的各个分区中进行聚合，并使用HashMap来存储聚合结果，然后将数据汇总到Driver端进行全局聚合，仍然是将聚合结果存在到HashMap。

### take

```scala
def take(num: Int): Array[T] = withScope {
  val scaleUpFactor = Math.max(conf.get(RDD_LIMIT_SCALE_UP_FACTOR), 2)
  if (num == 0) {
    new Array[T](0)
  } else {
    val buf = new ArrayBuffer[T]
    val totalParts = this.partitions.length
    var partsScanned = 0
    while (buf.size < num && partsScanned < totalParts) {
      // The number of partitions to try in this iteration. It is ok for this number to be
      // greater than totalParts because we actually cap it at totalParts in runJob.
      var numPartsToTry = conf.get(RDD_LIMIT_INITIAL_NUM_PARTITIONS)
      val left = num - buf.size
      if (partsScanned > 0) {
        // If we didn't find any rows after the previous iteration, multiply by
        // limitScaleUpFactor and retry. Otherwise, interpolate the number of partitions we need
        // to try, but overestimate it by 50%. We also cap the estimation in the end.
        if (buf.isEmpty) {
          numPartsToTry = partsScanned * scaleUpFactor
        } else {
          // As left > 0, numPartsToTry is always >= 1
          numPartsToTry = Math.ceil(1.5 * left * partsScanned / buf.size).toInt
          numPartsToTry = Math.min(numPartsToTry, partsScanned * scaleUpFactor)
        }
      }

      val p = partsScanned.until(math.min(partsScanned + numPartsToTry, totalParts))
      val res = sc.runJob(this, (it: Iterator[T]) => it.take(left).toArray, p)

      res.foreach(buf ++= _.take(num - buf.size))
      partsScanned += p.size
    }

    buf.toArray
  }
}
```

take表示从rdd中取出前num个record。take操作首先取出rdd中第一个分区的前num个record，如果num大于partition1中record的总数，则take会继续从后续的分区中取出record，为了提高效率，spark会根据前面分区分区的平均大小估计后续需要取几个分区来满足take的需求。

### first

```scala
def first(): T = withScope {
  take(1) match {
    case Array(t) => t
    case _ => throw SparkCoreErrors.emptyCollectionError()
  }
}
```

只取出rdd中的第一个record。底层通过take(1)实现。

### takeOrdered

```scala
def takeOrdered(num: Int)(implicit ord: Ordering[T]): Array[T] = withScope {
  if (num == 0 || this.getNumPartitions == 0) {
    Array.empty
  } else {
    this.mapPartitionsWithIndex { case (pid, iter) =>
      if (iter.nonEmpty) {
        // Priority keeps the largest elements, so let's reverse the ordering.
        Iterator.single(collectionUtils.takeOrdered(iter, num)(ord).toArray)
      } else if (pid == 0) {
        // make sure partition 0 always returns an array to avoid reduce on empty RDD
        Iterator.single(Array.empty[T])
      } else {
        Iterator.empty
      }
    }.reduce { (array1, array2) =>
      val size = math.min(num, array1.length + array2.length)
      val array = Array.ofDim[T](size)
      collectionUtils.mergeOrdered[T](Seq(array1, array2))(ord).copyToArray(array, 0, size)
      array
    }
  }
}
```

取出rdd中最小的num个record。首先使用mapPartitionsWithIndex在每个分区中找出最小的num个record，因为全局最小的n个元素一定是每个分区中最小的n个元素的子集，然后通过reduce操作将这些record收集到Driver段，进行排序，然后取出前num个record。

### top

```scala
def top(num: Int)(implicit ord: Ordering[T]): Array[T] = withScope {
  takeOrdered(num)(ord.reverse)
}
```

取出rdd中最大的num个record。底层通过takeOrdered实现。

### max/min

```scala
def max()(implicit ord: Ordering[T]): T = withScope {
  this.reduce(ord.max)
}
def min()(implicit ord: Ordering[T]): T = withScope {
  this.reduce(ord.min)
}
```

返回rdd中的最大、最小值。底层基于reduce实现。

### isEmpty

```scala
def isEmpty(): Boolean = withScope {
  partitions.length == 0 || take(1).length == 0
}
```

判断rdd是否为空，如果rdd不包含任何record，则返回true。如果分区数为0，则rdd一定为空，分区数大于0并不意味着rdd一定不为空，需要通过`take(1)`判断是否有数据。如果对rdd执行一些数据操作，比如过滤、求交集等，rdd为空的话，那么执行其他操作也一定为空，因此，提前判断rdd是否为空，可以避免提交冗余的job。

### lookup

```scala
def lookup(key: K): Seq[V] = self.withScope {
  self.partitioner match {
    case Some(p) =>
      val index = p.getPartition(key)
      val process = (it: Iterator[(K, V)]) => {
        val buf = new ArrayBuffer[V]
        for (pair <- it if pair._1 == key) {
          buf += pair._2
        }
        buf.toSeq
      } : Seq[V]
      val res = self.context.runJob(self, process, Array(index).toImmutableArraySeq)
      res(0)
    case None =>
      self.filter(_._1 == key).map(_._2).collect().toImmutableArraySeq
  }
}
```

loopup函数找出rdd中包含特定key的value，将这些value形成List。loopup首先过滤出给定key的record，然后使用map得到相应的value，最后使用collect将这些value收集到Driver端形成list。如果rdd的partitioner已经确定，那么在过滤前，通过getPartition确定key所在的分区，减少操作的数据量。

### saveAsTextFile

```scala
def saveAsTextFile(path: String): Unit = withScope {
  saveAsTextFile(path, null)
}
def saveAsTextFile(path: String, codec: Class[_ <: CompressionCodec]): Unit = withScope {
  this.mapPartitions { iter =>
    val text = new Text()
    iter.map { x =>
      require(x != null, "text files do not allow null rows")
      text.set(x.toString)
      (NullWritable.get(), text)
    }
  }.saveAsHadoopFile[TextOutputFormat[NullWritable, Text]](path, codec)
}
```

saveAsTextFile将rdd保存成文本文件，通过toString操作获取record的字符串形式，然后将record转化为<NullWriter, Text>类型，NullWriter的意思是控血，也就是每条输出数据只包含类似为文本的Value。底层调用saveAsHadoopFile。

### saveAsObjectFile

```scala
def saveAsObjectFile(path: String): Unit = withScope {
  this.mapPartitions(iter => iter.grouped(10).map(_.toArray))
    .map(x => (NullWritable.get(), new BytesWritable(Utils.serialize(x))))
    .saveAsSequenceFile(path)
}
```

saveAsObjectFile将rdd保存为序列化对象形式的SequenceFile，针对普通对象类型，将record惊醒序列化，并且以每10个record为1组转化为`SequenceFile<NullableWritable, Array[Object]>`，调用saveAsSequenceFile将文件写入HDFS中。

### saveAsSequenceFile

```scala
def saveAsSequenceFile(
    path: String,
    codec: Option[Class[_ <: CompressionCodec]] = None): Unit = self.withScope {
  def anyToWritable[U: IsWritable](u: U): Writable = u

  // TODO We cannot force the return type of `anyToWritable` be same as keyWritableClass and
  // valueWritableClass at the compile time. To implement that, we need to add type parameters to
  // SequenceFileRDDFunctions. however, SequenceFileRDDFunctions is a public class so it will be a
  // breaking change.
  val convertKey = self.keyClass != _keyWritableClass
  val convertValue = self.valueClass != _valueWritableClass

  logInfo(log"Saving as sequence file of type " +
    log"(${MDC(LogKeys.KEY, _keyWritableClass.getSimpleName)}," +
    log"${MDC(LogKeys.VALUE, _valueWritableClass.getSimpleName)})")
  val format = classOf[SequenceFileOutputFormat[Writable, Writable]]
  val jobConf = new JobConf(self.context.hadoopConfiguration)
  if (!convertKey && !convertValue) {
    self.saveAsHadoopFile(path, _keyWritableClass, _valueWritableClass, format, jobConf, codec)
  } else if (!convertKey && convertValue) {
    self.map(x => (x._1, anyToWritable(x._2))).saveAsHadoopFile(
      path, _keyWritableClass, _valueWritableClass, format, jobConf, codec)
  } else if (convertKey && !convertValue) {
    self.map(x => (anyToWritable(x._1), x._2)).saveAsHadoopFile(
      path, _keyWritableClass, _valueWritableClass, format, jobConf, codec)
  } else if (convertKey && convertValue) {
    self.map(x => (anyToWritable(x._1), anyToWritable(x._2))).saveAsHadoopFile(
      path, _keyWritableClass, _valueWritableClass, format, jobConf, codec)
  }
}
```

saveAsSequenceFile将rdd保存为SequenceFile形式的文件，针对<K, V> 类型的record，将record进行序列化后，以SequenceFile形式写入分布式文件系统中，底层调用saveAsHadoopFile实现。

### saveAsHadoopFile

```scala
def saveAsHadoopFile(
    path: String,
    keyClass: Class[_],
    valueClass: Class[_],
    outputFormatClass: Class[_ <: OutputFormat[_, _]],
    conf: JobConf = new JobConf(self.context.hadoopConfiguration),
    codec: Option[Class[_ <: CompressionCodec]] = None): Unit = self.withScope {
  // Rename this as hadoopConf internally to avoid shadowing (see SPARK-2038).
  val hadoopConf = conf
  hadoopConf.setOutputKeyClass(keyClass)
  hadoopConf.setOutputValueClass(valueClass)
  conf.setOutputFormat(outputFormatClass)
  for (c <- codec) {
    hadoopConf.setCompressMapOutput(true)
    hadoopConf.set("mapreduce.output.fileoutputformat.compress", "true")
    hadoopConf.setMapOutputCompressorClass(c)
    hadoopConf.set("mapreduce.output.fileoutputformat.compress.codec", c.getCanonicalName)
    hadoopConf.set("mapreduce.output.fileoutputformat.compress.type",
      CompressionType.BLOCK.toString)
  }

  // Use configured output committer if already set
  if (conf.getOutputCommitter == null) {
    hadoopConf.setOutputCommitter(classOf[FileOutputCommitter])
  }

  // When speculation is on and output committer class name contains "Direct", we should warn
  // users that they may loss data if they are using a direct output committer.
  val speculationEnabled = self.conf.get(SPECULATION_ENABLED)
  val outputCommitterClass = hadoopConf.get("mapred.output.committer.class", "")
  if (speculationEnabled && outputCommitterClass.contains("Direct")) {
    val warningMessage =
      log"${MDC(CLASS_NAME, outputCommitterClass)} " +
        log"may be an output committer that writes data directly to " +
        log"the final location. Because speculation is enabled, this output committer may " +
        log"cause data loss (see the case in SPARK-10063). If possible, please use an output " +
        log"committer that does not have this behavior (e.g. FileOutputCommitter)."
    logWarning(warningMessage)
  }

  FileOutputFormat.setOutputPath(hadoopConf,
    SparkHadoopWriterUtils.createPathFromString(path, hadoopConf))
  saveAsHadoopDataset(hadoopConf)
}
def saveAsHadoopDataset(conf: JobConf): Unit = self.withScope {
  val config = new HadoopMapRedWriteConfigUtil[K, V](new SerializableJobConf(conf))
  SparkHadoopWriter.write(
    rdd = self,
    config = config)
}
```

saveAsHadoopFile将rdd保存为Haddop HDFS文件系统支持的文件，进行必要的初始化和配置后，通过`SparkHadoopWriter`将rdd写入hadoop中。




