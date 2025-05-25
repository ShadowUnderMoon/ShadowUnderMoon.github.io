---
title: "Spark物理执行计划"
author: "爱吃芒果"
description:
date: "2025-05-25T11:16:21+08:00"
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

## Spark物理执行计划生成方法

Spark具体采用3个步骤来生成物理执行计划，首先根据action操作顺序将应用划分为作业（job），然后根据每个job的逻辑处理流程中的ShuffleDependency依赖关系，将job划分为执行阶段（stage）。最后在每个stage中，根据最后生成的RDD的分区个数生成多个计算任务（task）。

### 根据action操作将应用划分为作业（job）

当应用程序出现action操作时，如resultRDD.action()，表示应用会生成一个job，该job的逻辑处理流程为从输出数据到resultRDD的逻辑处理流程。

### 根据ShuffleDependency依赖关系将job划分成执行阶段（stage）

对于每个job，从其最后的RDD往前回溯整个逻辑处理流程，如果遇到NarrowDependency，则将当前RDD的parent RDD纳入，并继续向前追溯，当遇到ShuffleDependency时，停止回溯，将当前已经纳入的所有RDD按照其依赖关系建立一个执行阶段，命名为stage i。

如果将存在ShuffleDependency依赖的RDD也纳入同一个stage，计算每个分区时都需要重复计算ShuffleDependency上游的RDD，这显然没有必要。

### 根据分区计算将各个stage划分成计算任务（task）

每个分区上的计算逻辑相同，而且是独立的，因此每个分区上的计算可以独立成为一个task。同一个stage中的task可以同时分发到不同的机器并行执行。

### job、stage和task的计算顺序

job的提交时间和action被调用的时间有关，当应用程序执行到rdd.action()时，就会立即将rdd.actioin()形成的job提交给spark。job的逻辑处理流程实际上是一个DAG图，经过stage划分后，仍然是DAG图形状。每个stage的输出数据要不是job的输入数据，要不是上游stage的输出结果。因此，计算顺序从包含输入数据的stage开始，从前往后依次执行，仅当上游stage都执行完成后，再执行下游的stage。stage中的每个task因为是独立而且同构的，可以并行执行没有先后之分。

### task内部数据的存储的计算问题（流水线计算）

假设一个分区中有三条记录，分别为record1, record2, record3，需要对分区先执行f()操作，再执行g()操作，假设f()操作和g()操作都只依赖于上游分区中的单条记录，则可以采用流水线计算。类似于`record1 -> f(record1) -> record1' -> g(record') -> record''`，在task计算时只需要再内存中保留当前被处理的单个record即可，没有必要在执行f(record1)之前将record2和record3提前计算出来放入内存中。当然，如果f()操作和g()操作都依赖于上游分区中的多条记录，则流水线计算退化到计算-回收模式，需要一次读取上游分区中的所有数据，每执行完一个操作，回收之前的中间计算结果。

Spark采用流水线式计算来提高task的执行效率，减少内存使用量。这也是Spark可以在有限内存中处理大量大规模数据的原因。然而对于某些需要聚合中间计算结果的操作，还是需要占用一定的内存空间，也会在一定程度上影响流水线计算的效率。

### task间的数据传递和计算问题

stage之间存在的依赖关系是ShuffleDependency，而ShuffleDependency是部分依赖的，也就是下游stage中的每个task需要从parent RDD的每个分区中获取部分数据。ShuffleDependency的数据划分方式包括Hash划分、Range划分等，也就是要求上游stage预先将输出数据进行划分，按照分区存在，分区个数和下游task的个数一致，这个过程被称为Shuffle Write。按照分区存放完成后，下游的task将属于自己分区的数据通过网络传输获取，然后将来自上游不同分区的数据聚合在一起进行处理，这个过程被称为Shuffle Read。

### stage和task的命名方式

在Spark中，stage也可以有多个，有些stage既包含类似reduce的聚合操作有包含map操作，所以一般不区分是map stage还是reduce stage，而直接使用stage i来命名。

如果task的输出结果需要进行ShuffleWrite，以便传输给下一个stage，那么这些task被称为ShuffleMapTasks，而如果task的输出结果会汇总到Driver端或者直接写入分布式文件系统，那么这些task被称为ResultTasks。

## 生成物理执行计划的源码分析

### demo程序

