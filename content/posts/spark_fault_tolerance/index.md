---
title: "Spark错误容忍机制"
author: "爱吃芒果"
description:
date: "2025-06-07T19:05:49+08:00"
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

Spark的错误容忍机制的核心方法主要有两种：

1. 通过重新执行计算任务来容忍错误，当job抛出异常不能继续执行时，重新启动计算任务，再次执行
2. 通过采用checkpoint机制，对一些重要的输入、输出、中间数据进行持久化，这可以在一定程度上解决数据丢失问题，而且能够提高任务重新计算时的效率。

Spark采用了延迟删除策略，将上游stage的Shuffle Write的结果写入本地磁盘，只有在当前job完成后，才删除Shuffle Writre写入磁盘的数据。这样，即使stage2中某个task执行失败，但由于上游的stage0和stage1的输出数据还在磁盘上，也可以再次通过Shuffle Read读取得到相同的数据，避免再次执行上游stage中的task，所以，Spark根据ShuffleDependency切分出的stage既保证了task的独立性，也方便了错误容忍的重新计算。

Spark采用了lineage来统一对RDD的数据和计算依赖关系进行建模，使用回溯方法解决从哪里开始计算，以及计算什么的问题。

为了提高重新计算的效率，也为了更好的解决数据丢失问题，Spark采用了检查点（checkpoint）机制。该机制的核心思想是将计算过程汇总某些重要数据进行持久化，这样在再次之心时可以从检查点执行，从而减少重新计算的开销。

需要被checkpoint的RDD满足的特征是，RDD的数据依赖关系比较复杂且重新计算代价较高，如关联的数据过多，计算链过长，被多次重复使用。

checkpoint的目的是对重要数据进行持久化，在节点宕机时也能够恢复，因此需要可靠存储，另外，checkpoint的数据量可能很大，因此需要较大的存储空间，所以一般使用分布式文件系统来存储，比如HDFS或者Alluxio。

在Spark中，提供了spackContext.setCheckpointDir(directory)接口来设置checkpoint的存储路径，同时，提供了rdd.checkpoint来实现checkpoint。

用户设置rdd.checkpoint后只标记某个RDD需要持久化，计算过程也像正常一样计算，等到当前job计算结束时再重新启动该job计算一遍，对其中需要checkpoint的RDD进行持久化。也就是说，当前job结束后会另外启动专门的job去完成checkpoint，需要checkpoint的RDD会被计算两次。显然，checkpoint启动额外job来进行持久化会增加计算开销，为了解决这个问题，Spark推荐用户将需要被checkpoint的数据先进行缓存，这样额外启动的任务只需要将缓存数据进行checkpoint即可。

RDD需要经过Initialized -> checkpointingInProgress -> Checkpointed这三个阶段才能真正的被checkpoint

1. Initialized 当应用程序使用rdd.checkpoint设定某个RDD需要被checkpoint是，Spark为该RDD添加一个checkpointData属性，用来管理该RDD相关的checkpoint信息
2. CheckpointingInProgress 当前job结束后，会调用该job最后一个RDD的doCheckpoint方法，该方法根据finalRDD的computing chain回溯扫描，遇到需要被checkpoint的RDD就将其标记为CheckpointingInProgress。之后Spark会调用runJob再次提交一个job完成checkpoint
3. Checkpointed 再次提交的job对RDD完成checkpoint后，spark会建立一个新的newRDD，类型为ReliableCheckpointRDD，用于表示被checkpoint到磁盘上的RDD，和原先的RDD关联，并且切断RDD的lineage，数据已经进行了持久化，不再需要lineage。

当对某个RDD同时进行缓存和checkpoint时，会对其先进行缓存，然后再次启动job对其进行checkpoint。如果单纯是为了降低job lineage的复杂程度而不是为了持久化，Spark提供了localCheckpooint操作，功能上等价于数据缓存加上checkpoint切断lineage的功能。

**checkpint和数据缓存的区别**

1. 目的不同，数据缓存的目的是加速计算，即加速后续运行的job。而checkpint的目的是在job运行失败后能够快速回复，也就是加速当前需要重新运行的job
2. 存储性质和位置不同。数据缓存是为了读写速度快，因此主要使用内存，偶尔使用磁盘作为存储空间。而checkpoint是为了能够可靠读写，因此主要使用分布式文件系统作为存储空间
3. 写入速度和规则不同。数据缓存速度较快，对job的执行时间影响较小，因此可以在job运行时进行缓存，而checkpoint写入速度慢，为了减少对当前job的时延影响，会额外启动专门的job进行持久化
4. 对lineage的影响不同，对某个RDD进行缓存后，对该RDD的lineage没有影响，这样如果缓存后的RDD丢失还可以重新计算得到，而对某个RDD进行checkpoint以后，会切断该RDD的lineage，因为该RDD已经被可靠存储，所以不需要再保留该RDD是如何计算得到的。
5. 应用场景不同。数据缓存适用于会被多次读取，占用空间不是非常大的RDD，而checkpoint适用于数据依赖关系比较复杂，重新计算代价较高的RDD，如关联的数据过多、计算链过长、被多次重复使用等。



