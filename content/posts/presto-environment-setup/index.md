---
title: "Presto环境搭建"
author: "爱吃芒果"
description:
date: "2026-01-17T15:11:35+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
- Presto
- Trino
categories:
- Presto
- Trino
---

## 前言

Trino是从Presto分离出来的项目，在后面的文章中不会严格区分Presto和Trino，除非某些代码只在其中一个项目中存在，根据《OLAP引擎底层原理与设计实践》的推荐，后面基本会通过trino项目的v350版本为例分析presto的一些源码级的实现，希望能够比较系统地理解OLAP引擎的整体实现。

另外非常推荐阅读《OLAP引擎底层原理与设计实践》这本书。

## Presto集群的拓扑结构

在Presto集群中存在两种角色：

- 集群协调节点：负责集群的管理，以及查询任务的接收、SQL执行计划生成、优化，并将任务发布到不同的查询执行节点上，最终将结果返回给客户端
- 查询执行节点：只负责执行具体的查询任务

## 编译

推荐使用jdk11编译v350版本的trino代码，命令如下

```bash
./mvnw install -pl presto-main -am -DskipTests
./mvnw install -pl presto-tpcds -am -DskipTests # 编译需要的数据源模块
./mvnw install -pl presto-cli -am -DskipTests # 编译客户端模块
```

使用idea将项目中`presto-parser/target/generated-sources/antlr4`标记为`Generated Source Root`，这样antlr4生成的代码就可以被idea识别到。



## 单机调试环境搭建

在项目中创建`presto-server-main/etc/coordinator.properties`文件，文件内容如下

```properties
coordinator=true
node.id=ffffffff-ffff-ffff-ffff-ffffffff
node.environment=test
node.internal-address=localhost
http-server.http.port=8080

discovery-server.enabled=true
discovery.uri=http://localhost:8080

exchange.http-client.max-connections=1000
exchange.http-client.max-connections-per-server=1000
exchange.http-client.connect-timeout=1m
exchange.http-client.idle-timeout=1m

scheduler.http-client.max-connections=1000
scheduler.http-client.max-connections-per-server=1000
scheduler.http-client.connect-timeout=1m
scheduler.http-client.idle-timeout=1m

query.client.timeout=5m
query.min-expire-age=30m

plugin.bundles=\
  ../presto-tpcds/pom.xml

node-scheduler.include-coordinator=true
```

`node-scheduler.include-coordinator`指定为true表示集群协调节点可以作为查询执行节点，这样就不需要额外启动查询执行节点。

在idea中启动时，选择新建Applicatoin配置

VM启动参数如下

```bash
-ea
-XX:+UseG1GC
-XX:G1HeapRegionSize=32M
-XX:+UseGCOverheadLimit
-XX:+ExplicitGCInvokesConcurrent
-Xmx2G
-Dconfig=etc/coordinator.properties
-Dlog.levels-file=etc/log.properties
-Djdk.attach.allowAttachSelf=true
```

classpath为`presto-server-main`，启动类为`io.prestosql.server.PrestoServer`

## 集群调试环境搭建

集群调试环境会分别启动集群协调节点和查询执行节点：

将`coordinator.properties`文件中的`node-scheduler.include-coordinator`设置为`false`，集群协调节点不会执行具体的查询任务

每个查询执行节点的配置类似，比如，可以在项目中创建`presto-server-main/etc/worker1.properties`，内存如下

```properties
coordinator=false
node.id=3e6b1e41-9e8e-4ac6-92e4-cd3dfd3df7c9
node.environment=test
node.internal-address=localhost
http-server.http.port=8081

discovery.uri=http://localhost:8080

exchange.http-client.max-connections=1000
exchange.http-client.max-connections-per-server=1000
exchange.http-client.connect-timeout=1m
exchange.http-client.idle-timeout=1m

query.client.timeout=5m
query.min-expire-age=30m

plugin.bundles=\
  ../presto-tpcds/pom.xml

node-scheduler.include-coordinator=false

```

启动配置和协调节点类似，修改系统变量`-Dconfig=/etc/worker1.properties`，其余都不变。

## 启动成功

启动成功后可以访问`http://127.0.0.1:8080`，查看Presto的WebUI。   

运行`./presto-cli/target/presto-cli-350-executable.jar`可以打开客户端命令行，执行查询命令，比如：

```bash
presto> show catalogs;
 Catalog 
---------
 system  
 tpcds   
(2 rows)

Query 20260117_083946_00002_crpxr, FINISHED, 1 node
Splits: 19 total, 19 done (100.00%)
0.74 [0 rows, 0B] [0 rows/s, 0B/s]

presto> 

```

因为我们在配置文件中指定了tpcds数据源，所以这里面有两个数据源，Presto将自身的各种数据作为system数据源。