```scala

// scalastyle:off println
package org.apache.spark.examples

import java.util.concurrent.TimeUnit

import scala.collection.compat.immutable.ArraySeq

import org.apache.spark.{HashPartitioner, SparkContext}
import org.apache.spark.sql.SparkSession

object FilterDemo {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder()
      .appName("MapDemo")
      .master("local")
      .getOrCreate()
    val sc = spark.sparkContext.asInstanceOf[SparkContext]

    val data1 = Array[(Int, Char)]((1, 'a'), (2, 'b'), (3, 'c'),
      (4, 'd'), (5, 'e'), (3, 'f'), (2, 'g'), (1, 'h'))
    val rdd1 = sc.parallelize(ArraySeq.unsafeWrapArray(data1), 3)
    val partitionedRDD = rdd1.partitionBy(new HashPartitioner(3))

    val data2 = Array[(Int, String)]((1, "A"), (2, "B"),
      (3, "C"), (4, "D"))
    val rdd2 = sc.parallelize(ArraySeq.unsafeWrapArray(data2), 2)
      .map(x => (x._1, x._2 + "" + x._2))

    val data3 = Array[(Int, String)]((3, "X"), (5, "Y"),
      (3, "Z"), (4, "Y"))
    val rdd3 = sc.parallelize(ArraySeq.unsafeWrapArray(data3), 2)

    val unionedRDD = rdd2.union(rdd3)
    val resultRDD = partitionedRDD.join(unionedRDD)
    resultRDD.count()
    spark.stop()

  }
}

```

![image-20250525163523755](SPARK_UI_DAG.png)

![image-20250525163739208](SPARK_UI_RDD.png)

![image-20250525163906199](SPARK_UI_RDD2.png)

涉及到以下RDD类别

- ParallelCollectionRDD
- ShuffledRDD
- CoGroupedRDD
- MapPartitionsRDD
- UnionRDD

### count操作

```scala
def count(): Long = sc.runJob(this, Utils.getIteratorSize _).sum
def runJob[T, U: ClassTag](rdd: RDD[T], func: Iterator[T] => U): Array[U] = {
  runJob(rdd, func, rdd.partitions.indices)
}
def runJob[T, U: ClassTag](
    rdd: RDD[T],
    func: Iterator[T] => U,
    partitions: Seq[Int]): Array[U] = {
  val cleanedFunc = clean(func)
  runJob(rdd, (ctx: TaskContext, it: Iterator[T]) => cleanedFunc(it), partitions)
}
def runJob[T, U: ClassTag](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int]): Array[U] = {
  val results = new Array[U](partitions.size)
  runJob[T, U](rdd, func, partitions, (index, res) => results(index) = res)
  results
}
def runJob[T, U: ClassTag](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    resultHandler: (Int, U) => Unit): Unit = {
  if (stopped.get()) {
    throw new IllegalStateException("SparkContext has been shutdown")
  }
  val callSite = getCallSite()
  val cleanedFunc = clean(func)
  logInfo(log"Starting job: ${MDC(LogKeys.CALL_SITE_SHORT_FORM, callSite.shortForm)}")
  if (conf.getBoolean("spark.logLineage", false)) {
    logInfo(log"RDD's recursive dependencies:\n" +
      log"${MDC(LogKeys.RDD_DEBUG_STRING, rdd.toDebugString)}")
  }
  dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
  progressBar.foreach(_.finishAll())
  rdd.doCheckpoint()
}
```

count操作会调用runJob创建并执行新的job，`getCallSite`通过堆栈找到用户调用代码的位置以及调用的spark方法，类似于`union at FilterDemo.scala:35`。

```scala
def runJob[T, U](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    callSite: CallSite,
    resultHandler: (Int, U) => Unit,
    properties: Properties): Unit = {
  val start = System.nanoTime
  val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
  ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
  waiter.completionFuture.value.get match {
    case scala.util.Success(_) =>
      logInfo(log"Job ${MDC(LogKeys.JOB_ID, waiter.jobId)} finished: " +
        log"${MDC(LogKeys.CALL_SITE_SHORT_FORM, callSite.shortForm)}, took " +
        log"${MDC(LogKeys.TIME, (System.nanoTime - start) / 1e6)} ms")
    case scala.util.Failure(exception) =>
      logInfo(log"Job ${MDC(LogKeys.JOB_ID, waiter.jobId)} failed: " +
        log"${MDC(LogKeys.CALL_SITE_SHORT_FORM, callSite.shortForm)}, took " +
        log"${MDC(LogKeys.TIME, (System.nanoTime - start) / 1e6)} ms")
      // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
      val callerStackTrace = Thread.currentThread().getStackTrace.tail
      exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
      throw exception
  }
}
```

