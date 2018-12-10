---
date: 2018-12-08T09:00:00+08:00
title: Object Name
menu:
  main:
    parent: "introduction-object"
weight: 151
description : "Kubernetes对象的name属性"
---

> 备注： 内容来自 [Names](https://kubernetes.io/docs/concepts/overview/working-with-objects/names/)

Kubernetes REST API中的所有对象都由 Name 和 UID 明确标识。

对于用户提供的非唯一的属性，Kubernetes提供 label 和 annotation。

有关名称和UID的精确语法规则，请参阅 identifiers design doc](https://git.k8s.io/community/contributors/design-proposals/architecture/identifiers.md)。

### Name

客户端提供的字符串，用于引用资源URL中的对象，例如 `/api/v1/pods/some-name`。

对于给定名称，一次只能有一个给定类型的对象。 但是，如果删除该对象，则可以创建具有相同名称的新对象。

按照惯例，Kubernetes资源的名称最大长度应为253个字符，并由小写字母数字字符， `-` 和 `.` 组成，但某些资源具有更多特定限制。

### UID

Kubernetes系统生成的字符串，用于唯一标识对象。

在Kubernetes集群的整个生命周期中创建的每个对象都具有不同的UID。 它旨在区分类似实体的历史事件。




