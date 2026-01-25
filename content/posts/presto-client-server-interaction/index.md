---
title: "Presto客户端和服务区交互流程"
author: "爱吃芒果"
description:
date: "2026-01-25T21:34:59+08:00"
image:
math:
license:
hidden: false
comments: true
draft: false
tags:
- Presto
- Trino
- TODO
categories:
- Presto
- Trino
---

```java
StatementClientV1.advance
ExecutingStatementResource::getQueryResult
Query::getNextResult
```

