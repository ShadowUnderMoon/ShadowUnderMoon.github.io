---
title: "Spark Shuffle机制"
author: "爱吃芒果"
description:
date: "2025-06-01T09:18:40+08:00"
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

运行在不同stage、不同节点上的task见通过shuffle机制传递数据，shuffle解决的问题是如何将数据进行重新组织，使其能够在上游和下游task之间进行传递和计算。如果只是单纯的数据传递，则只需要将数据进行分区、通过网络传输即可，没有太大的难度，但shuffle机制还需要进行各种类型的计算（如聚合、排序），而且数据量一般会很大，如果支持这些不同类型的计算，如果提高shuffle的性能都是shuffle机制设计的难点。

shuffle机制分为shuffle write和shuffle read两个阶段，前者主要解决上游stage输出数据的分区问题，后者主要解决下游stage从上游stage获取数据、重新组织、并为后续操作提供数据的问题。

在shuffle过程中，我们将上游stage称为map stage，下游stage称为reduce stage，相应地，map stage包含多个map task，reduce stage包含多个reduce task。

分区个数和下游stage的task个数一致，分区个数可以通过用户自定义，如groupByKey(numPartitions)中的numPartitions一般被定义为集群中可用cpu个数的1~2倍，即将每个map task的输出数据划分为numPartitions份，相应地，在reduce stage中启动numPartition个task来获取并处理这些数据。如果用户没有自定义，则默认分区个数是parent RDD的分区个数的最大值。

对map task输出的每一个<K, V> recod，根据Key计算其partitionId，具有不同partitionId的record被输出到不同的分区（文件）中。

数据聚合的本质是将相同key的record放在一起，并进行必要的计算，这个过程可以利用HashMap实现。方法是使用两步聚合（two-phase aggregation），先将不同tasks获取到的<K, V> record存放到HashMap中，HashMap中的Key是K, Value是list(V)。然后，对于HashMap中每一个<K, list(V)> record，使用func计算得到<K, func(list(v))> record。两步聚合方案的优点是可以解决数据聚合问题，逻辑清晰、容易实现，缺点是所有shuffle的record都会先被存放到HashMap中，占用内存空间较大。另外，对于包含聚合函数的操作，如reduceByKey(func)，需要先将数据聚合到HashMap中以后再执行func()聚合函数，效率较低。

对于reduceByKey(func)等包含聚合函数的操作来说，我们可以采用一种在线聚合（Online aggregation）的方法来减少内存空间占用。该方法在每个record加入HashMap时，同时进行func()聚合操作，并更新相应的聚合结果。具体地，对于每一个新来的<K, V> record，首先从HashMap中get出已经存在的结果V' = HashMap.get(K)，然后执行聚合函数得到新的中间结果V'' = func(V, V')，最后将V''写入HashMap中，即HashMap.put(K, V'')。一般来说，聚合函数的执行结果会小于原始数据规模，即size(func(list(V))) < Size(list(V))，如sum(), max()等，所以在线聚合可以减少内存消耗。在线聚合将Shuffle Read和聚合函数计算耦合在一起，可以加速计算。但是，对于不包含聚合函数的操作，如groupByKey()等，在线聚合和两步聚合没有差别，因为这些操作不包含聚合函数，无法减少中间数据规模。

Shuffle Writer端的combine操作的目的是减少Shuffle的数据量，只有包含聚合函数的数据操作需要进行map段的combine，对于不包含聚合函数的操作，如groupByKey，我们即使进行了combine操作，也不能减少中间数据的规模。从本质上将，combine和Shuffle Read端的聚合过程没有区别，都是将<K, V> record聚合成<K, func(list(V))>，不同的是，Shuffle Read端聚合的是来自所有map task输出的数据，而combine聚合的是来自单一task输出的数据。因此仍然可以采用Shuffle Read端基于HashMap的解决方案。具体地，首先利用HashMap进行combine，然后对HashMap中每一个record进行分区，输出到对应的分区文件中。

如果需要排序，在Shuffle Read端必须必须执行sort，因为从每个task获取的数据组合起来以后不是全局按Key进行排序的。其次，理论上，在Shuffle Write端不需要进行排序，但如果进行了排序，那么Shuffle Read获取到（来自不同task）的数据是已经部分有序的数据，可以减少Shuffle Read端排序的复杂度。

根据排序和聚合的顺序，有三种方案可供选择：

第一种方案是先排序后聚合，这种方案需要先使用线性数据结果如Array，存在Shuffle Read的<K, V> record，然后对Key进行排序，排序后的数据可以直接从前到后进行扫描聚合，不需要再使用HashMap进行hash-based聚合。这种方案也是Hadoop MapReduce采用的方案，方案优点是既可以满足排序要求又可以满足聚合要求，缺点是需要较大内存来存储线性数据结构，同时排序和聚合过程不能同时进行，即不能使用在线聚合，效率较低。

第二种方案是排序和聚合同时进行，我们可以使用带有排序功能的Map，如TreeMap来对中间数据进行聚合，每次Shuffle Read获取到一个record，就将其放入TreeMap中与现有的record进行聚合，过程与HashMap类似，只有TreeMap自带排序功能。这种方案的优点是排序和聚合可以同时进行，缺点是相比HashMap，TreeMap的排序复杂度较高，TreeMap的插入时间复杂度为O(nlogn)，而且需要不断调整树的结果，不适合数据规模非常大的情况。

第三种方案是先聚合再排序，即维持现有基于HashMap的聚合方案不变，将HashMap中的record或record的引用放入线性数据结构中就行排序。这种方案的优点是聚合和排序过程独立，灵活性较高，而且之间的在线聚合方案不需要改动，缺点是需要复制（copy）数据或者引用，空间占用较大，Spark选择的是第三种方案，设计了特殊的HashMap来高效完成先聚合再排序的任务。

由于我们使用HashMap对数据进行combine和聚合，在数据量大的时候，会出现内存溢出，这个问题既可能出现在Shuffle Write阶段，也可能出现在Shuffle Read阶段。通过使用内存+磁盘混合存储来解决这个问题（吐磁盘），先在内存（如HashMap）中进行数据聚合，如果内存空间不足，则将内存中的数据spill到磁盘上，此时空闲出来的内存可以继续处理新的数据。此过程可以不断重复，直到数据处理完成。然而，问题是spill到磁盘上的数据实际上是部分聚合的结果，并没有和后续的数据进行过聚合。因此，为了得到完成的聚合结果，我们需要再进行下一步数据操作之前对磁盘上和内存中的数据进行再次聚合，这个过程我们称为全局聚合，为了加速全局聚合，我们需要将数据spill到磁盘上时进行排序，这样全局聚合才能够按照顺序读取spill到磁盘上的数据，并减少磁盘I/O。

## Spark中Shuffle框架的设计

### Shuffle Write框架设计和实现

