---
title: "工作负载"
linkTitle: "工作负载"
weight: 500
date: 2021-02-01
description: >
  工作负载是在 Kubernetes 上运行的应用程序。
---

> https://kubernetes.io/zh-cn/docs/concepts/workloads/

在 Kubernetes 中，无论你的负载是由单个组件还是由多个一同工作的组件构成， 你都可以在一组 Pod 中运行它。 在 Kubernetes 中，Pod 代表的是集群上处于运行状态的一组 容器 的集合。

不过，为了减轻用户的使用负担，通常不需要用户直接管理每个 Pod。 而是使用负载资源来替用户管理一组 Pod。 这些负载资源通过配置 控制器 来确保正确类型的、处于运行状态的 Pod 个数是正确的，与用户所指定的状态相一致。

Kubernetes 提供若干种内置的工作负载资源：

- Deployment 和 ReplicaSet （替换原来的资源 ReplicationController）。 Deployment 很适合用来管理你的集群上的无状态应用，Deployment 中的所有 Pod 都是相互等价的，并且在需要的时候被替换。
- StatefulSet 让你能够运行一个或者多个以某种方式跟踪应用状态的 Pod。 例如，如果你的负载会将数据作持久存储，你可以运行一个 StatefulSet，将每个 Pod 与某个 PersistentVolume 对应起来。你在 StatefulSet 中各个 Pod 内运行的代码可以将数据复制到同一 StatefulSet 中的其它 Pod 中以提高整体的服务可靠性。
- DaemonSet 定义提供节点本地支撑设施的 Pod。这些 Pod 可能对于你的集群的运维是 非常重要的，例如作为网络链接的辅助工具或者作为网络 插件 的一部分等等。每次你向集群中添加一个新节点时，如果该节点与某 DaemonSet 的规约匹配，则控制平面会为该 DaemonSet 调度一个 Pod 到该新节点上运行。
- Job 和 CronJob。 定义一些一直运行到结束并停止的任务。Job 用来执行一次性任务，而 CronJob 用来执行的根据时间规划反复运行的任务。



![kubernetes-workloads](images/kubernetes-workloads.png)
