---
title: "Kubernetes 对象概述"
linkTitle: "概述"
weight: 10
date: 2025-03-31
description: >
  本页说明了在 Kubernetes API 中是如何表示 Kubernetes 对象的， 以及如何使用 .yaml 格式的文件表示 Kubernetes 对象
---



## 理解 Kubernetes 对象

在 Kubernetes 系统中，**Kubernetes 对象**是持久化的实体。 

Kubernetes 对象是一种“意向表达（Record of Intent）”。一旦创建该对象， Kubernetes 系统将不断工作以确保该对象存在。通过创建对象，你本质上是在告知 Kubernetes 系统，你想要的集群工作负载状态看起来应是什么样子的， 这就是 Kubernetes 集群所谓的**期望状态（Desired State）**。

> 笔记：k8s 中处处都在体现申明式 API。

### 对象规约（Spec）与状态（Status）

`spec` 描述希望对象所具有的特征： **期望状态（Desired State）**。

`status` 描述对象的**当前状态（Current State）**

### 必需字段

要创建的 Kubernetes 对象需要配置的字段如下：

- `apiVersion` - 创建该对象所使用的 Kubernetes API 的版本
- `kind` - 想要创建的对象的类别
- `metadata` - 唯一标识对象的数据，包括 `name` 字符串、`UID` 和可选的 `namespace`
- `spec` - 期望的对象的状态
