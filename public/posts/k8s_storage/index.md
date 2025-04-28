# k8s 存储


## Pods

Pod 是 Kubernetes 中最小的可部署计算单元，你可以创建和管理它们。

**Pod（类似于一群鲸鱼的“pod”或豌豆荚“pea pod”）** 是一组一个或多个容器，这些容器共享存储和网络资源，并且有一个规范来定义如何运行它们。Pod 内部的内容始终是 **共同调度（co-scheduled）并在相同的上下文中运行** 的。Pod 充当一个特定应用的“**逻辑主机**”（logical host）：它包含一个或多个 **相对紧密耦合的应用容器**。在非云环境下，运行在同一台物理机或虚拟机上的应用程序，可以类比于在 Kubernetes 中运行在同一逻辑主机上的应用。

除了应用容器之外，Pod 还可以包含 **Init 容器**（init containers），这些容器在 Pod 启动时运行。此外，你还可以 **注入临时容器（ephemeral containers）** 来调试正在运行的 Pod。

## Volumes

k8s Volumes 为pod中的容器提供了通过文件系统访问和共享数据的方式，数据共享可以在一个容器中的不同进程或者容器间甚至不同的pod。

volume能够解决数据的持久化以及共享存储的问题。

k8s支持多种volumes，pod可以同时使用任意数量的不同类型volume，**Ephemeral volume** 的生命周期和pod相同，**persistent volumes** 可以超出一个pod的生命周期。当pod挂掉时，K8s会摧毁 **ephemeral volume** 但不会摧毁 **persistent volume** 。对于在给定pod中的任意类型的volume，数据在容器重启时都会被保留。

本质上，卷（Volume）是一个目录，其中可能包含一些数据，并且可供 Pod 内的容器访问。该目录如何创建、由何种存储介质支持以及其内容，取决于所使用的特定卷类型。

为了使用一个卷，声明要被提供给pod的卷在`.spec.volumes`下，声明在容器的哪里挂载这些卷在`spec.containers[*].volumeMounts`中。

当一个pod被启动时，容器中的进程看到的文件系统视图有两部分组成，一部分是容器镜像的初始内容，另一部分是挂载到容器中的卷。对于pod中的每个容器，需要独立的声明不同容器的挂载点。

## Storage Classes

StorageClass 为管理员提供了一种描述其提供的存储类别的方法。不同的存储类别可能对应不同的 **服务质量（QoS）级别**、**备份策略**，或者由集群管理员自定义的其他策略。Kubernetes 本身并不对这些存储类别的具体含义做任何规定。

每个StorageClass 包含字段 `provisioner`, `parameters`和`raclaimPolicy`，当一个属于某个storage class 的persistent volume需要被动态提供给 persistent volume claim时被使用。

Storage Class的名字非常重要，用户通过名字请求某类存储，管理员在创建storage class对象是设置名字以及类别的其他参数。

作为管理员，你可以声明一个默认的storage class用于没有指定类别的任何PVC。

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: low-latency
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: csi-driver.example-vendor.example
reclaimPolicy: Retain # default value is Delete
allowVolumeExpansion: true
mountOptions:
  - discard # this might enable UNMAP / TRIM at the block storage layer
volumeBindingMode: WaitForFirstConsumer
parameters:
  guaranteedReadWriteLatency: "true" # provider-specific

```

## Persistent Volumes

**PersistentVolume (PV)** 是集群中的一块存储，可以由管理员提供或者通过 **Storage Classes**动态提供。

**PersistentVolumeClaim (PVC)**是用户对存储的青秀区，类似于pod，pod消费节点资源而PVCs消费PV资源

有两种方式提供PVs：

- 静态：管理员直接创建PV
- 动态：当没有静态PV满足PVC，集群可能尝试动态的提供卷，PVC必须要求Storage Class，管理员必须创建并且配置storage class，Storage Class `""`关闭动态获取卷

Pod使用PVC作为卷，集群检查PVC得到对应的卷并将卷绑定到pod。

### 回收策略

当用户使用完卷后，可以将PVC对象删除从而允许资源的回收，PV的回收策略告诉集群当卷被释放后应该怎么样处理卷，目前有三种策略：`Retained`, `Recycled`和 `Deleted`。

这里只介绍`Retain`策略：

`Retain`回收策略允许手动的资源回收，当PVC被删除时， PV依然存在，卷被认为是释放状态，但它并不能够被另一个PVC请求直接使用因为前任的数据还在上面。管理员可以通过以下方式手动回收卷：

1. 删除PV
2. 手动清理数据
3. 手动删除对应的storage asset

如果想要重用相同的storage asset，使用相同的storage asset definition 创建一个新的PV

### PV的类型

PV类型作为插件实现，这里给出k8s支持的一些插件：

- csi : Container Storage Interface
- local: 挂载在节点上的本地存储设备

---

每个PV包含一个规范(spec) 和状态 (status)，PV对象的名字必须是一个有效的 DNS subdomain name，这意味着

- 包含不超过253个字符
- 只包含小写字符、数字、`-`或者`.`
- 字符或者数字开头
- 字符或者数字结尾

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv0003
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /tmp
    server: 172.17.0.2
```

访问模式 **ReadWriteOnce**，卷可以被挂载为单个节点可读写，ReadWriteOnce仍然允许多个pod访问，只要这些pod在相同的节点上。对于单pod访问，可以使用**ReadWriteOncePod**。

节点亲和度 **Node Affinity**，对于大多数卷类型，不需要设置这个字段，对于`local`卷需要显示设置这个字段。

一个PV可以声明节点亲和度来限制在那个节点上这个卷可以被访问，使用某个PV的Pod将只会被调度到满足节点亲和度的节点上。

### PersistentVolumeClaims

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 8Gi
  storageClassName: slow
  selector:
    matchLabels:
      release: "stable"
    matchExpressions:
      - {key: environment, operator: In, values: [dev]}
```



## 参考文献

1. https://kubernetes.io/docs/concepts/storage/persistent-volumes/
