---
title: "Ingress"
linkTitle: "Ingress"
weight: 20
date: 2021-02-01
description: >
  Kubernetes Ingress
---

> https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/

Ingress 是对集群中服务的外部访问进行管理的 API 对象

## 术语

- 节点（Node）: Kubernetes 集群中的一台工作机器，是集群的一部分。
- 集群（Cluster）: 一组运行由 Kubernetes 管理的容器化应用程序的节点。 在此示例和在大多数常见的 Kubernetes 部署环境中，集群中的节点都不在公共网络中。
- 边缘路由器（Edge Router）: 在集群中强制执行防火墙策略的路由器。可以是由云提供商管理的网关，也可以是物理硬件。
- 集群网络（Cluster Network）: 一组逻辑的或物理的连接，根据 Kubernetes [网络模型](https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/networking/)在集群内实现通信。
- 服务（Service）：Kubernetes [服务（Service）](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)， 使用[标签](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)选择器（selectors）辨认一组 Pod。 除非另有说明，否则假定服务只具有在集群网络中可路由的虚拟 IP。

## Ingress 是什么？

[Ingress](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.25/#ingress-v1beta1-networking-k8s-io) 暴露从集群外部到集群内[服务](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/)的 HTTP 和 HTTPS 路由。 流量路由由 Ingress 资源上定义的规则控制。

下面是一个将所有流量都发送到同一 Service 的简单 Ingress 示例：

[![ingress-diagram](https://d33wubrfki0l68.cloudfront.net/4f01eaec32889ff16ee255e97822b6d165b633f0/a54b4/zh-cn/docs/images/ingress.svg)](https://mermaid.live/edit#pako:eNqNkktLAzEQgP9KSC8Ku6XWBxKlJz0IHsQeuz1kN7M2uC-SrA9sb6X26MFLFZGKoCC0CIIn_Td1139halZq8eJlE2a--TI7yRn2YgaYYCc6EDRpod39DSdCyAs4RGqhMRndffRfs6dxc9Euox0NgZR2NhpmF73sqos2XVFD-ctt_vY2uTnPh8PJ4BGV7Ro3ZKOoaH5Li6Bt19r56zi7fM4fupP-oC1BHHEPGnWzGlimruno87qXvd__qjdpw2pXErOlxl7Mmn_j1VkcImb-i0q5BT5KAsoj5PMgICXGmCWViA-BlHzfL_b2MWeqRVaSE8uLg1iQUqVS2ZiTHK7LQrFcXfNg9V8WnZu3eEEqFYjCNCslJdd15zXVmcacODP9TMcqJmBN5zL9VKdt_uLM1ZoBzIVNF8WqM06ELRyCCCln-oWcTVkHqxaE4GCitwx8mgbK0Y-no9E0YVTBNuMqFpj4NJBgYZqquH4aeZgokcIPtMWpvtywoDpfU3_yww)



Ingress 不会公开任意端口或协议。 将 HTTP 和 HTTPS 以外的服务公开到 Internet 时，通常使用 [Service.Type=NodePort](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#type-nodeport) 或 [Service.Type=LoadBalancer](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#loadbalancer) 类型的 Service。



