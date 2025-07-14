---
title: "Spark内存管理"
author: "爱吃芒果"
description: 
date: 2025-03-09T22:32:09+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
  - "Spark"
  - "内存管理"
categories:
  - "Spark"
  - "内存管理"
---

## 关键问题

1. 内存被分成哪些区域，各分区之间的关系是什么，通过什么参数控制

2. 内存上报和释放的单位是什么，上报和释放是如何实现的

3. 如何避免内存没有释放导致资源泄露

4. 如何避免重复上报和漏上报问题

5. 对象的生命周期和内存上报释放之间的关系

6. 哪些对象会被上报，为什么选择这些对象上报

7. 内存上报是否持有对象引用

## 静态内存管理模型

Spark早期版本（Spark 1.6之前的版本）使用静态内存管理模型（StaticfMemoryManager）将内存空间划分为3个分区：

1. 数据缓存空间（Storage Memory）：约占60%的内存空间，用于存储RDD缓存数据、广播数据、task的一些计算结果等
2. 框架执行空间（Execution Memory）：约占20%的内存空间，用于存储Shuffle机制中的中间数据
3. 用户代码空间（User Memory）：约占20%的内存空间，用于存储用户代码的中间计算结果、Spark框架自身产生的内部对象，以及Eexecutor JVM自身的一些内存对象等

## 统一内存管理模型

Executor JVM的整个内存空间划分为以下3个部分

1. 系统保留内存（Reserved Memory) 系统保留内存使用较小的空间存储Spark框架产生的内部对象（如Spark Executor对象，TaskMemoryManager对象等Spark内部对象），系统保留内存大小通过spark.testing.ReservedMemory默认设置为300MB
2. 用户代码空间（User Memory）用户代码空间被用于存储用户代码生成的对象，如map中用户自定义的数据结构，用户代码空间模拟热约为40%的内存空间
3. 框架内存空间 （Frameworke Memory）框架缓存空间包括框架执行空间（Execution Memory）和数据缓存空间（Storage Memory）。总大小为spark.memory.fraction (default 0.6) * (heap - ReservedMemory)，约等于60%的内存空间。两者共享这个空间，其中一方空间不足时可以动态向另一方借用。具体地，当数据缓存空间不足时，可以向框架执行空间借用其空闲空间，后续当框架执行需要更多空间时，数据缓存空间需要归还借用的空间，这时候Spark可能将部分缓存数据移除内存来归还空间。同样，当框架执行空间不足时，可以向数据缓存空间借用空间，但至少要保证数据缓存空间具有约50%左右（spark.memory.storageFraction (default 0.5) * Framework memory) 的空间。在框架执行时借走的空间不会归还给数据缓存空间，愿意是难以代码实现。

如果用户定义了堆外内存，其大小通过spark.memory.offHeap.size设置，那么Spark仍然会按照堆内内存使用的spark.memory.storageFraction将堆外内存分为框架执行空间和数据缓存空间，而且堆外内存的管理方式和功能与堆外内存的Framework Memory一样。在运行应用时，Spark会根据应用的Shuffle方式及用户设定的数据缓存级别来决定使用堆外内存还是堆外内存。如SerializedShuffle方法可以利用堆外内存来进行Shuffle Write，用户使用rdd.persist(OFF_HEAP)可以将rdd存储到堆外内存。

