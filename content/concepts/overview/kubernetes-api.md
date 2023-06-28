---
title: "Kubernetes API"
linkTitle: "API"
weight: 20
date: 2021-02-01
description: >
  Kubernetes API
---

> https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api/

Kubernetes 控制面的核心是 API 服务器。 API 服务器负责提供 HTTP API，以供用户、集群中的不同部分和集群外部组件相互通信。

Kubernetes API 使你可以查询和操纵 Kubernetes API 中对象（例如：Pod、Namespace、ConfigMap 和 Event）的状态。

大部分操作都可以通过 kubectl 命令行接口或类似 kubeadm 这类命令行工具来执行， 这些工具在背后也是调用 API。不过，你也可以使用 REST 调用来访问这些 API。



## OpenAPI 规范

### OpenAPI V2

Kubernetes API 服务器通过 `/openapi/v2` 端点提供聚合的 OpenAPI v2 规范。 

Kubernetes 为 API 实现了一种基于 Protobuf 的序列化格式，主要用于集群内部通信。 

### OpenAPI V3

**特性状态：** `Kubernetes v1.24 [beta]`

Kubernetes v1.25 提供将其 API 以 OpenAPI v3 形式发布的 beta 支持； 这一功能特性处于 beta 状态，默认被开启。

## API 变更

Kubernetes 对维护达到正式发布（GA）阶段的官方 API 的兼容性有着很强的承诺，通常这一 API 版本为 `v1`。

## API 扩展

有两种途径来扩展 Kubernetes API：

1. 你可以使用[自定义资源](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/api-extension/custom-resources/)来以声明式方式定义 API 服务器如何提供你所选择的资源 API。
2. 你也可以选择实现自己的[聚合层](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)来扩展 Kubernetes API。
