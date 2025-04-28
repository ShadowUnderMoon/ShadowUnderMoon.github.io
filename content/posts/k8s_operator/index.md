---
title: "K8s_operator"
author: "爱吃芒果"
description: 
date: 2025-02-06T22:57:07+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: true
categories:
  - kubernetes

---



k8s operator主要用于 有状态服务

## 背景

对于无状态服务，不需要手动干预，k8s默认的控制循环能够自动化运维工作，比如创建、更新、重启、删除等操作

然而对于有状态服务，同种服务的不同服务实例本身是不同的，k8s不同自动化运维工作

## k8s operator的作用

根据前面的叙述，对于有状态服务，默认情况下需要依赖于手动的运维

然而我们使用k8s正是为了实现运维的自动化

通过引入 k8s operator相当于实现了具有服务领域知识的控制循环过程

k8s operator是由社区或者厂商提供的，他们是最了解在不同事件发生时应该做什么的人员



https://www.youtube.com/watch?v=ha3LjlD6g7g