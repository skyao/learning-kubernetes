---
date: 2018-12-08T09:00:00+08:00
title: ReplicaSet
menu:
  main:
    parent: "concept-controller"
weight: 321
description : "Kubernetes的ReplicaSet controller"
---



### 和Replication Controller的关系

ReplicaSet是下一代副本控制器。

现在 ReplicaSet 和 Replication Controller 之间的唯一区别是选择器支持：

- ReplicaSet 支持[标签用户指南中](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)描述的新的 set-based 的选择器
- Replication Controller 仅支持基于 equality-based 选择器要求

大多数 kubectl 支持的 Replication Controller 的命令也支持 ReplicaSet。rolling-update 命令是一个例外 。如果您想要滚动更新功能，请考虑使用 Deployment。此外， rolling-update 命令是必需的，而 Deployments 是声明性的，因此我们建议通过 rollout 命令使用 Deployments 。

### 和Deployment的关系

虽然 ReplicaSet 可以独立使用，但是它主要是 Deployment 用作协调pod创建，删除和更新的机制。使用“Deployment”时，不必担心管理它们创建的副本集。Deployment 拥有并管理其 ReplicaSet。

这实际上意味着您可能永远不需要操作 ReplicaSet 对象：改为使用 Deployment，并在spec部分中定义您的应用程序。

### 和HPA的关系

ReplicaSet 也可以是 Horizontal Pod Autoscalers（HPA）的目标 。也就是说，HPA可以自动调整ReplicaSet。

### 参考资料

- 官方文档中的 [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)