DAGScheduler负责提交job，并等待job结束，判断job是否成功。

```scala
def submitJob[T, U](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    callSite: CallSite,
    resultHandler: (Int, U) => Unit,
    properties: Properties): JobWaiter[U] = {
  // Check to make sure we are not launching a task on a partition that does not exist.
  val maxPartitions = rdd.partitions.length
  partitions.find(p => p >= maxPartitions || p < 0).foreach { p =>
    throw new IllegalArgumentException(
      "Attempting to access a non-existent partition: " + p + ". " +
        "Total number of partitions: " + maxPartitions)
  }

  // SPARK-23626: `RDD.getPartitions()` can be slow, so we eagerly compute
  // `.partitions` on every RDD in the DAG to ensure that `getPartitions()`
  // is evaluated outside of the DAGScheduler's single-threaded event loop:
  eagerlyComputePartitionsForRddAndAncestors(rdd)

  val jobId = nextJobId.getAndIncrement()
  if (partitions.isEmpty) {
    val clonedProperties = Utils.cloneProperties(properties)
    if (sc.getLocalProperty(SparkContext.SPARK_JOB_DESCRIPTION) == null) {
      clonedProperties.setProperty(SparkContext.SPARK_JOB_DESCRIPTION, callSite.shortForm)
    }
    val time = clock.getTimeMillis()
    listenerBus.post(
      SparkListenerJobStart(jobId, time, Seq.empty, clonedProperties))
    listenerBus.post(
      SparkListenerJobEnd(jobId, time, JobSucceeded))
    // Return immediately if the job is running 0 tasks
    return new JobWaiter[U](this, jobId, 0, resultHandler)
  }

  assert(partitions.nonEmpty)
  val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
  val waiter = new JobWaiter[U](this, jobId, partitions.size, resultHandler)
  eventProcessLoop.post(JobSubmitted(
    jobId, rdd, func2, partitions.toArray, callSite, waiter,
    JobArtifactSet.getActiveOrDefault(sc),
    Utils.cloneProperties(properties)))
  waiter
}
```

submitJob会计算DAG中每个RDD的`.partitions`，确保`getPartitinos`在`DAGScheduler`单线程事件循环外计算，避免`RDD.getPartitions`的计算影响调度效率。

```scala
private def eagerlyComputePartitionsForRddAndAncestors(rdd: RDD[_]): Unit = {
  val startTime = System.nanoTime
  val visitedRdds = new HashSet[RDD[_]]
  // We are manually maintaining a stack here to prevent StackOverflowError
  // caused by recursively visiting
  val waitingForVisit = new ListBuffer[RDD[_]]
  waitingForVisit += rdd

  def visit(rdd: RDD[_]): Unit = {
    if (!visitedRdds(rdd)) {
      visitedRdds += rdd

      // Eagerly compute:
      rdd.partitions

      for (dep <- rdd.dependencies) {
        waitingForVisit.prepend(dep.rdd)
      }
    }
  }

  while (waitingForVisit.nonEmpty) {
    visit(waitingForVisit.remove(0))
  }
  logDebug("eagerlyComputePartitionsForRddAndAncestors for RDD %d took %f seconds"
    .format(rdd.id, (System.nanoTime - startTime) / 1e9))
}
```

`eagerlyComputePartitionsForRddAndAncestors`函数对DAG中每个RDD调用`partitions`生成分区。为了避免StackOverFlowError，没有直接采用递归的方式遍历DAG，而是采用手动维护栈的方式遍历DAG。维护已经遍历的RDD的Set和还没有遍历过的RDD的列表，每次从列表中取出一个RDD，判断是否已经遍历过，如果已经遍历过，则忽略，否则计算`partitions`，并且将它依赖的RDD也加入到列表中，直到列表为空，所有RDD都已经遍历结束。

### getPartitions

```scala
final def partitions: Array[Partition] = {
  checkpointRDD.map(_.partitions).getOrElse {
    if (partitions_ == null) {
      stateLock.synchronized {
        if (partitions_ == null) {
          partitions_ = getPartitions
          partitions_.zipWithIndex.foreach { case (partition, index) =>
            require(partition.index == index,
              s"partitions($index).partition == ${partition.index}, but it should equal $index")
          }
        }
      }
    }
    partitions_
  }
}
```

