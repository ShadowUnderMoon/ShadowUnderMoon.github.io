---
title: "Spark_optimizer"
author: "爱吃芒果"
description: 
date: 2025-04-05T22:05:09+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
tags:
  - "Spark"
  - "优化器"
categories:
  - "Spark"
  - "优化器"
---

## 遇到的问题

```java
Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/logging/log4j/core/Filter
    at java.base/java.lang.Class.forName0(Native Method)
    at java.base/java.lang.Class.forName(Class.java:578)
    at java.base/java.lang.Class.forName(Class.java:557)
    at org.apache.spark.util.SparkClassUtils.classForName(SparkClassUtils.scala:41)
    at org.apache.spark.util.SparkClassUtils.classForName$(SparkClassUtils.scala:36)
    at org.apache.spark.util.SparkClassUtils$.classForName(SparkClassUtils.scala:141)
    at org.apache.spark.sql.SparkSession$.lookupCompanion(SparkSession.scala:826)
    at org.apache.spark.sql.SparkSession$.CLASSIC_COMPANION$lzycompute(SparkSession.scala:816)
    at org.apache.spark.sql.SparkSession$.org$apache$spark$sql$SparkSession$$CLASSIC_COMPANION(SparkSession.scala:815)
    at org.apache.spark.sql.SparkSession$.$anonfun$DEFAULT_COMPANION$1(SparkSession.scala:820)
    at scala.util.Try$.apply(Try.scala:217)
    at org.apache.spark.sql.SparkSession$.org$apache$spark$sql$SparkSession$$DEFAULT_COMPANION(SparkSession.scala:820)
    at org.apache.spark.sql.SparkSession$Builder.<init>(SparkSession.scala:854)
    at org.apache.spark.sql.SparkSession$.builder(SparkSession.scala:833)
    at org.apache.spark.examples.SparkPi$.main(SparkPi.scala:28)
    at org.apache.spark.examples.SparkPi.main(SparkPi.scala)
Caused by: java.lang.ClassNotFoundException: org.apache.logging.log4j.core.Filter
    at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:641)
    at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:188)
    at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:528)
    ... 16 more
```

在idea `Run/Debug Configuration`中添加`Add dependencies with provided scope to classpath`

```java
org.apache.spark.SparkException: A master URL must be set in your configuration
    at org.apache.spark.SparkContext.<init>(SparkContext.scala:421)
    at org.apache.spark.SparkContext$.getOrCreate(SparkContext.scala:3062)
    at org.apache.spark.sql.classic.SparkSession$Builder.$anonfun$build$2(SparkSession.scala:911)
    at scala.Option.getOrElse(Option.scala:201)
    at org.apache.spark.sql.classic.SparkSession$Builder.build(SparkSession.scala:902)
    at org.apache.spark.sql.classic.SparkSession$Builder.getOrCreate(SparkSession.scala:931)
    at org.apache.spark.sql.classic.SparkSession$Builder.getOrCreate(SparkSession.scala:804)
    at org.apache.spark.sql.SparkSession$Builder.getOrCreate(SparkSession.scala:923)
    at org.apache.spark.examples.SparkPi$.main(SparkPi.scala:30)
    at org.apache.spark.examples.SparkPi.main(SparkPi.scala)
```

添加jvm参数`-Dspark.master=local`，本地运行

不知道为什么idea不能直接找到`parallelize`的定义而飘红，这里直接导入`SparkContext`并且通过`asInstanceOf[SparkContext]`明示idea。

```java
// scalastyle:off println
package org.apache.spark.examples

import scala.math.random

import org.apache.spark.SparkContext
import org.apache.spark.sql.SparkSession

/** Computes an approximation to pi */
object SparkPi {
  def main(args: Array[String]): Unit = {
    val spark = SparkSession
      .builder()
      .appName("Spark Pi")
      .getOrCreate()
    val slices = if (args.length > 0) args(0).toInt else 2
    val n = math.min(100000L * slices, Int.MaxValue).toInt // avoid overflow
    val count = spark.sparkContext.asInstanceOf[SparkContext].parallelize(
      1 until n, slices).map { i =>
      val x = random() * 2 - 1
      val y = random() * 2 - 1
      if (x*x + y*y <= 1) 1 else 0
    }.reduce(_ + _)
    println(s"Pi is roughly ${4.0 * count / (n - 1)}")
    spark.stop()
  }
}
```