由于Executor中存在多个task，因此框架执行空间实际上是由多个task（ShuffleMapTask或ResultTask）共享的。在运行过程中，Executor中活跃的task数目在[0, #ExecutorCores]内变化，#ExecutorCores表示为每个Executor分配的cpu个数。为了公平性，每个task可使用的内存空间被均分，也就是空间大小被控制在[1/2N, 1/N] * ExecutorMemory内，N是当前活跃的task数目。假设一个Executor中最初有4个活跃的task，且只使用堆内内存，那么每个task最多可以占用1/4的On-heap Execution Memory，当其中2个task完成而又新加入4个task后，活跃task变为6个，那么后加入的每个task最多使用1/6的On-heap Execution Memory，这个策略也适用于堆外内存的Execution Memory。

这里重点介绍Shuffle机制中的Serialized Shuffle，Serialized Shuffle用来不需要map端聚合、不需要按照Key进行排序，且分区个数较大的情形，将record对象序列化后存放到可分页存储的数组中，序列化可以减少存储开销，分页可以利用不连续的空间。首先将新来的<K, V> record序列化写入一个1MB的缓冲区（serBuffer），然后将serBuffer中序列化的record放到ShuffleExternalSorter的Page中进行排序。插入和排序的方法是，首先分配一个LongArray来保存record的指针，指针为64位，前24位存储record的partitionId，中间13为存储record所在的Page Num，后27位存储record在该Page中的偏移量。也就是说LongArray最多可以管理1TB的内存，随着record不断地插入Page中，如果LongArray不够用或Page不够用，则会通过allocatePage向TaskMemoryManager申请，如果申请不到，就启动spill程序，将中间结果spill到磁盘上，最后再由UnsafeShuffleWriter进行统一的merge。Page由TaskMemoryManager管理和分配，可以存放在堆内内存或者堆外内存。

数据缓存空间主要用于存放3种数据：RDD缓存数据（RDD partition）、广播数据（Broadcast data），以及task的计算结果（TaskResult）。

Broadcast默认使用类似BT下载的TorrentBroadcast方式，需要广播的数据一般预先存储在Driver端，Spark在Driver端将要广播的数据划分大小为spark.Broadcast.blockSize = 4MB的数据块（block），然后赋予每个数据块一个blockId为BroadcastblockId(id, "piece" + i) ，id表示block的编号，piece表示被划分后的第几个block。之后使用类似BT的方式将每个block广播到每个Executor中，Executor接收到每个block数据块后，将其放到堆内的数据缓存空间的ChunkedByteBuffer里面，缓存模式为MEMORY_AND_DISK_SER，因此，这里的ChunkedByteBuffer构造与MEMORY_ONLY_SER模式中的一样，都是用不连续的空间来存储序列化数据。

许多应用需要在Driver端收集task的计算结果并进行处理，如调用了rdd.collect的应用，当task的输出结果大小超过spark.task.maxDirectResultSize = 1MB且小于1GB使，需要先将每个task的输出结果缓存到执行该task的Executor中，存放模式是MEMORY_AND_DISK_SER，然后Executor将task的输出结果发送到Driver端进一步处理。

目前，针对RDD操作，Spark只提供了Serialized Shuffle Writer方式，没有提供Serialized Shuffle Read方式，实际上， 在SparkSQL项目中，Spark利用SQL操作的特点（如SUM、AVG计算结果的等宽性），提供了更多的Serialized Shuffle方式，直接在序列化的数据上实现聚合等计算，详情可以参考UnsafeFixedAggregationMap、ObjectAggregationMap等数据结构的实现。



## 源码分析

### MemoryBlock

MemoryBlock表示一段连续的内存空间，类似于操作系统中page的概念。

MemoryBlock继承自**MemoryLocation**，当追踪堆外分配时，obj为空，offset表示堆外内存地址，当追踪堆内内存分配时，obj为对象引用，offset为对象内偏移量，可以看到MemoryLocation只是记录了对象的位置信息，没有记录对象内存占用的信息。

```java

public class MemoryLocation {
  @Nullable
  Object obj;
  long offset;
```

```java
public class MemoryBlock extends MemoryLocation {
	  private final long length;
  	public int pageNumber = NO_PAGE_NUMBER;
```

MemoryBlock新增两个字段，length表示page的大小，pageNumber很好理解，TaskMemoryManager会给每个页分配一个页号，有以下几种特殊情况

1. **NO_PAGE_NUMBER** 表示没有被TaskMemoryManager分配，初始值
2. **FREED_IN_TMM_PAGE_NUMBER** 表示被TaskMemoryManager释放，`TaskMemoryManager.free`操作中会将页号设置为此值，`MemoryAllocator.free`遇到没有被TaskMemoryMananger释放的页时，会报错
3. **FREED_IN_ALLOCATOR_PAGE_NUMBER** 被MemoryAllocator释放，可以检测多次释放

### MemoryAllocator

MemoryAllocator接口定义了申请和释放MemoryBlock的方法，**HeapMemoryAllocator**和**UnsafeMemoryAllocator**分别实现了堆内和堆外的内存分配器。

```java
public interface MemoryAllocator {

  /**
   * Allocates a contiguous block of memory. Note that the allocated memory is not guaranteed
   * to be zeroed out (call `fill(0)` on the result if this is necessary).
   */
  MemoryBlock allocate(long size) throws OutOfMemoryError;

  void free(MemoryBlock memory);

  MemoryAllocator UNSAFE = new UnsafeMemoryAllocator();

  MemoryAllocator HEAP = new HeapMemoryAllocator();
}
```

#### HeapMemoryAllocator

```java
public class HeapMemoryAllocator implements MemoryAllocator {

  @GuardedBy("this")
  private final Map<Long, LinkedList<WeakReference<long[]>>> bufferPoolsBySize = new HashMap<>();

  private static final int POOLING_THRESHOLD_BYTES = 1024 * 1024;
```

可以看到实际分配的对象就是long数组，并且做了池化，对于1MB以上的内存尝试放入池中，这里没有限制池的大小，持有的是long数组的弱引用，减少频繁申请和释放大内存造成的开销。

如果申请不到内存，会抛出`OutOfMemoryError`

#### UnsafeMemoryAllocator

实现没有什么特殊的地方，直接调用Spark包装过的`Unsafe` API，直接调用Unsafe包中的API，所以不受`MaxDirectMemorySize`的控制

```java
public long allocateMemory(long bytes) {
  beforeMemoryAccess();
  return theInternalUnsafe.allocateMemory(bytes);
}
```

### MemoryManager

MemoryManager抽象类负责管理内存，在计算和存储之间共享内存，计算内存指在shuffles, joins, sorts and aggregations 中计算过程所使用的内存，而存储内存指被用于缓存或者在集群中传播内部数据所占用的内存，每个JVM只有一个MemoryManager。

#### Spark内存参数

`spark.memory.offHeap.enabled` 如果开启，某些计算将使用堆外内存，要求`spark.memory.offHeap.size`必须为正数，默认关闭

`spark.memory.fraction` (堆内存 - 300MB)被用于计算和存储的比例，这个值越低，吐磁盘以及缓存驱逐发生的越频繁，这个设置的主要目的是留出空间给用户数据结构以及比如稀疏、不寻常的大内存记录导致的内存估算不准确。默认值为0.6

`spark.memory.offHeap.size`指定了spark堆外使用的内存大小

`saprk.memory.storageFraction`免于驱逐的存储内存占用内存大小，这里表示为`spark.memory.fraction`留出的内存的百分比。默认为0.5

堆外内存由`spark.memory.offHeap.size`规定，堆外存储内存为$spark.memory.offHeap.size * spark.memory.storageFractioin$，剩余的内存为堆外计算内存。

#### 主要字段和方法

```java
@GuardedBy("this")
protected val onHeapStorageMemoryPool = new StorageMemoryPool(this, MemoryMode.ON_HEAP)
@GuardedBy("this")
protected val offHeapStorageMemoryPool = new StorageMemoryPool(this, MemoryMode.OFF_HEAP)
@GuardedBy("this")
protected val onHeapExecutionMemoryPool = new ExecutionMemoryPool(this, MemoryMode.ON_HEAP)
@GuardedBy("this")
protected val offHeapExecutionMemoryPool = new ExecutionMemoryPool(this, MemoryMode.OFF_HEAP)
```

`StorageMemoryPool`实际管理存储内存，`ExecutionMemoryPool`实际管理计算内存，这两者在处理关键操作是都需要持有`MemoryManager`对象锁，从而实现在存储和计算之间共享内存的操作。

- `acquireStorageMemory`:获得存储内存用来缓存block等
- `acquireUnrollMemory`: 获取展开内存用来展开给定的block
- `acquireExecutionMemory`: 获得计算内存，调用可能阻塞，确保每个任务至少有机会获得$1/ 2N$内存池大小，N表示当前活跃任务数量，比如老的任务已经占用了很多内存而任务数增加  
- `releaseExecutionMemory` 释放计算内存
- `releaseAllExecutionMemoryForTask` 释放当前任务的所有计算内存
- `releaseStorageMemory` 释放存储内存
- `releaseAllStorageMemory` 释放所有存储内存
- `releaseUnrollMemory` 释放展开内存

### UnifiedMemoryManager

堆内的内存总量由`spark.testing.memory`指定，默认为jvm堆大小，保留内存为300MB
$$
可用于存储或者计算的内存 = (spark.testing.memory - reserved\ memory) * spark.memory.fraction
$$
初始存储内存大小占比由`spark.memory.storageFraction`指定，存储和计算可以相互借用对方的内存，遵循以下规则：

- 如果计算内存不足，可最多可以让存储将占用超过初始存储内存大小的空间返还给计算内存
- 如果存储空间不足，可以借用计算内存的多余空间

`acquireExecutionMemory`: 实际调用`executionPool.acquireMemory`，依赖于回调函数`maybeGrowExecutionPool`和`computeMaxExecutionPoolSize`，前者可能将部分存储内存转移到计算内存，后者计算当前情况下最大计算内存，等于可用内存减去存储内存当前占用和存储内存初始大小的最小值。

`acquiredStorageMemory`: 如果存储内存空间不足，则尝试借用部分计算内存空间，最后调用`storagePool.acquireMemory`实际执行操作

`acquireUnrollMemory`: 实际调用`acquiredStorageMemory`

### MemoryPool

管理一块可以调整大小的内存区域的内部状态和使用记录。

#### ExecutionMemoryPool

字段`memoryForTask`记录了每个task id (long)对应的内存消耗(long)。

每个任务最少可以占用 $1 / 2N * poolSize$，而每个任务最多占用$1 / N * maxPoolsize$

`acquireMemory`:

1. 如果是新的任务，加入memoryForTask，并且通知所有等待获取计算内存的任务，当前任务数增加
2. 循环，直到任务占用超过了上限1/N，或者有空闲内存，以下步骤均在循环体来
3. 调用`maybeGrowPool`尝试从存储空间获取内存
4. 计算每个任务的最少内存占用和最高内存占用
5. 如果获得的计算内存加上当前内存占用低于最少内存占用，则等待通知
6. 否则更新状态，并返回获取到的内存大小

`releaseMemory`: 释放内存，如果释放后当前内存为0，则移除当前任务，只要释放内存，则通知在`acquiredMemory`等待的任务内存已经释放

#### StorageMemoryPool

`acquireMemory`: 如果存储空间不足，则调用`memoryStore.evictBlocksToFreeSpace`释放部分空间，判断需要的内存大小是否小于等于当前空闲内存

`releaseMemory`: 释放内存

`freeSpaceToShrinkPool`: 释放内存来减少存储空间的占用，必要时调用`memoryStore.evictBlocksToFreeSpace`驱逐block

### TaskMemoryManager

#### 内存地址编码

当需要将一个`int`或者`long`之类的元素插入到数组或者堆外的指定位置时

对于堆内，需要知道数据的引用以及偏移量，在TaskMemoryManager中保存了pageNumber和MemoryBlock的映射，而MemoryBlock保存了对象的引用，所以使用64位编码内存地址时，前13位用来储存pageNumber，后51位用来存储数组中的偏移量。（对象的地址会由于gc的原因而变动，所以不能直接使用对象地址）

对于堆外，需要知道申请到堆外内存的起始地址和偏移量，依然使用前13位存储pageNumber，使用后51位存储偏移量。这里如果直接使用内存地址，则不能知道对应的page是那个，当使用前13位储存pageNumber后，后51位显然不能储存内存的绝对地址，而应该存储内存相对于起始地址的偏移量。

#### 主要字段作用

- pageTable: 页表，保存pageNumber到MemoryBlock的映射，MemoryBlock[PAGE_TABLE_SIZE]
- memoryManager: TaskMemoryManager共享MemoryManager的内存资源
- taskAttemptId: task Id
- tungtenMemoryMode: 使用堆内还是堆外内存，和MemoryManger保持一致
- consumers：内存消费者，支持吐磁盘，`HashSet<MemoryConsumer>`
- acquiredButNotUsed: 向内存管理框架申请内存成功，但实际申请内存时发生OOM，认为MemoryManager可能高估了实际的可用内存，将这部分内存配额保存在此字段，方便后续触发吐磁盘，long
- currentOffHeapMemory: 任务当前堆外内存占用，long
- currentOnHeapMemory：任务当前堆内内存占用，long
- peakOffHeapMemory：任务最高堆外内存占用，long
- peakOnHeapMemory：任务最高堆内内存占用，long

#### 主要方法

**acquireExecutionMemory**为指定的MemoryConsumer获取内存，如果没有足够的内存，触发吐磁盘释放内存，返回成功获得的计算内存(<=N)。

```java
public long acquireExecutionMemory(long required, MemoryConsumer requestingConsumer) {
```

- 首先调用`MemoryManager.acquireExecutionMemory`尝试获取计算内存
- 如果获取到足够的内存，则跳过吐磁盘逻辑
- 如果没有获取到足够的内存，尝试吐磁盘释放内存，并尝试获取计算内存
  - 吐磁盘有两个优化的目标：
    1. 最小化吐磁盘调用的次数，减少吐磁盘文件的数量并且避免小的吐磁盘文件
    2. 避免吐磁盘释放内存超过所需，如果我们只是想要一丁点内存，不希望尽可能多的吐磁盘，很多内存消费者吐磁盘时会释放比请求多的内存
  - 所以这里采用一种启发式的算法，选择内存占用超过所需内存的MemoryConsumer中最小的MemoryConsumer来平衡这些因素，当只有少量大内存请求时，这种方法效率很好，但如果场景中有大量小内存请求，这种方法会导致产生大量小的spill文件
  - 具体实现，将所有的MemoryConsumer放入一个`TreeMap`中，根据内存占用排序，如果是当前MemoryConsumer，则认为内存占用为0，这样当前MemoryConsumer被spill的优先级最低。
    然后选择内存占用超过所需内存的MemoryConsumer中最小的MemoryConsumer进行吐磁盘操作并且尝试获取计算内存，如果没有符合这一条件的MemoryConsumer，则直接选择内存占用最大的MemoryCosumer进行吐磁盘并尝试获取计算内存`trySpillAndAcquire`。
    如果获取到的内存依然不满足需求，则继续吐磁盘流程，选择下一个MemoryConsumer，重复上述流程。

- 最终不管是否获取到了所需的内存，都将`MemoryConsumer`加入consumers中，并更新当前和最高的任务内存占用

**trySpillAndAcquire**对选中的MemoryConsumer执行吐磁盘操作释放内存，并尝试获取所需的计算内存

```java
 * @return number of bytes acquired (<= requested)
 * @throws RuntimeException if task is interrupted
 * @throws SparkOutOfMemoryError if an IOException occurs during spilling
 */
private long trySpillAndAcquire(MemoryConsumer requestingConsumer, long requested, List<MemoryConsumer> cList, int idx)

```

- 首先调用`MemoryConsumer#spill`方法尝试释放内存，如果释放内存为0，则直接返回0
- 如果释放内存大于0，调用`MemoryManager#acquireExecutionMemory`尝试获取计算内存，这里需要注意，吐磁盘释放的内存会被所有任务公平竞争，所以可能无法获取到这次吐磁盘释放的所有内存，需要在下一次循环中继续尝试吐磁盘
- 两种异常场景，当任务被中断时，抛出`RuntimeException`，吐磁盘遇到`IOException`时，抛出`SparkOutOfMemoryError`

**releaseExecutionMemory** 为一个MemoryConsumer释放N字节的计算内存，实际调用了`MemoryManager#releaseExecutionMemory`，并更新当前内存占用

**showMemoryUsage** dump所有Consumer的内存占用

**allocatePage** 分配内存，并更新页表，该操作旨在为多个算子之间共享的大块内存分配空间

```java
public MemoryBlock allocatePage(long size, MemoryConsumer consumer) 
```

- 首先调用`TaskMemoryManager#acquiredExectionMemory`获取计算内存，如果没有获取到内存，则返回null
- 然后通过`MemoryManager#tungstenMemoryAllocator#allocate`实际申请内存，如果遇到`OutOfMemoryError`，则认为实际上没有足够多的内存，实际的空闲内存要比MemoryManager认为的少一些，所以将从内存管理框架中获得的内存配额添加到`acquiredButNotUsed`字段中，并再次调用当前函数，这次将触发吐磁盘操作释放内存（p.s. 感觉处理OutOfMemoryError的意义不大，OutOfMeomryError发生时应该直接结束程序，因为程序已经进入了异常状态，无法预料OutOfMemoryError对程序的影响）
- 如果成功获取到内存，则需要更新页表，并返回对应的页，其实就是MemoryBlock

**freePage**释放页占用的内存，更新pageNumber为FREED_IN_TMM_PAGE_NUMBER，清理页表，调用`MemoryManager.tunstenMemoryAllocator#free`实际释放内存，调用`releaseExecutionMemory`释放内存管理框架对应的内存配额。

似乎用逻辑内存指代内存管理框架中的内存配额，而用物理内存指代实际的内存更加好一些 

```java
public void freePage(MemoryBlock page, MemoryConsumer consumer) {
```

**cleanUpAllAllocatedMemory**清理所有申请的内存和页

- 调用`MemoryManager#tungstenMemoryAllocator#free`释放每个页的内存
- 调用`MemoryManager#releaseExectionMemory`释放`acquiredButNotUsed`内存
- 调用`MemoryManager#ReleaseAllExecutionMemoryForTask`释放任务的所有计算内存，并返回释放的内存大小，非0值可以用来检测内存泄露

## 参考资料

1. [Deep Dive into Spark Memory Management](https://luminousmen.com/post/dive-into-spark-memory/)
2. [Apache Spark Memory Management: Deep Dive](https://www.linkedin.com/pulse/apache-spark-memory-management-deep-dive-deepak-rajak)
