# Spark Job执行流程


```java
// CoarseGrainedExecutorBackend
case LaunchTask(data) =>
  if (executor == null) {
    exitExecutor(1, "Received LaunchTask command but executor was null")
  } else {
    val taskDesc = TaskDescription.decode(data.value)
    logInfo(log"Got assigned task ${MDC(LogKeys.TASK_ID, taskDesc.taskId)}")
    executor.launchTask(this, taskDesc)
  }
```

接收到Drive端传来的task，反序列化后，启动task

```java
// Executor
def launchTask(context: ExecutorBackend, taskDescription: TaskDescription): Unit = {
  val taskId = taskDescription.taskId
  val tr = createTaskRunner(context, taskDescription)
  runningTasks.put(taskId, tr)
  val killMark = killMarks.get(taskId)
  if (killMark != null) {
    tr.kill(killMark._1, killMark._2)
    killMarks.remove(taskId)
  }
  threadPool.execute(tr)
  if (decommissioned) {
    log.error(s"Launching a task while in decommissioned state.")
  }
}
```

创建TaskRunner，在线程池中开始执行

```java
private[executor] val threadPool = {
  val threadFactory = new ThreadFactoryBuilder()
    .setDaemon(true)
    .setNameFormat("Executor task launch worker-%d")
    .setThreadFactory((r: Runnable) => new UninterruptibleThread(r, "unused"))
    .build()
  Executors.newCachedThreadPool(threadFactory).asInstanceOf[ThreadPoolExecutor]
}
```

底层的线程池是一个cache thread pool，没有限制数量

最后调用`Task.runTask`实际执行task，ShuffleMapTask和ResultTask都重写了这个方法。

```scala
// ResultTask
override def runTask(context: TaskContext): U = {
  // Deserialize the RDD and the func using the broadcast variables.
  val threadMXBean = ManagementFactory.getThreadMXBean
  val deserializeStartTimeNs = System.nanoTime()
  val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime
  } else 0L
  val ser = SparkEnv.get.closureSerializer.newInstance()
  // 反序列化获得RDD和func
  val (rdd, func) = ser.deserialize[(RDD[T], (TaskContext, Iterator[T]) => U)](
    ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
  _executorDeserializeTimeNs = System.nanoTime() - deserializeStartTimeNs
  _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
  } else 0L
	// 调用func计算结果
  func(context, rdd.iterator(partition, context))
}
```

`ResultTask.runTask`反序列化获得RDD和func后，调用`func`函数利用RDD对应分区的数据计算task的结果。

```scala
final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
  if (storageLevel != StorageLevel.NONE) {
    getOrCompute(split, context)
  } else {
    computeOrReadCheckpoint(split, context)
  }
}
```

`RDD.iterator`方法会检查分区是否被缓存或者checkpoint，如果有，则不需要重新计算，否则需要重新计算。

```scala
class StorageLevel private(
    private var _useDisk: Boolean,
    private var _useMemory: Boolean,
    private var _useOffHeap: Boolean,
    private var _deserialized: Boolean,
    private var _replication: Int = 1)
  extends Externalizable {
 object StorageLevel {
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
  val DISK_ONLY_3 = new StorageLevel(true, false, false, false, 3)
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(true, true, true, false, 1)
```

`StorageLevel`控制了RDD的存储，每种StorageLevel记录了是否使用内存、是否使用磁盘、是否使用堆外、是否序列化对象、是否在不同节点间复制RDD。

```scala
private def persist(newLevel: StorageLevel, allowOverride: Boolean): this.type = {
  // TODO: Handle changes of StorageLevel
  if (storageLevel != StorageLevel.NONE && newLevel != storageLevel && !allowOverride) {
    throw SparkCoreErrors.cannotChangeStorageLevelError()
  }
  // If this is the first time this RDD is marked for persisting, register it
  // with the SparkContext for cleanups and accounting. Do this only once.
  if (storageLevel == StorageLevel.NONE) {
    // 这里使用weakref
    sc.cleaner.foreach(_.registerRDDForCleanup(this))
    sc.persistRDD(this)
  }
  storageLevel = newLevel
  this
}
def persist(newLevel: StorageLevel): this.type = {
  if (isLocallyCheckpointed) {
    // This means the user previously called localCheckpoint(), which should have already
    // marked this RDD for persisting. Here we should override the old storage level with
    // one that is explicitly requested by the user (after adapting it to use disk).
    persist(LocalRDDCheckpointData.transformStorageLevel(newLevel), allowOverride = true)
  } else {
    persist(newLevel, allowOverride = false)
  }
}
def persist(): this.type = persist(StorageLevel.MEMORY_ONLY)
def cache(): this.type = persist()
def unpersist(blocking: Boolean = false): this.type = {
  if (isLocallyCheckpointed) {
    // This means its lineage has been truncated and cannot be recomputed once unpersisted.
    logWarning(log"RDD ${MDC(RDD_ID, id)} was locally checkpointed, its lineage has been" +
      log" truncated and cannot be recomputed after unpersisting")
  }
  logInfo(log"Removing RDD ${MDC(RDD_ID, id)} from persistence list")
  sc.unpersistRDD(id, blocking)
  storageLevel = StorageLevel.NONE
  this
}
```

