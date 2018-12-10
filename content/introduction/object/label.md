---
date: 2018-12-08T09:00:00+08:00
title: Label和选择器
menu:
  main:
    parent: "introduction-object"
weight: 153
description : "Kubernetes对象的Label和选择器"
---

> 备注： 内容来自 [Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

标签是附加到对象（例如pod）的键/值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接影响核心系统的语义。 标签可用于组织和选择对象的子集。 标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。 每个Key对于给定对象必须是唯一的。

```json
"metadata": {
  "labels": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

我们最终将索引和反向索引标签以用于高效查询和监视，使用它们在UI和CLI中进行排序和分组等。我们不希望使用具有非标识，特别是大型和/或结构化的数据来污染标签。 非识别信息应使用 annotation 记录。 




