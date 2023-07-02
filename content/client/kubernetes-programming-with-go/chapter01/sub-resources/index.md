---
title: "子资源"
linkTitle: "子资源"
weight: 50
date: 2021-02-01
description: >
  子资源
---

按照REST API的惯例，资源可以有子资源。子资源是属于另一个资源的，可以通过在资源名称后面指定其名称来访问，如下所示：

- `/api/v1/<plural>/<res-name>/<sub-resource>`

  例如: /api/v1/nodes/node1/status

- `/api/v1/namespaces/<ns>/<plural>/<res-name>/<sub-resource>`

  例如: /api/v1/namespaces/ns1/pods/pod1/status

- `/apis/<group>/<version>/<plural>/<res-name>/<sub-resource>`

  例如: /apis/storage.k8s.io/v1/volumeattachments/volatt1/status

- `/apis/<grp>/<v>/namespaces/<ns>/<plural>/<name>/<sub-res>`

  例如: /apis/apps/v1/namespaces/ns1/deployments/dep1/status

大多数 Kubernetes 资源都有一个 status 子资源。你可以看到，在编写 operator 时，operator 需要更新 status 子资源，以便能够表明 operator 观察到的这个资源的状态。在status 子资源中可以执行的操作有：get、patch 和 update。Pod有更多的子资源，包括 attach, binding, eviction, exec, log, portforward, 和 proxy。这些子资源对于获取特定的正在运行的 pod 的信息，或者在正在运行的 pod 上执行一些特定的操作，等等都很有用。

可以缩放的资源（即 deployment 、replicaset 等）有一个 scale 子资源。可以在 scale 子资源中执行的操作是 get、patch和update。