storagelevel可以通过`persist`和`unpersit`调用修改，可以缓存RDD到内存、磁盘中。

如果RDD需要计算，则调用`RDD.compute`

对于ResultTask，计算应该比较简单，比如map操作依赖的MapPartitionsRDD

```scala
private[spark] class MapPartitionsRDD[U: ClassTag, T: ClassTag](
    var prev: RDD[T],
    f: (TaskContext, Int, Iterator[T]) => Iterator[U],  // (TaskContext, partition index, iterator)
    preservesPartitioning: Boolean = false,
    isFromBarrier: Boolean = false,
    isOrderSensitive: Boolean = false)
  extends RDD[U](prev) {

  override val partitioner = if (preservesPartitioning) firstParent[T].partitioner else None

  override def getPartitions: Array[Partition] = firstParent[T].partitions

  override def compute(split: Partition, context: TaskContext): Iterator[U] =
    f(context, split.index, firstParent[T].iterator(split, context))
```

可以看到会递归调用`rdd.iterator`来计算最终结果，`rdd.iterator`相比于`rdd.compute`多了检查cache和checkpoint。

```scala
// ShuffleRDD
override def getDependencies: Seq[Dependency[_]] = {
  val serializer = userSpecifiedSerializer.getOrElse {
    val serializerManager = SparkEnv.get.serializerManager
    if (mapSideCombine) {
      serializerManager.getSerializer(implicitly[ClassTag[K]], implicitly[ClassTag[C]])
    } else {
      serializerManager.getSerializer(implicitly[ClassTag[K]], implicitly[ClassTag[V]])
    }
  }
  List(new ShuffleDependency(prev, part, serializer, keyOrdering, aggregator, mapSideCombine))
}
override def compute(split: Partition, context: TaskContext): Iterator[(K, C)] = {
  val dep = dependencies.head.asInstanceOf[ShuffleDependency[K, V, C]]
  val metrics = context.taskMetrics().createTempShuffleReadMetrics()
  SparkEnv.get.shuffleManager.getReader(
    dep.shuffleHandle, split.index, split.index + 1, context, metrics)
    .read()
    .asInstanceOf[Iterator[(K, C)]]
}
```

`ShuffleRDD.compute`会从ShuffleManager中获取reader读取shuffle数据

```scala
// ShuffleMapTask
override def runTask(context: TaskContext): MapStatus = {
  // Deserialize the RDD using the broadcast variable.
  val threadMXBean = ManagementFactory.getThreadMXBean
  val deserializeStartTimeNs = System.nanoTime()
  val deserializeStartCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime
  } else 0L
  val ser = SparkEnv.get.closureSerializer.newInstance()
  val rddAndDep = ser.deserialize[(RDD[_], ShuffleDependency[_, _, _])](
    ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
  _executorDeserializeTimeNs = System.nanoTime() - deserializeStartTimeNs
  _executorDeserializeCpuTime = if (threadMXBean.isCurrentThreadCpuTimeSupported) {
    threadMXBean.getCurrentThreadCpuTime - deserializeStartCpuTime
  } else 0L

  val rdd = rddAndDep._1
  val dep = rddAndDep._2
  // While we use the old shuffle fetch protocol, we use partitionId as mapId in the
  // ShuffleBlockId construction.
  val mapId = if (SparkEnv.get.conf.get(config.SHUFFLE_USE_OLD_FETCH_PROTOCOL)) {
    partitionId
  } else {
    context.taskAttemptId()
  }
  dep.shuffleWriterProcessor.write(
    rdd.iterator(partition, context),
    dep,
    mapId,
    partitionId,
    context)
}
```