在Shuffle Write阶段，数据操作需要分区、聚合和排序3个功能，Spark为了支持所有可能的情况，设计了一个通用的Shuffle Write框架，框架的计算顺序为`map()输出 --> 数据聚合 --> 排序 --> 分区输出`。map task每计算出一个record及其partitionId，就将record放入类似HashMap的数据结构中进行聚合，聚合完成后，再将HashMap中的数据放入类似Array的数据结构中进行排序，即可按照partitionId，也可以按照partitionId+Key进行排序，最后根据partitionId将数据写入不同的数据分区中，存放到本地磁盘上。其中聚合和排序过程是可选的，如果数据操作不需要聚合或者排序，那么可以去掉相应的聚合或排序过程。

#### 不需要map端聚合和排序

map依次输出<K, V> record并计算其partititionId, Spark根据partitionId，将record依次输出到不同的buffer中，每当buffer填满就将record溢写到磁盘中的分区文件中。分配buffer的原因是map输出record的速度很快，需要进行缓冲来减少磁盘I/O。在实现代码中，Spark将这种Shuffle Write的方式称为BypassMergeSortShuffleWriter，即不需要进行排序的Shuffle Write方式。

该模式的优缺点：优点是速度快，直接将record输出到不同的分区文件中。缺点是资源消耗过高，每个分区都需要一个buffer（大小由spark.Shuffle.file.buffer控制，默认为32KB），且同时需要建立多个分区文件进行溢写。当分区个数太大，如10000，每个map task需要月320MB的内存，会造成内存消耗过大，而且每个task需要同时建立和打开10000个文件，造成资源不足，因此，该shuffle方案适合分区个数较少的情况（< 200）。

该模式适用的操作类型：map端不需要聚合，key不需要排序且分区个数较少（<=spark.Shuffle.sort.bypassMergeThreshold，默认值为200），例如 groupByKey(100)、partitionBy(100)、sortByKey(100)等，注意sortByKey是在Shuffle Rread端进行排序。

#### 不需要map端聚合，但需要排序

在这种情况下需要按照partitionId+Key进行排序。Spark采用的实现方法是建立一个Array来存放map输出的record，并对Array中元素的Key进行精心设计，将每个<K, V> record转化为<(PID, K), V> record存储然后按照partitionId + Key对record进行排序，最后将所有record写入一个文件中，通过建立索引来标示每个分区。

如果Array存放不下，则会先扩容，如果还存放不下，就将Array中的record排序后spill到磁盘上，等待map输出完以后，再将Array中的record与磁盘上已排序的record进行全局排序，得到最终有序的record，并写入文件中。

该Shuffle模式被命名为SortShuffleWriter(KeyOrdering=true)，使用的Array被命名为PartitionedPairBuffer。

该Shuffle模式的优缺点：优点是只需要一个Array结构就可以支持按照partitionId+Key进行排序，Array大小可控，而且具有扩容和spill到磁盘的功能，支持从小规模到大规模数据的排序。同时，输出的数据已经按照partitionId进行排序，因此只需要一个分区文件存储，即可标示不同的分区数据，克服了ByPassMergeSortShuffleWriter中建立文件数过多的问题，适用于分区个数很大的情况，缺点是排序增加计算时延。

该Shuffle模式适用的操作：map端不需要聚合、Key需要排序、分区个数无限制。目前，Spark本身没有提供这种排序类型的数据操作，但不排除用户会自定义，或者系统未来会提供这种类型的操作。sortByKey操作虽然需要按Key进行排序，但这个排序过程在Shuffle Read端完成即可，不需要在Shuffle Write端进行排序。

SortShuffleWriter可以解决BypassMergeSortShuffleWriter模式的缺点，而BypassMergeSortShuffleWriter面向的操作不需要按照Key进行排序。因此，我们只需要将“按PartitionId+key”排序改成“只按PartitionId排序”，就可以支持不需要map端combine、不需要按照key进行排序、分区个数过大的操作，例如，groupByKey(300), partitionBy(300), sortByKey(300)。

#### 需要map段聚合，需要或者不需要按照key进行排序

Spark采用的实现方法是建立一个类似HashMap的数据结构对map输出的record进行聚合。HashMap中的Key是partitionId+Key，HashMap中的Value是经过combine的聚合结果。聚合完成后，Spark对HashMap中的record进行排序，最后将排序后的record写入一个分区文件中。

该Shuffle模式的优缺点：优点是只需要一个HashMap结构就可以支持map端的combine功能，HashMap具有扩容和spill到磁盘的功能，支持小规模到大规模数据的聚合，也适用于分区个数很大的情况。在聚合后使用Array排序，可以灵活支持不同的排序需求。缺点是在内存中进行聚合，内存消耗较大，需要额外的数组进行排序，而且如果有数据spill到磁盘上，还需要再次进行聚合。在实现中，Spark在Shuffle Write端使用一个经过特殊设计和优化的HashMap，命名为PartitionedAppendOnlylMap，可以同时支持聚合和排序操作。相当于HashMap和Array的合体。

该Shuffle模式适用的操作：适合map端聚合、需要或者不需要按照Key进行排序，分区个数无限制的应用，如reduceByKey、aggregateByKey等。

### Shuffle Read框架设计和实现

在Shuffle Read阶段，数据操作需要3个功能：跨节点数据获取、聚合和排序。Spark为了支持所有的情况，设计了一个通用的Shuffle Read框架，框架的计算顺序为数据获取--> 聚合 ---> 排序输出。

#### 不需要聚合，不需要按照Key进行排序

这种情况最简单，只需要实现数据获取功能即可。等待所有的map task结束后, reduce task开始不断从各个map task获取<K, V> record，并将record输出到一个buffer中（大小为spark.reducer.maxSizeInFlight=48MB），下一个操作直接从buffer获取即可。

该Shuffle模式的优缺点：优点是逻辑和实现简单，内存消耗很小。缺点是不支持聚合、排序等复杂功能。

该Shuffle模式适用的操作：适合既不需要聚合也不需要排序的应用，如partitionBy等。

#### 不需要聚合，需要按Key进行排序

获取数据后，将buffer中的record依次输出到一个Array结构（PartitionedPairBuffer）中。由于这里采用了本来用于Shuffle Write端的PartitionedPairBuffer结构，所以还保留了每个record的partitionId。然后，对Array中的record按照Key进行排序，并将排序结果输出或者传递给下一步操作。当内存无法存在所有的record时，PartitionedPairBuffer将record排序后spill到磁盘上，最后将内存中和磁盘上的record进行全局排序，得到最终排序后的record。

该Shuffle模式的优缺点：优点是只需要一个Array结构就可以支持按照Key进行排序，Array大小可控，而且具有扩容和spill到磁盘的功能，不受数据规模限制。缺点是排序增加计算时延。

该Shuffle模式适用的操作：适合reduce端不需要聚合，但需要按照Key进行排序的操作，如sortByKey，sortBy等。

#### 需要聚合，不需要或者需要按Key进行排序

获取record后，Spark建立一个类似HashMap的数据结构（ExternalAppendOnlyMap）对buffer中的record进行聚合，HashMap中的Key是record中的Key，HashMap中的Value是经过聚合函数计算后的结果。之后如果需要按照Key进行排序，则建立一个Array结构，读取HashMap中的record，并对record按Key进行排序，排序完成后，将结果输出或者传递给下一步操作。

