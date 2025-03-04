---
title: "Verbs 和 Kinds"
linkTitle: "Verbs 和 Kinds"
weight: 30
date: 2021-02-01
description: >
  Verbs 和 Kinds
---

Kubernetes API为该规范添加了两个概念：Kubernetes API Verbs （动词）和Kubernetes Kinds。

Kubernetes API Verbs 被直接映射到 OpenAPI 规范中的操作。定义的 Verbs 动词是 get、create、update、patch、delete、list、watch 和d eletecollection。与HTTP动词的对应关系可以在表1-1中找到。

表1-1 Kubernetes API动词和HTTP动词的对应关系

| Kubernetes API Verb | HTTP Verb |
| :------------------ | :-------- |
| get                 | GET       |
| create              | POST      |
| update              | PUT       |
| patch               | PATCH     |
| delete              | DELETE    |
| list                | GET       |
| watch               | GET       |
| deletecollection    | DELETE    |

Kubernetes Kinds 是 OpenAPI 规范中定义的一个子集。当向 Kubernetes API 发出请求时，数据结构会通过请求和响应的主体进行交换。这些结构共享共同的字段，`apiVersion` 和 `kind`，以帮助请求的参与者识别这些结构。

如果你想让你的 User API 管理这个 Kind 概念，User 结构体将包含两个额外的字段，`apiVersion` 和 `kind` --例如，其值为 `v1` 和 `User`。要确定 Kubernetes OpenAPI 规范中的定义是否是 Kubernetes Kind，你可以查看定义的 `x-kubernetes-group-version-kind` 字段。如果这个字段被定义了，那么该定义就是一个 kind，它给你提供了`apiVersion` 和 `kind` 字段的值。
