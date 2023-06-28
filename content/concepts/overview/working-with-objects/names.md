---
title: "对象名称和 ID"
linkTitle: "对象名称和 ID"
weight: 30
date: 2021-02-01
description: >
  对象名称和 ID
---

> https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/names/

集群中的每一个对象都有一个名称来标识在同类资源中的唯一性。

每个 Kubernetes 对象也有一个 UID 来标识在整个集群中的唯一性。

比如，在同一个名字空间 中只能有一个名为 myapp-1234 的 Pod，但是可以命名一个 Pod 和一个 Deployment 同为 myapp-1234。

对于用户提供的非唯一性的属性，Kubernetes 提供了 标签（Labels）和 注解（Annotation）机制。



## 名称

客户端提供的字符串，引用资源 URL 中的对象，如`/api/v1/pods/some name`。

某一时刻，只能有一个给定类型的对象具有给定的名称。但是，如果删除该对象，则可以创建同名的新对象。

**说明：**

当对象所代表的是一个物理实体（例如代表一台物理主机的 Node）时， 如果在 Node 对象未被删除并重建的条件下，重新创建了同名的物理主机， 则 Kubernetes 会将新的主机看作是老的主机，这可能会带来某种不一致性。

### DNS 子域名

很多资源类型需要可以用作 DNS 子域名的名称。

### RFC 1123 标签名

某些资源类型需要其名称遵循 [RFC 1123](https://tools.ietf.org/html/rfc1123) 所定义的 DNS 标签标准。

### RFC 1035 标签名

某些资源类型需要其名称遵循 [RFC 1035](https://tools.ietf.org/html/rfc1035) 所定义的 DNS 标签标准。

### 路径分段名称

某些资源类型要求名称能被安全地用作路径中的片段。 换句话说，其名称不能是 `.`、`..`，也不可以包含 `/` 或 `%` 这些字符。

## UID

Kubernetes 系统生成的字符串，唯一标识对象。

在 Kubernetes 集群的整个生命周期中创建的每个对象都有一个不同的 UID，它旨在区分类似实体的历史事件。

Kubernetes UID 是全局唯一标识符（也叫 UUID）。 UUID 是标准化的，见 ISO/IEC 9834-8 和 ITU-T X.667。

## 深入学习

### Kubernetes 标识符和名称的设计文档

> https://github.com/kubernetes/design-proposals-archive/blob/main/architecture/identifiers.md

对Kubernetes中标识符的目标和建议进行了总结。

#### 定义

**UID**：一个非空的、不透明的、系统生成的值，保证在时间和空间上是唯一的；旨在区分类似实体的历史出现。

**name**: 一个非空的字符串，保证在特定时间的特定范围内是唯一的；在资源URL中使用；由客户在创建时提供，并鼓励对人友好；旨在促进单子对象的创建同位性和空间唯一性，区分不同的实体，并在操作中引用特定的实体。

rfc1035/rfc1123 `label`（DNS_LABEL）。一个字母数字（a-z和0-9）字符串，最大长度为63个字符，除第一个或最后一个字符外，任何地方都允许使用'-'字符，适合作为主机名或域名中的段。

rfc1035/rfc1123 `subdomain`（DNS_SUBDOMAIN）。一个或多个小写的rfc1035/rfc1123标签，用'.'分隔，最大长度为253个字符。

rfc4122 通用唯一标识符（universally unique identifier / UUID）。一个128位的生成值，跨时空碰撞的可能性极低，不需要中央协调。

rfc6335 `port name`（IANA_SVC_NAME）。一个字母数字（a-z和0-9）字符串，最大长度为15个字符，除了第一个或最后一个字符或与另一个'-'字符相邻外，允许在任何地方使用'-'字符，它必须至少包含一个（a-z）字符。

#### 名称和UID的目标

1. 跨越空间和时间唯一地识别（通过UID）一个对象。
2. 跨越空间唯一地命名（通过 name）一个对象。
3. 在API操作和/或配置文件中提供人性化的名称。
4. 允许幂等的创建API资源，并执行单子对象的空间唯一性。
5. 允许为某些对象自动生成DNS名称。

#### 一般设计

1. 当通过API创建一个对象时，必须指定一个Name字符串（一个`DNS_SUBDOMAIN`）。名称必须是非空的，并且在 apiserver 中是唯一的。这可以实现幂等和空间唯一的创建操作。系统的某些部分（如副本控制器）可以连接字符串（如基本名称和随机后缀）来创建一个唯一的Name。对于生成名字不切实际的情况，一些或所有对象可以支持一个参数来自动生成名字。生成随机的名字会破坏同位素。

   例子。"guestbook.user", "backend-x4eb1"

2. 当一个对象通过API被创建时，可以指定一个命名空间字符串（`DNS_LABEL`）。根据API接收器，命名空间可能被验证（例如，apiserver可能确保命名空间实际存在）。如果没有指定命名空间，API接收器将分配一个命名空间。这种分配策略在不同的API接收器中可能有所不同（例如，apiserver可能有一个默认值，kubelet可能会生成一些半随机的东西）。

   例如。"api.k8s.example.com"

3. 在通过API接受一个对象时，该对象被分配了一个UID（一个UUID）。UID必须是非空的，并且是跨空间和时间的唯一。

   例如。"01234567-89ab-cdef-0123-456789abcdef"