该Shuffle模式的优缺点：优点是只需要一个HashMap和一个Array结构就可以支持reduce端的聚合和排序功能，HashMap具有扩容和spill到磁盘上的功能，支持小规模到大规模数据的聚合，边获取数据边聚合，效率较高。缺点是需要在内存中进行聚合，内存消费较大，如果有数据spill到磁盘上，还需要进行再次聚合。另外，经过HashMap聚合后的数据仍然需要拷贝到Array中进行排序，内存消耗较大。在实现中，Spark使用的HashMap是一个经过特殊优化的HashMap，命名为ExternalAppendOnlyMap，可以同时支持聚合和排序操作，相当于HashMap和Array的合体。

该Shuffle模式适用的操作：适合reduce端需要聚合、不需要或需要按Key进行排序的操作，如reduceByKey、aggregateByKey等。

## 支持高效聚合和排序的数据结构

仔细观察Shuffle Write/Read过程，我们会发现Shuffle机制中使用的数据结构的两个特征：

一是只需要支持record的插入和更新操作，不需要支持删除操作，这样我们可以对数据结构进行优化，减少内存消耗

二是只有内存放不下时才需要spill到磁盘上，因此数据结构设计以内存为主，磁盘为辅

### AppendOnlyMap

AppendOnlyMap实际上是一个只支持record添加和对Value进行更新的HashMap。于Java HashMap采用数组 + 链表的实现不同，AppendOnlyMap只使用数组来存储元素，根据元素的Hash值确定存储位置，如果存储元素时发生Hash值冲突，则使用二次地址探测法（Quadratic probing）来解决Hash值的冲突。

AppendOnlyMap将K, V相邻放在数组中，对于每个新来的<K, V> record，先使用Hash(K)计算其存放位置，如果存放位置为空，就把record存放到该位置，如果该位置已经被占用，则根据二次探测法向后指数递增位置，直到发现空位。查找和更新操作也需要根据上面的流程进行寻址。

扩容：AppendOnlyMap使用数组来实现的问题是，如果插入的record太多，则很快就被填满，Spark的解决方案是，如果AppendOnlyMap的利用率达到70%，那么就扩张一倍，扩张意味着原来的Hash失效，因此对所有Key进行rehash，重新排列每个Key的位置。

排序：由于AppendOnlyMap采用了数组作为底层存储结构，可以支持快速排序等排序算法。实现层面，先将数组中所有的<K, V> record转移到数组的前端，用begin和end来表示起始结束位置，然后调用排序算法对[begin, end]中的record进行排序。对于需要按Key进行排序的操作，如sortByKey，可以按照Key进行排序，对于其他操作，只按照Key的Hash值进行排序即可。

输出：迭代AppendOnlyMap数组中的record，从前往后扫描输出即可。

##### 源码分析

开放地址（Open addressing）是解决hash表中hash冲突的一种解决方案，通过探测或者搜索数组中的可选位置（被称为探测序列），直到目标记录被找到或者一个未使用的数组槽被发现。目前主要有以下三种探测方法：

- 线性探测（Linear probing）探测序列之间的间隔固定，通常设置为1
- 二次探测（Quadratic probing）探测序列之间的间隔线性增长，因此下标可以通过二次函数来表示
- 双散列（Double hash）对于每条记录探测序列之间的间隔固定，通过另一个hash函数确定

