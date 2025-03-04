---
title: "controller-runtime简介"
linkTitle: "简介"
weight: 10
date: 2021-02-01
description: >
  controller-runtime简介
---

正如你在第1章中所看到的，Controller Manager  是 Kubernetes 架构的一个重要部分。它嵌入了几个 Controller，每个控制器的作用是观察特定高层资源（Deployments 等）的实例，并使用低层资源（Pod等）来实现这些高层实例。

举例，Kubernetes 用户在部署无状态的应用程序时可以创建 Deployment。这个 Deployment 定义了 Pod 模板，用来在集群中创建 Pod 以及一些规范。以下是最重要的规格：

- 副本的数量：控制器必须为 Deployment 实例部署相同的 Pod。
- 部署策略：当 Pod 模板更新时，Pod 被替换的方式。

- 默认策略：滚动更新（Rolling Update）可用于在不中断服务的情况下将应用程序更新到新版本，它接受各种参数。还存在一个更简单的策略：Recreate，它将首先停止Pod，然后再开始替换它。


Deployment 控制器将创建 ReplicaSet 资源的实例，每个新版本的 Pod 模板都有一个，并更新这些 ReplicaSets 的副本数量，以满足副本和策略规范。你会发现退役版本的 ReplicaSet 的副本数为零，而实际应用版本的 ReplicaSet 的副本数为正数。在滚动更新期间，两个 ReplicaSets 将拥有正数的副本（新的副本数量增加，以前的副本数量减少），以便能够在两个版本之间过渡而不中断服务。

在它这边，ReplicaSet 控制器将负责维护由 Deployment 控制器创建的每个 ReplicaSet 实例所要求的 Pod 副本数量。

第8章已经表明，可以定义新的 Kubernetes 资源来扩展 Kubernetes 的 API。即使有原生控制器管理器运行控制器来处理原生 Kubernetes 资源，你也需要编写控制器来处理自定义资源。

一般来说，这样的控制器被称为 operator，处理第三方资源，并为处理原生 Kubernetes 资源的控制器保留 Controller 这个名字。

Client-go 库提供了使用 Go 语言编写控制器和 operator 的工具，controller-runtime 库利用这些工具提供围绕控制器模式的抽象，帮助你编写 operator 。这个库可以用下面的命令来安装：

```bash
go get sigs.k8s.io/controller-runtime@v0.13.0
```

你可以从 `github.com/kubernetes-sigs/controller-runtime/releases` 的源码库中获得可用的修订版。
