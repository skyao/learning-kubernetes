---
date: 2018-12-27T11:00:00+08:00
title: Deployment
menu:
  main:
    parent: "concept-controller"
weight: 323
description : "Kubernetes的Deployment"
---

Deployment 控制器为 pod 和 ReplicaSet 提供声明式更新。

在 Deployment 对象中描述了所需的状态，Deployment 控制器以受控速率将实际状态更改为所需状态。您可以定义“deployment”以创建新的ReplicaSet，或者删除现有的 Deployment 并使用新的 Deployment 接管所有资源。

> 注意：不应管理 Deployment 所拥有的 ReplicaSet。应该通过操作 Deployment 对象来涵盖所有用例。

### 用例

以下是 Deployment 的典型用例：

- 创建Deployment以部署ReplicaSet。ReplicaSet在后台创建Pod。检查rollout的状态以查看其是否成功。
- 通过更新 Deployment 的 PodTemplateSpec 来声明 Pod 的新状态。会创建一个新的ReplicaSet，Deployment 管理以受控速率将 Pod 从旧 ReplicaSet 移动到新 ReplicaSet。每个新的ReplicaSet 都会更新 Deployment 的修订版。
- 如果 Deployment 的当前状态不稳定，则回滚到早期的 Deployment 修订版。每次回滚都会更新 Deployment 的修订版。
- 扩展 Deployment 以承载更多负载。
- 暂停 Deployment 以将多个修复程序应用于其PodTemplateSpec，然后将其恢复以启动新的 Deployment。
- 使用 Deployment 的状态作为 rollout 卡住的指示。
- 清理不再需要的旧ReplicaSet。

### 参考资料

- 官方文档中的 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)