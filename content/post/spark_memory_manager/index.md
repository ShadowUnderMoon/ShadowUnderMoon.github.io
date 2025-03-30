---
title: "Spark内存管理"
description: 
date: 2025-03-09T22:32:09+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
---

## 关键问题

1. 内存被分成哪些区域，各分区之间的关系是什么，通过什么参数控制

2. 内存上报和释放的单位是什么，上报和释放是如何实现的

3. 如何避免内存没有释放导致资源泄露

4. 如何避免重复上报和漏上报问题

5. 对象的生命周期和内存上报释放之间的关系

6. 哪些对象会被上报，为什么选择这些对象上报

7. 内存上报是否持有对象引用



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

### TaskMemoryManager

#### 内存地址编码

当需要将一个`int`或者`long`之类的元素插入到数组或者堆外的指定位置时

todo: MemoryBlock offset

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







所以将

acquireExecutionMemory 是从内存管理框架中获取内存

MemoryAllocator.allocate 是实际获取内存，如果内存超限，说明内存管理框架任务有内存，但实际上没有内存，将这部分内存固定住，这里好奇怪，如果taskmananger关闭，这部分内存又会回到吗



## 参考资料

1. [Deep Dive into Spark Memory Management](https://luminousmen.com/post/dive-into-spark-memory/)
2. [Apache Spark Memory Management: Deep Dive](https://www.linkedin.com/pulse/apache-spark-memory-management-deep-dive-deepak-rajak)