Spark中使用的是[二次探测法](https://en.wikipedia.org/wiki/Quadratic_probing)，如果数组长度为2的幂次方，假定为m，则可以选择探测序列 h, h + 1, h + 3 , h+ 6，间隔分别为1， 2， 3...，从而保证m长度的探测序列一定是所有数组下标的一种排列，所以只要数组中有空间，一定可以将元素顺利插入。

实现上和一般的HashMap差别不大，这里主要说一些不太一样的地方

AppendOnlyMap支持Key为null，对Key为null的情况特殊处理，使用字段`hashNullValue`和`nullValue`记录Key为null的record。

```scala
/** Get the value for a given key */
def apply(key: K): V = {
  assert(!destroyed, destructionMessage)
  val k = key.asInstanceOf[AnyRef]
  if (k.eq(null)) {
    return nullValue
  }
  var pos = rehash(k.hashCode) & mask
  var i = 1
  while (true) {
    val curKey = data(2 * pos)
    if (k.eq(curKey) || k.equals(curKey)) {
      return data(2 * pos + 1).asInstanceOf[V]
    } else if (curKey.eq(null)) {
      return null.asInstanceOf[V]
    } else {
      val delta = i
      pos = (pos + delta) & mask
      i += 1
    }
  }
  null.asInstanceOf[V]
}
```

`apply`方法用于获取指定Key对应的Value，对AppendOnlyMap进行原地排序后，原先的HashMap的性质已经丧失，使用`destoryed`字段表示这种情况，除了对Key为null的特殊处理外，通过二次探测法寻找对应记录。

```scala
/** Set the value for a key */
def update(key: K, value: V): Unit = {
  assert(!destroyed, destructionMessage)
  val k = key.asInstanceOf[AnyRef]
  if (k.eq(null)) {
    if (!haveNullValue) {
      incrementSize()
    }
    nullValue = value
    haveNullValue = true
    return
  }
  var pos = rehash(key.hashCode) & mask
  var i = 1
  while (true) {
    val curKey = data(2 * pos)
    if (curKey.eq(null)) {
      data(2 * pos) = k
      data(2 * pos + 1) = value.asInstanceOf[AnyRef]
      incrementSize()  // Since we added a new key
      return
    } else if (k.eq(curKey) || k.equals(curKey)) {
      data(2 * pos + 1) = value.asInstanceOf[AnyRef]
      return
    } else {
      val delta = i
      pos = (pos + delta) & mask
      i += 1
    }
  }
}
```

`update`更新操作也是通过二次探测法寻找对应位置并插入或者更新。

```scala
/**
 * Return an iterator of the map in sorted order. This provides a way to sort the map without
 * using additional memory, at the expense of destroying the validity of the map.
 */
def destructiveSortedIterator(keyComparator: Comparator[K]): Iterator[(K, V)] = {
  destroyed = true
  // Pack KV pairs into the front of the underlying array
  var keyIndex, newIndex = 0
  while (keyIndex < capacity) {
    if (data(2 * keyIndex) != null) {
      data(2 * newIndex) = data(2 * keyIndex)
      data(2 * newIndex + 1) = data(2 * keyIndex + 1)
      newIndex += 1
    }
    keyIndex += 1
  }
  assert(curSize == newIndex + (if (haveNullValue) 1 else 0))

  new Sorter(new KVArraySortDataFormat[K, AnyRef]).sort(data, 0, newIndex, keyComparator)

  new Iterator[(K, V)] {
    var i = 0
    var nullValueReady = haveNullValue
    def hasNext: Boolean = (i < newIndex || nullValueReady)
    def next(): (K, V) = {
      if (nullValueReady) {
        nullValueReady = false
        (null.asInstanceOf[K], nullValue)
      } else {
        val item = (data(2 * i).asInstanceOf[K], data(2 * i + 1).asInstanceOf[V])
        i += 1
        item
      }
    }
  }
}
```

`destructiveSortedIterator`将HashMap中的所有记录整理到数组的开头，然后调用tim sort进行原地排序，tim sort结合了归并排序和插入排序，最后返回记录的迭代器。

### ExternalAppendOnlyMap

AppendOnlyMap的优点是能够将聚合和排序功能很好地结合在一起，缺点是只能使用内存，难以适用于内存空间不足的问题。为了解决这个问题，Spark基于AppendOnlyMap设计实现了基于内存+磁盘的ExternalAppendOnlyMap，用于Shuffle Read端大规模数据聚合。同时，由于Shuffle Write端聚合需要考虑partitionId，Spark也设计了带有partitionId的ExternalAppendOnlyMap，名为PartitionedAppendOnlyMap。

ExternalAppendOnlyMap的工作原理是，先持有一个AppendOnlyMap来不断接收和聚合新来的record，AppendOnlyMap快被装满时检查一下内存剩余空间是否可以扩展，可以的话直接在内存中扩展，否则对AppendOnlyMap中的record进行排序，然后将record都spill到磁盘上。因为record不断到来，可能会多次填满AppendOnlyMap，所以这个spill过程可以出现多次形成多个spill文件。等record都处理完，此时AppendOnlyMap中可能还留存一些聚合后的record，磁盘上也有多个spill文件。因为这些数据都经过了部分聚合，还需要进行全局聚合（merge）。因此ExternalAppendOnlyMap的最后一步是将内存中的AppendOnlyMap的数据和磁盘上spill文件中的数据进行全局聚合，得到最终结果。

#### AppendOnlyMap的大小估计

虽然我们知道AppendOnlyMap中持有的数组的长度和大小，但数组里面存放的是Key和Value的引用，并不是它们的实际对象大小，而且Value会不断被更新，实际大小不断变化。想要准备得到AppendOnlyMap的大小比较困难。一种简单的解决方法是在每次插入record或对现有record的Value进行更新后，都扫描一下AppendOnlyMap中存放的record，计算每个record的实际对象大小并相加，但这样会非常耗时。

Spark设计了一个增量式的高效估算算法，在每个record插入或更新时根据历史统计值和当前变化量直接估算当前AppendOnlyMap的大小，算法的复杂度为O(1)，开销很小，在record插入和聚合过程中会定期对当前AppendOnlyMap中的record进行抽样，然后精确计算这些record的总大小、总个数、更新个数及平均值等，并作为历史统计值。进行抽样是因为AppendOnlyMap中的record可能有上万个，难以对每个都精确计算。之后，每当有record插入或更新时，会根据历史统计值和历史平均的变化值，增量估算AppendOnlyMap的总大小，详见SizeTracker.estimateSize方法。抽样也会定期进行，更新统计值以获取更高的精度。

#### Spill过程与排序

当AppendOnlyMap达到内存限制时，会将record排序后写入磁盘中。排序是为了方便下一步全局聚合（聚合内存和磁盘上的record）时可以采用更高效的merge-sort（外部排序+聚合）。那么，问题是依据什么对record进行排序？自然想要可以根据record的Key进行排序，但是这就要求操作定义Key的排序方法，如sortByKey等操作定义了按照Key进行的排序。大部分操作，如groupByKey，并没有定义Key的排序方法，也不需要输出结果按照Key进行排序。在这种情况下，Spark采用按照Key的Hash值进行排序的方法，这样既可以进行merge-sort，又不要求操作定义Key排序的方法。然而，这种方法的问题是会出现Hash值冲突，也就是不同的Key具有相同的Hash值。为了解决这个问题，Spark在merge-sort的同时会比较Key的Hash值是否相等，以及Key的实际值是否相等。

#### 全局聚合

由于最终的spill文件和内存中的AppendOnlyMap都是经过部分聚合后的结果，其中可能存在相同Key的record，因此还需要一个全局聚合阶段将AppendOnlyMap中的record与spill文件中的record进行聚合，得到最终聚合后的结果。

全局聚合的方法是建立一个最小堆或者最大堆，每次从各个spill文件中读取前几个具有相同Key（或者相同Key的hash值）的record，然后与AppendOnlyMap中的record进行聚合，并输出聚合后的结果。

#### 源码分析

##### SizeTracker

```scala
/**
 * A general interface for collections to keep track of their estimated sizes in bytes.
 * We sample with a slow exponential back-off using the SizeEstimator to amortize the time,
 * as each call to SizeEstimator is somewhat expensive (order of a few milliseconds).
 */
private[spark] trait SizeTracker {

  import SizeTracker._

  /**
   * Controls the base of the exponential which governs the rate of sampling.
   * E.g., a value of 2 would mean we sample at 1, 2, 4, 8, ... elements.
   */
  private val SAMPLE_GROWTH_RATE = 1.1

  /** Samples taken since last resetSamples(). Only the last two are kept for extrapolation. */
  private val samples = new mutable.Queue[Sample]

  /** The average number of bytes per update between our last two samples. */
  private var bytesPerUpdate: Double = _

  /** Total number of insertions and updates into the map since the last resetSamples(). */
  private var numUpdates: Long = _

  /** The value of 'numUpdates' at which we will take our next sample. */
  private var nextSampleNum: Long = _

  resetSamples()

  /**
   * Reset samples collected so far.
   * This should be called after the collection undergoes a dramatic change in size.
   */
  protected def resetSamples(): Unit = {
    numUpdates = 1
    nextSampleNum = 1
    samples.clear()
    takeSample()
  }

  /**
   * Callback to be invoked after every update.
   */
  protected def afterUpdate(): Unit = {
    numUpdates += 1
    if (nextSampleNum == numUpdates) {
      takeSample()
    }
  }

  /**
   * Take a new sample of the current collection's size.
   */
  private def takeSample(): Unit = {
    samples.enqueue(Sample(SizeEstimator.estimate(this), numUpdates))
    // Only use the last two samples to extrapolate
    if (samples.size > 2) {
      samples.dequeue()
    }
    val bytesDelta = samples.toList.reverse match {
      case latest :: previous :: tail =>
        (latest.size - previous.size).toDouble / (latest.numUpdates - previous.numUpdates)
      // If fewer than 2 samples, assume no change
      case _ => 0
    }
    bytesPerUpdate = math.max(0, bytesDelta)
    nextSampleNum = math.ceil(numUpdates * SAMPLE_GROWTH_RATE).toLong
  }

  /**
   * Estimate the current size of the collection in bytes. O(1) time.
   */
  def estimateSize(): Long = {
    assert(samples.nonEmpty)
    val extrapolatedDelta = bytesPerUpdate * (numUpdates - samples.last.numUpdates)
    (samples.last.size + extrapolatedDelta).toLong
  }
}

private object SizeTracker {
  case class Sample(size: Long, numUpdates: Long)
}
```

通过`SizeEstimator.estimate`估算当前容器占用的内存大小，通过指数退避算法周期性地采样，以均摊估算内存大小的成本。每次容器数据更新时，都增加`numUpdates`计数，如果达到了指数退避算法指定的采样计数`nextSampleNum`，则进行一次采样，并保留最近两次采样，计算最近两次采样间隔中每次更新数据增加的平均内存大小，则当前时刻的`estimateSize`是最近一次采样的容器大小 + 自上次采样以来的更新次数 * 每次更新数据增加的平均内存大小。

##### SizeTrackingAppendOnlyMap

```scala
private[spark] class SizeTrackingAppendOnlyMap[K, V]
  extends AppendOnlyMap[K, V] with SizeTracker
{
  override def update(key: K, value: V): Unit = {
    super.update(key, value)
    super.afterUpdate()
  }

  override def changeValue(key: K, updateFunc: (Boolean, V) => V): V = {
    val newValue = super.changeValue(key, updateFunc)
    super.afterUpdate()
    newValue
  }

  override protected def growTable(): Unit = {
    super.growTable()
    resetSamples()
  }
}
```

SizeTrackingAppendOnlyMap在每次更新数据时，都会增加更新计数，并决定是否采样。另外如果容器发生扩容，则重置采样数据。

##### ExternalAppendOnlyMap

ExternalAppendOnlyMap拓展了`Spillable`，实现了吐磁盘的功能

```scala
class ExternalAppendOnlyMap[K, V, C](
    createCombiner: V => C,
    mergeValue: (C, V) => C,
    mergeCombiners: (C, C) => C,
    serializer: Serializer = SparkEnv.get.serializer,
    blockManager: BlockManager = SparkEnv.get.blockManager,
    context: TaskContext = TaskContext.get(),
    serializerManager: SerializerManager = SparkEnv.get.serializerManager)
  extends Spillable[SizeTracker](context.taskMemoryManager())
  with Serializable
  with Logging
  with Iterable[(K, C)] {

  @volatile private[collection] var currentMap = new SizeTrackingAppendOnlyMap[K, C]
  private val spilledMaps = new ArrayBuffer[DiskMapIterator]
  private val sparkConf = SparkEnv.get.conf
  private val diskBlockManager = blockManager.diskBlockManager

  /**
   * Size of object batches when reading/writing from serializers.
   *
   * Objects are written in batches, with each batch using its own serialization stream. This
   * cuts down on the size of reference-tracking maps constructed when deserializing a stream.
   *
   * NOTE: Setting this too low can cause excessive copying when serializing, since some serializers
   * grow internal data structures by growing + copying every time the number of objects doubles.
   */
  private val serializerBatchSize = sparkConf.get(config.SHUFFLE_SPILL_BATCH_SIZE)

  // Number of bytes spilled in total
  private var _diskBytesSpilled = 0L
  def diskBytesSpilled: Long = _diskBytesSpilled

  // Use getSizeAsKb (not bytes) to maintain backwards compatibility if no units are provided
  private val fileBufferSize = sparkConf.get(config.SHUFFLE_FILE_BUFFER_SIZE).toInt * 1024

  // Write metrics
  private val writeMetrics: ShuffleWriteMetrics = new ShuffleWriteMetrics()

  // Peak size of the in-memory map observed so far, in bytes
  private var _peakMemoryUsedBytes: Long = 0L
  def peakMemoryUsedBytes: Long = _peakMemoryUsedBytes

  private val keyComparator = new HashComparator[K]
  private val ser = serializer.newInstance()

  @volatile private var readingIterator: SpillableIterator = null
```

字段：

`currentMap`: SizeTrackingAppendOnlyMap，内存中的hash表，支持估计内存占用

`spillendMaps`：ArrayBuffer[DiskMapIterator] 每次吐磁盘后，都会返回iterator用来遍历磁盘中的数据

`_diskByteSpilled`：吐磁盘的总字节数

`fileBufferSize`：每个shuffle文件输出流的buffer大小，通过`spark.shuffle.file.buffer`指定，默认是32k

`keyComparator`：HashComparator，排序的目的是为了使用归并排序，不是所有操作都定义了按照Key的排序，所以这里使用基于Hash值的排序

`readingIterator`：SpillableIterator TODO

方法：

```scala
/**
 * Insert the given iterator of keys and values into the map.
 *
 * When the underlying map needs to grow, check if the global pool of shuffle memory has
 * enough room for this to happen. If so, allocate the memory required to grow the map;
 * otherwise, spill the in-memory map to disk.
 *
 * The shuffle memory usage of the first trackMemoryThreshold entries is not tracked.
 */
def insertAll(entries: Iterator[Product2[K, V]]): Unit = {
  if (currentMap == null) {
    throw new IllegalStateException(
      "Cannot insert new elements into a map after calling iterator")
  }
  // An update function for the map that we reuse across entries to avoid allocating
  // a new closure each time
  var curEntry: Product2[K, V] = null
  val update: (Boolean, C) => C = (hadVal, oldVal) => {
    if (hadVal) mergeValue(oldVal, curEntry._2) else createCombiner(curEntry._2)
  }
	// 遍历每个K, V对
  while (entries.hasNext) {
    curEntry = entries.next()
    // 插入前，先估算当前currentMap的大小
    val estimatedSize = currentMap.estimateSize()
    // 更新巅峰占用内存
    if (estimatedSize > _peakMemoryUsedBytes) {
      _peakMemoryUsedBytes = estimatedSize
    }
    // 判断是否spill，如果发生spill，创建新的currentMap
    if (maybeSpill(currentMap, estimatedSize)) {
      currentMap = new SizeTrackingAppendOnlyMap[K, C]
    }
    // 插入新的记录
    currentMap.changeValue(curEntry._1, update)
    // 更新_elementsRead计数
    addElementsRead()
  }
}
```

insertAll负责更新记录，如果内存占用超过限制，则吐磁盘，核心实现在`Spillable.maybeSpill`方法中。

```scala
// Initial threshold for the size of a collection before we start tracking its memory usage
// For testing only
private[this] val initialMemoryThreshold: Long =
  SparkEnv.get.conf.get(SHUFFLE_SPILL_INITIAL_MEM_THRESHOLD)

// Threshold for this collection's size in bytes before we start tracking its memory usage
// To avoid a large number of small spills, initialize this to a value orders of magnitude > 0
@volatile private[this] var myMemoryThreshold = initialMemoryThreshold
/**
 * Spills the current in-memory collection to disk if needed. Attempts to acquire more
 * memory before spilling.
 *
 * @param collection collection to spill to disk
 * @param currentMemory estimated size of the collection in bytes
 * @return true if `collection` was spilled to disk; false otherwise
 */
protected def maybeSpill(collection: C, currentMemory: Long): Boolean = {
  var shouldSpill = false
  // myMemoryThresold的作用类似于内存配额，或者说已经申请到的内存大小
  // 如果当前内存占用大于申请到的内存大小
  if (elementsRead % 32 == 0 && currentMemory >= myMemoryThreshold) {
    // Claim up to double our current memory from the shuffle memory pool
    // 尝试申请最多两倍的currentMemory
    val amountToRequest = 2 * currentMemory - myMemoryThreshold
    val granted = acquireMemory(amountToRequest)
    // 更新申请到的内存大小
    myMemoryThreshold += granted
    // If we were granted too little memory to grow further (either tryToAcquire returned 0,
    // or we already had more memory than myMemoryThreshold), spill the current collection
    // 如果内存申请不到，并且集合的内存占用大于申请到的内存
    shouldSpill = currentMemory >= myMemoryThreshold
  }
  shouldSpill = shouldSpill || _elementsRead > numElementsForceSpillThreshold
  // Actually spill
  if (shouldSpill) {
    _spillCount += 1
    logSpillage(currentMemory)
    spill(collection)
    _elementsRead = 0
    _memoryBytesSpilled += currentMemory
    releaseMemory()
  }
  shouldSpill
}
```

`spark.shuffle.spill.initialMemoryThreshold`默认为5kb，如果集合的内存占用低于指定值，则不会跟踪内存占用，避免大量的小集合被吐磁盘。

`maybeSpill`如果当前集合的内存占用高于申请到的内存大小，则调用`spill(collection)`开始吐磁盘，并释放内存`releaseMemory`。

```scala
/**
 * Sort the existing contents of the in-memory map and spill them to a temporary file on disk.
 */
override protected[this] def spill(collection: SizeTracker): Unit = {
  val inMemoryIterator = currentMap.destructiveSortedIterator(keyComparator)
  val diskMapIterator = spillMemoryIteratorToDisk(inMemoryIterator)
  spilledMaps += diskMapIterator
}
/**
 * Spill the in-memory Iterator to a temporary file on disk.
 */
private[this] def spillMemoryIteratorToDisk(inMemoryIterator: Iterator[(K, C)])
    : DiskMapIterator = {
  // 利用uuid创建一个临时的block
  val (blockId, file) = diskBlockManager.createTempLocalBlock()
  // 获取文件的writer
  val writer = blockManager.getDiskWriter(blockId, file, ser, fileBufferSize, writeMetrics)
  var objectsWritten = 0

  // List of batch sizes (bytes) in the order they are written to disk
  val batchSizes = new ArrayBuffer[Long]

  // Flush the disk writer's contents to disk, and update relevant variables
  def flush(): Unit = {
    val segment = writer.commitAndGet()
    batchSizes += segment.length
    _diskBytesSpilled += segment.length
    objectsWritten = 0
  }

  var success = false
  try {
    // 遍历迭代器，分批将record flush到磁盘
    while (inMemoryIterator.hasNext) {
      val kv = inMemoryIterator.next()
      writer.write(kv._1, kv._2)
      objectsWritten += 1

      if (objectsWritten == serializerBatchSize) {
        flush()
      }
    }
    if (objectsWritten > 0) {
      flush()
      writer.close()
    } else {
      writer.revertPartialWritesAndClose()
    }
    success = true
  } finally {
    if (!success) {
      // This code path only happens if an exception was thrown above before we set success;
      // close our stuff and let the exception be thrown further
      writer.closeAndDelete()
    }
  }

  new DiskMapIterator(file, blockId, batchSizes)
}
```

`spill`首先调用hashMap的`destructiveSortedIterator`按照Key的hash值进行排序，然后将排好序的集合输出到磁盘。

```scala
/**
 * Returns a destructive iterator for iterating over the entries of this map.
 * If this iterator is forced spill to disk to release memory when there is not enough memory,
 * it returns pairs from an on-disk map.
 */
def destructiveIterator(inMemoryIterator: Iterator[(K, C)]): Iterator[(K, C)] = {
  readingIterator = new SpillableIterator(inMemoryIterator)
  readingIterator.toCompletionIterator
}
/**
 * Return a destructive iterator that merges the in-memory map with the spilled maps.
 * If no spill has occurred, simply return the in-memory map's iterator.
 */
override def iterator: Iterator[(K, C)] = {
  if (currentMap == null) {
    throw new IllegalStateException(
      "ExternalAppendOnlyMap.iterator is destructive and should only be called once.")
  }
  if (spilledMaps.isEmpty) {
    destructiveIterator(currentMap.iterator)
  } else {
    new ExternalIterator()
  }
}
```

iterator返回map的迭代器，如果没有spill发生，直接返回内存中map的迭代器，否则返回一个合并了内存中map和磁盘中map的迭代器。

```scala
/**
 * Wrapper around an iterator which calls a completion method after it successfully iterates
 * through all the elements.
 */
private[spark]
abstract class CompletionIterator[ +A, +I <: Iterator[A]](sub: I) extends Iterator[A] {

  private[this] var completed = false
  private[this] var iter = sub
  def next(): A = iter.next()
  def hasNext: Boolean = {
    val r = iter.hasNext
    if (!r && !completed) {
      completed = true
      // reassign to release resources of highly resource consuming iterators early
      iter = Iterator.empty.asInstanceOf[I]
      completion()
    }
    r
  }

  def completion(): Unit
}

private[spark] object CompletionIterator {
  def apply[A, I <: Iterator[A]](sub: I, completionFunction: => Unit) : CompletionIterator[A, I] = {
    new CompletionIterator[A, I](sub) {
      def completion(): Unit = completionFunction
    }
  }
}
```

CompletionIterator包装了原先的Iterator，会在遍历完全部记录后，调用回调函数`completion`。

```scala
private class SpillableIterator(var upstream: Iterator[(K, C)])
  extends Iterator[(K, C)] {

  private val SPILL_LOCK = new Object()

  private var cur: (K, C) = readNext()

  private var hasSpilled: Boolean = false

  def spill(): Boolean = SPILL_LOCK.synchronized {
    if (hasSpilled) {
      false
    } else {
      logInfo(log"Task ${MDC(TASK_ATTEMPT_ID, context.taskAttemptId())} force spilling" +
        log" in-memory map to disk and it will release " +
        log"${MDC(NUM_BYTES, org.apache.spark.util.Utils.bytesToString(getUsed()))} memory")
      val nextUpstream = spillMemoryIteratorToDisk(upstream)
      assert(!upstream.hasNext)
      hasSpilled = true
      upstream = nextUpstream
      true
    }
  }

  private def destroy(): Unit = {
    freeCurrentMap()
    upstream = Iterator.empty
  }

  def toCompletionIterator: CompletionIterator[(K, C), SpillableIterator] = {
    CompletionIterator[(K, C), SpillableIterator](this, this.destroy())
  }

  def readNext(): (K, C) = SPILL_LOCK.synchronized {
    if (upstream.hasNext) {
      upstream.next()
    } else {
      null
    }
  }

  override def hasNext: Boolean = cur != null

  override def next(): (K, C) = {
    val r = cur
    cur = readNext()
    r
  }
}
```

`SpillableIterator`也是上游迭代器的包装，支持spill操作，可以返回`CompletionIterator`，在遍历完成后调用`destory`清理资源。

```scala
/**
 * An iterator that sort-merges (K, C) pairs from the in-memory map and the spilled maps
 */
private class ExternalIterator extends Iterator[(K, C)] {

  // A queue that maintains a buffer for each stream we are currently merging
  // This queue maintains the invariant that it only contains non-empty buffers
  // 堆，用作堆排序
  private val mergeHeap = new mutable.PriorityQueue[StreamBuffer]

  // Input streams are derived both from the in-memory map and spilled maps on disk
  // The in-memory map is sorted in place, while the spilled maps are already in sorted order
  // input stearam集合，包括内存中map和磁盘上map
  private val sortedMap = destructiveIterator(
    currentMap.destructiveSortedIterator(keyComparator))
  private val inputStreams = (Seq(sortedMap) ++ spilledMaps).map(it => it.buffered)
	// 将每个iterator中最小hash值对应的多条记录保存到kcPairs，并放入堆中进行排序
  // 每个iterator中，由于hash碰撞，可能出现多个Key对应同一个hash值
  // 但由于iterator中保存的都是聚合的结果，所以不可以有同一个key的多条记录
  inputStreams.foreach { it =>
    val kcPairs = new ArrayBuffer[(K, C)]
    readNextHashCode(it, kcPairs)
    if (kcPairs.length > 0) {
      mergeHeap.enqueue(new StreamBuffer(it, kcPairs))
    }
  }

  private def readNextHashCode(it: BufferedIterator[(K, C)], buf: ArrayBuffer[(K, C)]): Unit = {
    if (it.hasNext) {
      var kc = it.next()
      buf += kc
      val minHash = hashKey(kc)
      while (it.hasNext && it.head._1.hashCode() == minHash) {
        kc = it.next()
        buf += kc
      }
    }
  }

  /**
   * If the given buffer contains a value for the given key, merge that value into
   * baseCombiner and remove the corresponding (K, C) pair from the buffer.
   */
  private def mergeIfKeyExists(key: K, baseCombiner: C, buffer: StreamBuffer): C = {
    var i = 0
    // 遍历buffer中的记录，找到相同的key，并且merge，然后删除这条记录
    while (i < buffer.pairs.length) {
      val pair = buffer.pairs(i)
      if (pair._1 == key) {
        // Note that there's at most one pair in the buffer with a given key, since we always
        // merge stuff in a map before spilling, so it's safe to return after the first we find
        removeFromBuffer(buffer.pairs, i)
        return mergeCombiners(baseCombiner, pair._2)
      }
      i += 1
    }
    baseCombiner
  }

	// 这里通过交换元素并移除最后一个元素的方式实现高效的移除任意元素
  private def removeFromBuffer[T](buffer: ArrayBuffer[T], index: Int): T = {
    val elem = buffer(index)
    buffer(index) = buffer(buffer.size - 1)  // This also works if index == buffer.size - 1
    buffer.dropRightInPlace(1)
    elem
  }

  /**
   * Return true if there exists an input stream that still has unvisited pairs.
   */
  override def hasNext: Boolean = mergeHeap.nonEmpty

  /**
   * Select a key with the minimum hash, then combine all values with the same key from all
   * input streams.
   */
  override def next(): (K, C) = {
    if (mergeHeap.isEmpty) {
      throw new NoSuchElementException
    }
    // Select a key from the StreamBuffer that holds the lowest key hash
    // 取出拥有最小hash值的buffer
    val minBuffer = mergeHeap.dequeue()
    val minPairs = minBuffer.pairs
    val minHash = minBuffer.minKeyHash
    val minPair = removeFromBuffer(minPairs, 0)
    val minKey = minPair._1
    var minCombiner = minPair._2
    assert(hashKey(minPair) == minHash)

    // For all other streams that may have this key (i.e. have the same minimum key hash),
    // merge in the corresponding value (if any) from that stream
    // 查询堆顶元素，如果hash值相同，则出队，并尝试合并相同Key的记录
    val mergedBuffers = ArrayBuffer[StreamBuffer](minBuffer)
    while (mergeHeap.nonEmpty && mergeHeap.head.minKeyHash == minHash) {
      val newBuffer = mergeHeap.dequeue()
      minCombiner = mergeIfKeyExists(minKey, minCombiner, newBuffer)
      mergedBuffers += newBuffer
    }

    // Repopulate each visited stream buffer and add it back to the queue if it is non-empty
    // 检查出队的iterator是否为空，如果为空，则丢弃，否则如果buffer不为空，直接入队
    // 如果buffer为空，读取下一个最小的hash值的记录，并入队
    mergedBuffers.foreach { buffer =>
      if (buffer.isEmpty) {
        readNextHashCode(buffer.iterator, buffer.pairs)
      }
      if (!buffer.isEmpty) {
        mergeHeap.enqueue(buffer)
      }
    }

    (minKey, minCombiner)
  }

  private class StreamBuffer(
      val iterator: BufferedIterator[(K, C)],
      val pairs: ArrayBuffer[(K, C)])
    extends Comparable[StreamBuffer] {

    def isEmpty: Boolean = pairs.length == 0

    // Invalid if there are no more pairs in this stream
    def minKeyHash: Int = {
      assert(pairs.length > 0)
      hashKey(pairs.head)
    }

    override def compareTo(other: StreamBuffer): Int = {
      // descending order because mutable.PriorityQueue dequeues the max, not the min
      if (other.minKeyHash < minKeyHash) -1 else if (other.minKeyHash == minKeyHash) 0 else 1
    }
  }
}
```

ExternalIterator通过堆排序对内存中map和磁盘上map的iterator进行合并。

```scala
/**
 * An iterator that returns (K, C) pairs in sorted order from an on-disk map
 */
private class DiskMapIterator(file: File, blockId: BlockId, batchSizes: ArrayBuffer[Long])
  extends Iterator[(K, C)]
{
  private val batchOffsets = batchSizes.scanLeft(0L)(_ + _)  // Size will be batchSize.length + 1
  assert(file.length() == batchOffsets.last,
    "File length is not equal to the last batch offset:\n" +
    s"    file length = ${file.length}\n" +
    s"    last batch offset = ${batchOffsets.last}\n" +
    s"    all batch offsets = ${batchOffsets.mkString(",")}"
  )

  private var batchIndex = 0  // Which batch we're in
  private var fileStream: FileInputStream = null

  // An intermediate stream that reads from exactly one batch
  // This guards against pre-fetching and other arbitrary behavior of higher level streams
  private var deserializeStream: DeserializationStream = null
  private var batchIterator: Iterator[(K, C)] = null
  private var objectsRead = 0

  /**
   * Construct a stream that reads only from the next batch.
   */
  private def nextBatchIterator(): Iterator[(K, C)] = {
    // Note that batchOffsets.length = numBatches + 1 since we did a scan above; check whether
    // we're still in a valid batch.
    if (batchIndex < batchOffsets.length - 1) {
      if (deserializeStream != null) {
        deserializeStream.close()
        fileStream.close()
        deserializeStream = null
        fileStream = null
      }

      val start = batchOffsets(batchIndex)
      fileStream = new FileInputStream(file)
      fileStream.getChannel.position(start)
      batchIndex += 1

      val end = batchOffsets(batchIndex)

      assert(end >= start, "start = " + start + ", end = " + end +
        ", batchOffsets = " + batchOffsets.mkString("[", ", ", "]"))

      val bufferedStream = new BufferedInputStream(ByteStreams.limit(fileStream, end - start))
      val wrappedStream = serializerManager.wrapStream(blockId, bufferedStream)
      deserializeStream = ser.deserializeStream(wrappedStream)
      deserializeStream.asKeyValueIterator.asInstanceOf[Iterator[(K, C)]]
    } else {
      // No more batches left
      cleanup()
      null
    }
  }

  /**
   * Return the next (K, C) pair from the deserialization stream.
   *
   * If the current batch is drained, construct a stream for the next batch and read from it.
   * If no more pairs are left, return null.
   */
  private def readNextItem(): (K, C) = {
    val item = batchIterator.next()
    objectsRead += 1
    if (objectsRead == serializerBatchSize) {
      objectsRead = 0
      batchIterator = nextBatchIterator()
    }
    item
  }

  override def hasNext: Boolean = {
    if (batchIterator == null) {
      // In case of batchIterator has not been initialized
      batchIterator = nextBatchIterator()
      if (batchIterator == null) {
        return false
      }
    }
    batchIterator.hasNext
  }

  override def next(): (K, C) = {
    if (!hasNext) {
      throw new NoSuchElementException
    }
    readNextItem()
  }

  private def cleanup(): Unit = {
    batchIndex = batchOffsets.length  // Prevent reading any other batch
    if (deserializeStream != null) {
      deserializeStream.close()
      deserializeStream = null
    }
    if (fileStream != null) {
      fileStream.close()
      fileStream = null
    }
    if (file.exists()) {
      if (!file.delete()) {
        logWarning(log"Error deleting ${MDC(FILE_NAME, file)}")
      }
    }
  }

  context.addTaskCompletionListener[Unit](context => cleanup())
}
```

`DiskMapIterator`从磁盘中的shuffle文件中读取Key、Value键值对，由于shuffle文件是分批写入的，并且每次达到`serializerBatchSize`将之前的内容批量写入。

所以读取时，如果达到`serializerBatchSize`也需要新创建一个批的读取stream。

##### HashComparator

```scala
/**
 * A comparator which sorts arbitrary keys based on their hash codes.
 */
private class HashComparator[K] extends Comparator[K] {
  def compare(key1: K, key2: K): Int = {
    val hash1 = hash(key1)
    val hash2 = hash(key2)
    if (hash1 < hash2) -1 else if (hash1 == hash2) 0 else 1
  }
}
```

HashComparator的实现非常简单，计算hash值，并根据hash值的大小进行排序。

### PartitionedAppendOnlyMap

```scala
/**
 * Implementation of WritablePartitionedPairCollection that wraps a map in which the keys are tuples
 * of (partition ID, K)
 */
private[spark] class PartitionedAppendOnlyMap[K, V]
  extends SizeTrackingAppendOnlyMap[(Int, K), V] with WritablePartitionedPairCollection[K, V] {

  def partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]])
    : Iterator[((Int, K), V)] = {
    val comparator = keyComparator.map(partitionKeyComparator).getOrElse(partitionComparator)
    destructiveSortedIterator(comparator)
  }

  def insert(partition: Int, key: K, value: V): Unit = {
    update((partition, key), value)
  }
}
```

PartitionedAppendOnlyMap用于在Shuffle Write端对record进行聚合（combine）。PartitionedAppendOnlyMap的功能和实现与ExternalAppendOnlyMap的功能和实现基本一样，唯一区别是PartitionedAppendOnlyMap中的Key是PartitionId + Key，这样既可以根据partitionId进行排序（面向不需要按key进行排序的操作），也可以根据partitionId + Key进行排序（面向需要按Key进行排序的操作），从而在Shuffle Write阶段可以进行聚合、排序和分区。

### PartitionedPairBuffer

```scala
private[spark] class PartitionedPairBuffer[K, V](initialCapacity: Int = 64)
  extends WritablePartitionedPairCollection[K, V] with SizeTracker
{
  import PartitionedPairBuffer._

  require(initialCapacity <= MAXIMUM_CAPACITY,
    s"Can't make capacity bigger than ${MAXIMUM_CAPACITY} elements")
  require(initialCapacity >= 1, "Invalid initial capacity")

  // Basic growable array data structure. We use a single array of AnyRef to hold both the keys
  // and the values, so that we can sort them efficiently with KVArraySortDataFormat.
  private var capacity = initialCapacity
  private var curSize = 0
  private var data = new Array[AnyRef](2 * initialCapacity)

  /** Add an element into the buffer */
  def insert(partition: Int, key: K, value: V): Unit = {
    if (curSize == capacity) {
      growArray()
    }
    data(2 * curSize) = (partition, key.asInstanceOf[AnyRef])
    data(2 * curSize + 1) = value.asInstanceOf[AnyRef]
    curSize += 1
    afterUpdate()
  }

  /** Double the size of the array because we've reached capacity */
  private def growArray(): Unit = {
    if (capacity >= MAXIMUM_CAPACITY) {
      throw new IllegalStateException(s"Can't insert more than ${MAXIMUM_CAPACITY} elements")
    }
    val newCapacity =
      if (capacity * 2 > MAXIMUM_CAPACITY) { // Overflow
        MAXIMUM_CAPACITY
      } else {
        capacity * 2
      }
    val newArray = new Array[AnyRef](2 * newCapacity)
    System.arraycopy(data, 0, newArray, 0, 2 * capacity)
    data = newArray
    capacity = newCapacity
    resetSamples()
  }

  /** Iterate through the data in a given order. For this class this is not really destructive. */
  override def partitionedDestructiveSortedIterator(keyComparator: Option[Comparator[K]])
    : Iterator[((Int, K), V)] = {
    val comparator = keyComparator.map(partitionKeyComparator).getOrElse(partitionComparator)
    new Sorter(new KVArraySortDataFormat[(Int, K), AnyRef]).sort(data, 0, curSize, comparator)
    iterator()
  }

  private def iterator(): Iterator[((Int, K), V)] = new Iterator[((Int, K), V)] {
    var pos = 0

    override def hasNext: Boolean = pos < curSize

    override def next(): ((Int, K), V) = {
      if (!hasNext) {
        throw new NoSuchElementException
      }
      val pair = (data(2 * pos).asInstanceOf[(Int, K)], data(2 * pos + 1).asInstanceOf[V])
      pos += 1
      pair
    }
  }
}
```

PartitionedPariBuffer本质上是一个基于内存+磁盘的Array，随着数据添加，不断的扩容，但到达内存限制时，就将Array中的数据按照partitionId或者partitionId+Key进行排序，然后spill到磁盘上，该过程可以进行多次，最后对内存中和磁盘上的数据进行全局排序，输出或者提供给下一个操作。

### ExternalSorter

```scala
private[spark] class ExternalSorter[K, V, C](
    context: TaskContext,
    aggregator: Option[Aggregator[K, V, C]] = None,
    partitioner: Option[Partitioner] = None,
    ordering: Option[Ordering[K]] = None,
    serializer: Serializer = SparkEnv.get.serializer)
  extends Spillable[WritablePartitionedPairCollection[K, C]](context.taskMemoryManager())
  with Logging with ShuffleChecksumSupport {
```

字段：

aggregator: Option[Aggregator[K, V, C]] 可选的聚合函数

partitioner: 分区

ordering： Option[Ordering[K]] 可选的排序

如果需要聚合，使用`PartitionedAppendOnlyMap`作为内存中的数据结构（类似于HashMap），否则使用`PartitionedPairBuffer`作为内存数据结构（类似于动态数组）。

由于需要分区，所以不管Key需不需要排序，都需要按照partitionId进行排序，然后才能写入map out writer。

