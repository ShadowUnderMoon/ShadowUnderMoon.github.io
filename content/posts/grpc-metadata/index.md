---
title: "gRPC元数据"
author: "爱吃芒果"
description:
date: "2025-08-02T17:15:19+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
    - "gRPC"
categories:
    - "gRPC"
---

Metadata是关于某个特定RPC调用的信息，grpc元数据通过HTTP/2 header实现，它以键值对的形式存在，其中键是字符串，值通常是字符串，但也可以是二进制数据。键不区分大小写，由ASCII字母、数字和特殊字符`-`、`_`、`.`组成，但不能以`grpc-`开头（该前缀保留给gRPC使用），二进制数据对应的key以`-bin`结尾，而ASCII类型的键不带`-bin`后缀。

grpc的metadata可以由客户端和服务端双方发送和接收，Headers（首部信息）：在RPC调用中，客户端在发送初始请求之前会将headers发送给服务端，而服务端在发送初始响应之前，也会将headers发送给客户端。Trailers（尾部信息）：当服务端关闭某个RPC时，会发送trailers给客户端。Headers是RPC开始时的metadata，类似于HTTP请求、响应的头部，Trailers是RPC结束时的metadata，常用于传递状态码或错误信息。grpc使用HTTP/2作为底层协议，这种机制借用了HTTP/2的header/trailer概念。

用户自定义的metadata不会被gRPC使用，这允许客户端向服务端提供与调用有关的信息，反之亦然。对metadata的访问方式取决于具体的编程语言。

元数据的使用场景：

- 验证证书
- 跟踪信息（tracing）
- 自定义header（负载均衡、速率控制、错误信息）
- 其他应用相关的信息，依赖于在rpc负载之前或者之后发送的数据

元数据的限制：

- 非流量控制，要求用户自定义的限制，GRPC_ARG_MAX_METADATA_SIZE控制单个请求被允许的元数据大小，如果请求的元数据大小超过限制，请求会被拒绝

**不一致的metadata限制:**请求的元数据大小小于服务器的限制，但却被拒绝，提供了`metadata limit exceeded`的错误信息。代理配置了一个更小的元数据限制，所以导致请求压根没有到服务器，直接被拒绝。通过为所有的代理配置全局的元数据限制，客户端和服务端都不应该超过这个限制，如果它们期望请求能够通过这些代理。

另外，如果代理新增一些元数据，也可能导致最终到达服务器的请求元数据超限，导致请求被拒绝。

**全部请求被拒绝：**如果新的metadata元素被添加到请求，恰好超过了对端允许的metadata大小限制，请求会被拒绝，如果某个客户端的所有请求都发生了这种情况，则所有请求都会被拒绝，在最坏情况下导致系统完全不可用。

![image-20250802175452930](image-20250802175452930.png)

引入两层限制，软限制和硬限制，在两个限制之间随机拒绝一些请求，随着元数据大小的增加请求被拒绝的概率提升，从而让客户有机会注意到增长的元数据超限报错，避免完全不可用。

要点：

- 保证元数据软限制和硬限制的一致性，并且意识到客户端和服务端之间元数据的限制，包括代理

- 监控增长的`RESOURCE_EXHAUSTED`错误，可能表示元数据超限
    ```java
    RESOURCE_EXHAUSTED: received metadata size exceeds soft limit (15000 vs 8192); :path:50B :authority:100B
    ```

- 使用GRPC_ARG_MAX_METADATA_SIZE设置软限制（默认8KB），GRPC_ARG_ABSOLUTE_MAX_METADATA_SIZE设置硬限制（默认16KB）

