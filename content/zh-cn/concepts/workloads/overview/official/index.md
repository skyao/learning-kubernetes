---
title: "官方文档"
linkTitle: "官方文档"
weight: 10
date: 2025-03-05
description: >
  Kubernetes 工作负载官方文档
---

- 中文： https://kubernetes.io/zh-cn/docs/concepts/workloads/
- 英文： https://kubernetes.io/docs/concepts/workloads/

工作负载是在 Kubernetes 上运行的应用程序。

在 Kubernetes 中，无论你的负载是由单个组件还是由多个一同工作的组件构成， 你都可以在一组 **Pod** 中运行它。 在 Kubernetes 中，Pod 代表的是集群上处于运行状态的一组 容器的集合。

Kubernetes Pod 遵循预定义的生命周期。 例如，当在你的集群中运行了某个 Pod，但是 Pod 所在的 节点 出现致命错误时， 所有该节点上的 Pod 的状态都会变成失败。Kubernetes 将这类失败视为最终状态： 即使该节点后来恢复正常运行，你也需要创建新的 Pod 以恢复应用。

不过，为了减轻用户的使用负担，通常不需要用户直接管理每个 `Pod`。 而是使用**负载资源**来替用户管理一组 Pod。 这些负载资源通过配置 控制器 来确保正确类型的、处于运行状态的 Pod 个数是正确的，与用户所指定的状态相一致。

Kubernetes 提供若干种内置的工作负载资源：

- Deployment 和 ReplicaSet （替换原来的资源 ReplicationController）。 Deployment 很适合用来管理你的集群上的无状态应用，Deployment 中的所有 Pod 都是相互等价的，并且在需要的时候被替换。
- StatefulSet 让你能够运行一个或者多个以某种方式跟踪应用状态的 Pod。 例如，如果你的负载会将数据作持久存储，你可以运行一个 StatefulSet，将每个 Pod 与某个 PersistentVolume 对应起来。你在 StatefulSet 中各个 Pod 内运行的代码可以将数据复制到同一 StatefulSet 中的其它 Pod 中以提高整体的服务可靠性。

- DaemonSet 定义提供节点本地支撑设施的 Pod。这些 Pod 可能对于你的集群的运维是 非常重要的，例如作为网络链接的辅助工具或者作为网络 插件 的一部分等等。每次你向集群中添加一个新节点时，如果该节点与某 `DaemonSet` 的规约匹配，则控制平面会为该 DaemonSet 调度一个 Pod 到该新节点上运行。
- Job 和 CronJob。 定义一些一直运行到结束并停止的任务。 你可以使用 Job 来定义只需要执行一次并且执行后即视为完成的任务。你可以使用 CronJob 来根据某个排期表来多次运行同一个 Job。

在庞大的 Kubernetes 生态系统中，你还可以找到一些提供额外操作的第三方工作负载相关的资源。 通过使用定制资源定义（CRD），你可以添加第三方工作负载资源，以完成原本不是 Kubernetes 核心功能的工作。 例如，如果你希望运行一组 Pod，但要求**所有** Pod 都可用时才执行操作 （比如针对某种高吞吐量的分布式任务），你可以基于定制资源实现一个能够满足这一需求的扩展， 并将其安装到集群中运行。

