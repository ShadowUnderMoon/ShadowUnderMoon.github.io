---
title: "K8s_components"
author: "爱吃芒果"
description: 
date: 2025-02-07T20:59:49+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: true
---

## Pod

- Smallest unit of k8s
- abstraction over container
- usually 1 application per pod
- each pod gets its own IP Address
- New IP address on re-creation

## Service

- permanent IP address
- load banlance
- internal service / external service (no nodeport in internal service)

## Ingress

- route traffic into the cluster
- forward request to the matching service 

**Ingress controller**

- evaluates all the rules
- manages redirections
- entrypoint to cluster
- many third-party implementations
- k8s nginx ingress controller

cloud load balancer 负责将流量导入到k8s集群中

当应用一个ingress rule后，会返回对应的地址，这个地址可以作为k8s的入口

## ConfigMap

- external configuration of your application

## Secret

- used to store secret data
- base64 encoded

## Volumes

## Deployment

- blueprint for my-app pods
- abstraction of pods



## Stateful set





