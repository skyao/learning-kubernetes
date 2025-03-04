---
title: "服务网络"
linkTitle: "服务网络"
weight: 600
date: 2021-02-01
description: >
  Kubernetes 服务网络
---

> https://kubernetes.io/zh-cn/docs/concepts/services-networking/

## Kubernetes 网络模型

Kubernetes 强制要求所有网络设施都满足以下基本要求（从而排除了有意隔离网络的策略）：

- Pod 能够与所有其他[节点](https://kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)上的 Pod 通信， 且不需要网络地址转译（NAT）
- 节点上的代理（比如：系统守护进程、kubelet）可以和节点上的所有 Pod 通信

Kubernetes 的 IP 地址存在于 `Pod` 范围内 —— 容器共享它们的网络命名空间 —— 包括它们的 IP 地址和 MAC 地址。 

Kubernetes 网络解决四方面的问题：

- Pod 中的容器之间[通过本地回路（loopback）通信](https://kubernetes.io/zh-cn/docs/concepts/services-networking/dns-pod-service/)。
- 集群网络在不同 Pod 之间提供通信。
- Service API 允许向外暴露 Pod 中运行的应用， 以支持来自于集群外部的访问。
  - [Ingress](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/) 提供专门用于暴露 HTTP 应用程序、网站和 API 的额外功能。
- 你也可以使用 Service 来[发布仅供集群内部使用的服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service-traffic-policy/)。

