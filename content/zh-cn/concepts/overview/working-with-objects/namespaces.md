---
title: "名字空间"
linkTitle: "名字空间"
weight: 50
date: 2021-02-01
description: >
  名字空间
---

> https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/namespaces/

在 Kubernetes 中，**名字空间（Namespace）** 提供一种机制，将同一集群中的资源划分为相互隔离的组。

## 何时使用多个名字空间

名字空间适用于存在很多跨多个团队或项目的用户的场景。对于只有几到几十个用户的集群，根本不需要创建或考虑名字空间。当需要名字空间提供的功能时，请开始使用它们。

名字空间为名称提供了一个范围。资源的名称需要在名字空间内是唯一的，但不能跨名字空间。 名字空间不能相互嵌套，每个 Kubernetes 资源只能在一个名字空间中。

不必使用多个名字空间来分隔仅仅轻微不同的资源，例如同一软件的不同版本： 应该使用[标签](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/) 来区分同一名字空间中的不同资源。

## 使用名字空间

### 查看名字空间

Kubernetes 会创建四个初始名字空间：

- `default` 没有指明使用其它名字空间的对象所使用的默认名字空间
- `kube-system` Kubernetes 系统创建对象所使用的名字空间
- `kube-public` 这个名字空间是自动创建的，所有用户（包括未经过身份验证的用户）都可以读取它。 这个名字空间主要用于集群使用，以防某些资源在整个集群中应该是可见和可读的。 这个名字空间的公共方面只是一种约定，而不是要求。
- `kube-node-lease` 此名字空间用于与各个节点相关的 [租约（Lease）](https://kubernetes.io/docs/reference/kubernetes-api/cluster-resources/lease-v1/)对象。 节点租期允许 kubelet 发送[心跳](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/#heartbeats)，由此控制面能够检测到节点故障。

### 设置名字空间偏好

你可以永久保存名字空间，以用于对应上下文中所有后续 kubectl 命令。

```shell
kubectl config set-context --current --namespace=<名字空间名称>
# 验证
kubectl config view --minify | grep namespace:
```

> 笔记：这个功能很实用，有时长期使用一个命名空间，每次输入 -n xxxx 好烦。

## 并非所有对象都在名字空间中

大多数 kubernetes 资源（例如 Pod、Service、副本控制器等）都位于某些名字空间中。 但是名字空间资源本身并不在名字空间中。而且底层资源， 例如[节点](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)和持久化卷不属于任何名字空间。

查看哪些 Kubernetes 资源在名字空间中，哪些不在名字空间中：

```shell
# 位于名字空间中的资源
kubectl api-resources --namespaced=true

# 不在名字空间中的资源
kubectl api-resources --namespaced=false
```

