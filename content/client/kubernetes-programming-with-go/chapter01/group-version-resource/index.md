---
title: "Group-Version-Resource"
linkTitle: "Group-Version-Resource"
weight: 30
date: 2021-02-01
description: >
  Group-Version-Resource
---

Kubernetes API 是 REST API，因此它管理资源和资源的路径，这些资源遵循REST的命名惯例--即使用复数名称来识别资源，并将这些资源分组。

因为 Kubernetes API 管理着数以百计的资源，所以它们被分组，而且因为 API 的发展，这些资源是有版本的。由于这些原因，每个资源都属于一个给定的 group和 version，每个资源都由 Group-Version-Resource 唯一标识，通常称为 GVR。

要找到 Kubernetes API 中的各种资源，你可以浏览 OpenAPI 规范，提取不同的路径。传统的资源（如pod或节点）将在 Kubernetes API 的早期引入，都属于组 core 和版本v1。

管理整个集群的遗留资源的路径遵循 `/api/v1/<plural_resource_name>` 的格式--例如，`/api/v1/nodes` 来管理节点。请注意，core group 在路径中不被代表。要管理特定命名空间中的资源，路径格式是 `/api/v1/namespaces/<namespace_name>/<plural_resource_name>` --例如，`/api/v1/namespaces/default/pods` 用于管理默认命名空间中的pod。

较新的资源可以通过格式为 `/apis/<group>/<version>/<plural_resource_name>`或 `/apis/<group>/<version>/namespaces/<namespace_name>/<plural_resource_name>` 的路径访问。

概括地说，获取资源的各种路径的格式是：

- `/api/v1/<plural_name>` - 访问传统的非命名空间的资源。例如：`/api/v1/nodes`，访问无命名空间的节点资源。或
- 访问整个集群的传统命名方式的资源，例如：`/api/v1/pods`，访问所有命名空间的pod。

- `/api/v1/namespaces/<ns>/<plural_name> `- 访问特定命名空间中的遗留带命名空间的资源。例如：`/api/v1/namespaces/default/pods`，访问默认命名空间中的pod。
- `/apis/<group>/<version>/<plural_name>` - 访问特定组和版本中的非命名空间资源。例如：`/apis/storage.k8s.io/v1/storageclasses`来访问非命名空间的storageclasses（组storage.k8s.io，版本v1）。或
- 访问整个集群的命名空间资源。例如：`/apis/apps/v1/deployments`，访问所有命名空间的 deployment。

- `/apis/<group>/<version>/namespaces/<ns>/<plural_name>` - 访问特定命名空间的命名资源。例如：`/apis/apps/v1/namespaces/default/deployments`访问默认命名空间中的deployment（分组为 apps，版本为 v1）。
