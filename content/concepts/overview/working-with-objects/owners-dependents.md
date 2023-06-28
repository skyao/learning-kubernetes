---
title: "属主与附属"
linkTitle: "属主与附属"
weight: 80
date: 2021-02-01
description: >
  在 Kubernetes 中，一些对象是其他对象的“属主（Owner）”。
---

> https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/owners-dependents/



在 Kubernetes 中，一些对象是其他对象的“属主（Owner）”。 例如，ReplicaSet 是一组 Pod 的属主。 具有属主的对象是属主的“附属（Dependent）”。

属主关系不同于一些资源使用的标签和选择算符机制。 例如，有一个创建 EndpointSlice 对象的 Service， 该 Service 使用标签来让控制平面确定，哪些 EndpointSlice 对象属于该 Service。 除开标签，每个代表 Service 所管理的 EndpointSlice 都有一个属主引用。 属主引用避免 Kubernetes 的不同部分干扰到不受它们控制的对象。

## 对象规约中的属主引用

附属对象有一个 `metadata.ownerReferences` 字段，用于引用其属主对象。 一个有效的属主引用，包含与附属对象同在一个命名空间下的对象名称和一个 UID。 Kubernetes 自动为一些对象的附属资源设置属主引用的值， 这些对象包含 ReplicaSet、DaemonSet、Deployment、Job、CronJob、ReplicationController 等。 你也可以通过改变这个字段的值，来手动配置这些关系。 然而，通常不需要这么做，你可以让 Kubernetes 自动管理附属关系。

附属对象还有一个 `ownerReferences.blockOwnerDeletion` 字段，该字段使用布尔值， 用于控制特定的附属对象是否可以阻止垃圾收集删除其属主对象。 如果[控制器](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/)（例如 Deployment 控制器） 设置了 `metadata.ownerReferences` 字段的值，Kubernetes 会自动设置 `blockOwnerDeletion` 的值为 `true`。 你也可以手动设置 `blockOwnerDeletion` 字段的值，以控制哪些附属对象会阻止垃圾收集。

Kubernetes 准入控制器根据属主的删除权限控制用户访问，以便为附属资源更改此字段。 这种控制机制可防止未经授权的用户延迟属主对象的删除。

**说明：**

根据设计，kubernetes 不允许跨名字空间指定属主。 名字空间范围的附属可以指定集群范围的或者名字空间范围的属主。 名字空间范围的属主**必须**和该附属处于相同的名字空间。 如果名字空间范围的属主和附属不在相同的名字空间，那么该属主引用就会被认为是缺失的， 并且当附属的所有属主引用都被确认不再存在之后，该附属就会被删除。



### 笔记

dapr在创建 service 对象时，会同时创建 xxx 和 xxx-dapr 两个service，其中 xxx-dapr 就会有 ownerReferences 的设置，指向 xxx deployment：
```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    dapr.io/enabled: "true"
  name: service-a-dapr
  namespace: dapr-tests
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: Deployment
    name: service-a
    uid: b7ffded5-c8ab-4316-a535-5d29a6ceae18 # 对应到 owner deployment 的 uid
```

xxx deployment 的内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  generation: 1
  name: service-a
  namespace: dapr-tests
  uid: b7ffded5-c8ab-4316-a535-5d29a6ceae18
```

