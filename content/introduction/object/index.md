---
date: 2018-12-08T09:00:00+08:00
title: Kubernetes对象
weight: 150
description : "Kubernetes对象"
---

> 备注： 内容来自 [Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

本页介绍Kubernetes对象如何在Kubernetes API中表示，以及如何以 `.yaml` 格式表达它们。

## 理解Kubernetes对象

*Kubernetes Objects* 是 Kubernetes 系统中的持久化实体。Kubernetes使用这些实体来表示集群的状态。具体来说，他们可以描述：

- 哪些容器化应用程序正在运行（以及在哪些节点上）
- 这些应用程序可用的资源
- 有关这些应用程序行为方式的策略，例如重新启动策略，升级和容错

Kubernetes 对象是一个“意图记录（record of intent）” - 一旦你创建了对象，Kubernetes系统将不断努力确保对象存在。通过创建一个对象，您可以有效地告诉 Kubernetes 系统您希望集群的工作负载的样子; 这是您的集群所需的状态。

要使用 Kubernetes 对象 - 无论是创建，修改还是删除它们 - 您需要使用 Kubernetes API。 例如，当您使用 kubectl 命令行界面时，CLI会为您进行必要的 Kubernetes API 调用。 您还可以在您自己的程序中使用 [客户端库](https://kubernetes.io/docs/reference/using-api/client-libraries/) 直接使用 Kubernetes API。

### 对象规范和状态

每个Kubernetes对象都包含两个嵌套对象字段，用于控制对象的配置：对象规范（object spec）和对象状态（object status）。您必须提供的规范（spec）描述了对象所需的状态 - 您希望对象具有的特征。状态（status）描述对象的实际状态，由Kubernetes系统提供和更新。在任何给定时间，Kubernetes控制平面都会主动管理对象的实际状态，以匹配您提供的所需状态。

例如，Kubernetes Deployment 是一个对象，可以表示在集群上运行的应用程序。创建 Deployment 时，您可以设置 Deployment spec 以指定您希望运行应用程序的三个副本。Kubernetes系统读取 Deployment spec 并启动所需应用程序的三个实例 - 更新状态以符合您的规范。如果这些实例中的任何一个失败（状态改变），Kubernetes系统通过进行校正来响应规范（spec）和状态（status）之间的差异 - 在这种情况下，启动替代实例。






