---
title: "Shuffle"
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

### PartitionedAppendOnlyMap

PartitionedAppendOnlyMap用于在Shuffle Write端对record进行聚合（combine）。PartitionedAppendOnlyMap的功能和实现与ExternalAppendOnlyMap的功能和实现基本一样，唯一区别是PartitionedAppendOnlyMap中的Key是PartitionId + Key，这样既可以根据partitionId进行排序（面向不需要按key进行排序的操作），也可以根据partitionId + Key进行排序（面向需要按Key进行排序的操作），从而在Shuffle Write阶段可以进行聚合、排序和分区。

### PartitionedPairBuffer

PartitionedPariBuffer本质上是一个基于内存+磁盘的Array，随着数据添加，不断的扩容，但到达内存限制时，就将Array中的数据按照partitionId或者partitionId+Key进行排序，然后spill到磁盘上，该过程可以进行多次，最后对内存中和磁盘上的数据进行全局排序，输出或者提供给下一个操作。





