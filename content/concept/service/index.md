---
date: 2018-12-08T09:00:00+08:00
title: Service
weight: 330
description : "Kubernetes的service概念"
---

> 备注： 内容摘要自 [Services](https://kubernetes.io/docs/concepts/services-networking/service/)

Kubernetes `Service`是一个定义 `Pods` 的逻辑集合和访问它们的策略的抽象 - 有时称为`微服务`。`Service` 所指的目标 Pod 集合（通常）是通过 `Label Selector` 来确定。

Kubernetes `Services`的支持`TCP`，`UDP`以及`SCTP`对协议(SCTP支持是自Kubernetes 1.12的alpha功能)。默认是`TCP`。