`getPartitions`是RDD中的虚方法，由RDD子类负责实现。

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



```scala
private[spark] class ParallelCollectionRDD[T: ClassTag](
  sc: SparkContext,
  @transient private val data: Seq[T],
  numSlices: Int,
  locationPrefs: Map[Int, Seq[String]])
  extends RDD[T](sc, Nil) {
// TODO: Right now, each split sends along its full data, even if later down the RDD chain it gets
// cached. It might be worthwhile to write the data to a file in the DFS and read it in the split
// instead.
// UPDATE: A parallel collection can be checkpointed to HDFS, which achieves this goal.

override def getPartitions: Array[Partition] = {
  val slices = ParallelCollectionRDD.slice(data, numSlices).toArray
  slices.indices.map(i => new ParallelCollectionPartition(id, i, slices(i))).toArray
}
```

ParallelCollectionRDD的getPartitions函数首先将输入的数据分成numSlices份，然后生成对应的分区。

```scala
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false,
    isFromBarrier: Boolean = false,
    isOrderSensitive: Boolean = false)
  extends RDD[U](prev) {
  override def getPartitions: Array[Partition] = firstParent[T].partitions
```

MapPartitionsRDD直接继承父RDD的分区。

```scala
@DeveloperApi
class UnionRDD[T: ClassTag](
    sc: SparkContext,
    var rdds: Seq[RDD[T]])
  extends RDD[T](sc, Nil) {  // Nil since we implement getDependencies

  // visible for testing
  private[spark] val isPartitionListingParallel: Boolean =
    rdds.length > conf.get(RDD_PARALLEL_LISTING_THRESHOLD)

  override def getPartitions: Array[Partition] = {
    val parRDDs = if (isPartitionListingParallel) {
      // scalastyle:off parvector
      val parArray = new ParVector(rdds.toVector)
      parArray.tasksupport = UnionRDD.partitionEvalTaskSupport
      // scalastyle:on parvector
      parArray
    } else {
      rdds
    }
    val array = new Array[Partition](parRDDs.iterator.map(_.partitions.length).sum)
    var pos = 0
    for ((rdd, rddIndex) <- rdds.zipWithIndex; split <- rdd.partitions) {
      array(pos) = new UnionPartition(pos, rdd, rddIndex, split.index)
      pos += 1
    }
    array
  }
```

UnionRDD类似于MapPartitionsRDD，分区依据父RDD的分区生成。依次遍历每个父RDD的每个分区，生成对应的UnionRDD的分区。

```scala
@DeveloperApi
class ShuffledRDD[K: ClassTag, V: ClassTag, C: ClassTag](
    @transient var prev: RDD[_ <: Product2[K, V]],
    part: Partitioner)
  extends RDD[(K, C)](prev.context, Nil) {
  override def getPartitions: Array[Partition] = {
    Array.tabulate[Partition](part.numPartitions)(i => new ShuffledRDDPartition(i))
  }
private[spark] class ShuffledRDDPartition(val idx: Int) extends Partition {
  override val index: Int = idx
}
```

ShuffledRDD的分区总数通过`partitioner.numPartitions`得到，生成的分区为ShuffledRDDPartition。

```scala
@DeveloperApi
class CoGroupedRDD[K: ClassTag](
    @transient var rdds: Seq[RDD[_ <: Product2[K, _]]],
    part: Partitioner)
  extends RDD[(K, Array[Iterable[_]])](rdds.head.context, Nil) {
    
  override def getPartitions: Array[Partition] = {
    val array = new Array[Partition](part.numPartitions)
    for (i <- array.indices) {
      // Each CoGroupPartition will have a dependency per contributing RDD
      array(i) = new CoGroupPartition(i, rdds.zipWithIndex.map { case (rdd, j) =>
        // Assume each RDD contributed a single dependency, and get it
        dependencies(j) match {
          case s: ShuffleDependency[_, _, _] =>
            None
          case _ =>
            Some(new NarrowCoGroupSplitDep(rdd, i, rdd.partitions(i)))
        }
      }.toArray)
    }
    array
  }
```

CoGroupedRDD通过`numPartitions`获取到当前rdd的分区个数，生成CoGroupPartition分区。当前RDD可能有很多父RDD，如果RDD之间的依赖关系为窄依赖，则记录依赖的父RDD中的对应分区信息，如果是宽依赖，则忽略。每个分区都保存自己依赖的父RDD的对应窄依赖分区。

### Stage

