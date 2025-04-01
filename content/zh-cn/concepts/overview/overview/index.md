---
title: "概述"
linkTitle: "概述"
weight: 1
date: 2021-02-01
description: >
  Kubernetes 是一个可移植、可扩展的开源平台，用于管理容器化的工作负载和服务，方便进行声明式配置和自动化。Kubernetes 拥有一个庞大且快速增长的生态系统，其服务、支持和工具的使用范围广泛。
---

> https://kubernetes.io/zh-cn/docs/concepts/overview/
>
> https://kubernetes.io/docs/concepts/overview/

Kubernetes 这个名字源于希腊语，意为“舵手”或“飞行员”。K8s 这个缩写是因为 K 和 s 之间有 8 个字符的关系。

Google 在 2014 年开源了 Kubernetes 项目。 Kubernetes 建立在 Google 大规模运行生产工作负载十几年经验的基础上， 结合了社区中最优秀的想法和实践。

## 为什么需要 Kubernetes，它能做什么？

Kubernetes 为你提供了一个可弹性运行分布式系统的框架。 Kubernetes 会满足你的扩展要求、故障转移你的应用、提供部署模式等。

Kubernetes 为你提供：

- **服务发现和负载均衡**
- **存储编排**
- **自动部署和回滚**
- **自动完成装箱计算**
- **自我修复**
- **密钥与配置管理**
- **批处理执行** 
- **水平扩缩** 
- **IPv4/IPv6 双栈** 
- **为可扩展性设计** 

## Kubernetes 不是什么

Kubernetes 不是传统的、包罗万象的 PaaS（平台即服务）系统

Kubernetes：

- 不限制支持的应用程序类型。 
- 不部署源代码，也不构建你的应用程序。
- 不提供应用程序级别的服务作为内置服务
- 不是日志记录、监视或警报的解决方案。 
- 不提供也不要求配置用的语言、系统（例如 jsonnet）
- 不提供也不采用任何全面的机器配置、维护、管理或自我修复系统。
- 此外，Kubernetes 不仅仅是一个编排系统，实际上它消除了编排的需要。 



