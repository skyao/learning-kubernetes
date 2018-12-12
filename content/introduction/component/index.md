---
date: 2018-12-08T09:00:00+08:00
title: 组件
weight: 230
description : "Kubernetes组件"
---

本文档概述了 Kubernetes 所需的各种二进制组件, 用于提供齐全的功能。

- Master组件
- Node组件
- 扩展（Addons）

补充一个图：

![](images/k8s-components.png)

## Master组件

Master组件提供群集的控制平面。Master组件做群集全局决策（例如，调度），以及检测和响应群集事件（当副本控制器的'replicas'字段不满足时，启动新的pod）。

Master组件可以在群集中的任何计算机上运行。但是，为简单起见，设置脚本通常会在同一台计算机上启动所有Master组件，并且不在此计算机上运行用户容器。

### kube-apiserver

Master上暴露Kubernetes API的组件。它是Kubernetes控制面板的前端。

它旨在水平扩展 - 也就是说，它通过部署更多实例来扩展。 

#### etcd

一致且高度可用的键值存储，用作Kubernetes的所有集群数据的存储。

始终为Kubernetes集群提供etcd数据的备份计划。

### kube-scheduler

Master上的组件，用于监视新创建而又未分配节点的的pod，并选择一个节点供其运行。

调度决策所考虑的因素包括个人和集体资源需求，硬件/软件/策略约束，亲和性和反亲和性规范，数据位置，工作负载间的干涉和最后期限。

### kube-controller-manager

Master上运行控制器的组件。

从逻辑上讲，每个控制器都是一个单独的过程，但为了降低复杂性，它们都被编译成单个二进制文件并在单个进程中运行。

这些控制器包括：

- 节点控制器(Node Controller): 负责在节点出现故障时注意和响应。
- 副本控制器(Replication Controller): 负责为系统中的每个副本控制器对象维护正确数量的pod。
- 端点控制器(Endpoints Controller): 填充端点对象(即连接 Services & Pods)。
- 服务帐户(Service Account)和令牌(Token)控制器: 为新的命名空间创建默认帐户和 API 访问令牌。

### cloud-controller-manager

云控制器管理器是用于与底层云提供商交互的控制器。云控制器管理器二进制是 Kubernetes v1.6 版本中引入的 Alpha 功能。

云控制器管理器仅运行云提供商特定的控制器循环。您必须在 kube-controller-manager 中禁用这些控制器循环，您可以通过在启动 kube-controller-manager 时将 --cloud-provider 标志设置为external来禁用控制器循环。

云控制器管理器允许云供应商代码和 Kubernetes 核心彼此独立发展，在以前的版本中，Kubernetes 核心代码依赖于云提供商特定的功能代码。在未来的版本中，云供应商的特定代码应由云供应商自己维护，并与运行 Kubernetes 的云控制器管理器相关联。

以下控制器具有云提供商依赖关系:

- 节点控制器（Node Controller）: 用于检查云提供商以确定节点在云中停止响应后是否被删除
- 路由控制器（Route Controller）: 用于在底层云基础架构中设置路由
- 服务控制器（Service Controller）: 用于创建，更新和删除云提供商负载均衡器
- 数据卷控制器（Volume Controller）: 用于创建，附加和装载卷，并与云提供商进行交互以协调卷

## Node组件

节点组件在每个节点上运行，维护运行的 Pod 并提供 Kubernetes 运行时环境。

### kubelet

在集群中的每个节点上运行的代理。它确保容器在pod中运行。

kubelet采用一组PodSpecs，这些PodSpecs是通过各种机制提供的。kubelet确保这些PodSpecs中描述的容器运行且健康。 kubelet不管理不是由Kubernetes创建的容器。

### kube-proxy

kube-proxy通过维护主机上的网络规则并执行连接转发来实现Kubernetes服务抽象。

### Container Runtime

容器运行时是负责运行容器的软件。 Kubernetes支持多种运行时：Docker，rkt，runc和任何OCI运行时规范实现。

## Addons

扩展（addons）是实现集群功能的pod和服务。可以通过 Deployments，ReplicationControllers 等管理 pod。 Namespaced 扩展对象在 kube-system 命名空间中创建。

### DNS

虽然其他插件并非严格要求，但所有Kubernetes集群都应具有集群DNS，因为许多示例都依赖于它。

Cluster DNS是为Kubernetes服务提供DNS记录的DNS服务器，除了环境中的其他DNS服务器之外。

Kubernetes启动的容器会在DNS搜索中自动包含此DNS服务器。

### Web UI (Dashboard)

Dashboard是用于Kubernetes集群的基于Web的通用UI。 允许用户管理和解决集群中运行的应用程序以及集群本身。

### Container Resource Monitoring

Container Resource Monitoring在中央数据库中记录关于容器的通用时间序列度量，并提供用于浏览该数据的UI。

### Cluster-level Logging

Cluster-level Logging机制负责将容器日志保存到具有搜索/浏览界面的中央日志存储。

### 参考资料

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/): 官方文档的介绍

