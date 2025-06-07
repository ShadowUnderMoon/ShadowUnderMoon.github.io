---
title: "Spark数据缓存"
author: "爱吃芒果"
description:
date: "2025-06-07T16:14:21+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
categories:
---

缓存机制实际上是一种空间换时间的方法，集体的，如果数据满足一下3条，就可以进行缓存.

1. 会被重复使用的数据。更确切地，会被多个job共享使用的数据。被共享使用的次数越多，那么缓存该数据的性价比越高。一般来说，对于迭代型和交互型应用非常适合。
2. 数据不宜过大。过大会占用大量的存储空间，导致内存不足，也会降低数据计算时可使用的空间。虽然缓存数据过大时也可以存放到磁盘，但磁盘的I/O代价比较高，有时甚至不如重新计算块。
3. 非重复缓存的数据。重复缓存的意思是如果缓存了某个RDD，那么该RDD通过OneToOneDependency连接的parent RDD就不需要被缓存了，除非有job不使用缓存的RDD，而直接使用parent RDD。

包含数据缓存操作的应用执行流程生成的规则：Spark首先假设应用没有数据缓存，正常生成逻辑处理流程（RDD之间的数据依赖关系），然后从第2个job开始，将cached RDD 之前的RDD都去掉，得到削减后的逻辑处理流程。最后，将逻辑处理流程转化为物理执行计划。

Spark从三个方面考虑了缓存级别（Storage Level），分别是存储外置、是否序列化存储、是否将缓存数据进行备份。缓存级别针对的是RDD中的所有分区，即对RDD中每个分区中的数据都进行缓存。对于MEMORY_ONLY级别来说，只使用内存进行缓存，如果某个分区在内存中存放不下，就不对该分区进行缓存。当后续job中的task计算需要这个分区中的数据时，需要重新计算得到该分区。

rdd.cache只是对RDD进行缓存标记，不是立即执行的，实际在action操作的job计算过程中进行缓存，当需要缓存的RDD中的record被计算出来时，及时进行缓存，再进行下一步操作。

在实现中，Spark在每个executor进行中分配一个区域，以进行数据缓存，该区域由BlockManager来管理。假设有两个task， task0和task1运行在同一个executor进程中，对于task0，当计算出partition0后，将partition0存放到BlockManager中的memoryStore内。memoryStore包含了一个LinkedHashMap，用于存储RDD的分区。该LinkedHashMap中的Key是blockId，即rddId + partitionId，Value是分区中的数据，LinkedHashMap基于双向链表。

如果需要访问缓存的分区，如果分区在本地，直接读取即可，否则需要通过远程访问，也就是通过getRemote读取，远程访问需要对数据进行序列化和反序列化，远程读取时是一条条record读取，并得到及时处理的。

Spark提供了通用的缓存操作rdd.persist和rdd.unpersist用来缓存和回收缓存数据，不管persist和unpersist都只能针对用户可见的RDD进行操作，Spark额外生成的rdd不能被用户操作。

Spark采用LRU替换算法，由于Spark每计算一个record就进行存储，因此在缓存结束前，Spark不能预知该RDD需要的存储空间，所以Spark采用动态替换策略，在当前可用内存空间不足时，每次通过LRU替换一个或多个RDD（具体数目与一个动态的阈值有关），如果替换掉所有旧的RDD都存不下新的RDD，那么需要分两种情况处理，如果新的RDD的存储级别包含磁盘，那么可以将新的RDD存放到磁盘，如果新的RDD的存储级别只是内存，那么就不存储该RDD，Spark直接利用LinkedHashMap自带的LRU功能实现缓存替换。此外，在进行缓存替换时，RDD的分区数据不能被该RDD的其他分区数据替换。

在Spark中可以通过unpersist主动回收缓存数据，不同于persit的延迟生效，unpersist操作是立即生效的，用户还可以设定unpersist是同步阻塞还是异步执行，如unpersist(blocking=true)表示同步阻塞，即程序需要等待unpersit结束后再进行下一步操作，这也是Spark的默认设定，而unpersist(blocking=false)表示异步执行，即边执行unpersist边进行下一步操作。

当前的缓存机制只能用在每个Spark应用内部，即缓存数据只能在job之间共享，不能在应用之间共享，Spark研究者后续开发了分布式内存文件系统Alluxio用来解决这个问题。



