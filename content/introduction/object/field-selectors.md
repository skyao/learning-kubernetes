---
date: 2018-12-08T09:00:00+08:00
title: 字段选择器
menu:
  main:
    parent: "introduction-object"
weight: 255
description : "Kubernetes对象的字段选择器"
---

> 备注： 内容来自 [Field Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/field-selectors/)

字段选择器允许您根据一个或多个资源字段的值选择Kubernetes资源。 以下是一些字段选择器查询的示例：

- `metadata.name=my-service`
- `metadata.namespace!=default`
- `status.phase=Pending`

这个 kubectl 命令选择 status.phase 字段的值是 Running 的所有Pod：

```bash
$ kubectl get pods --field-selector status.phase=Running
```

字段选择器本质上是资源过滤器。 默认情况下，不应用选择器/过滤器，这意味着将选择指定类型的所有资源。 这使得以下 kubectl 查询等效：

```bash
$ kubectl get pods
$ kubectl get pods --field-selector ""
```

### 支持的字段

支持的字段选择器因Kubernetes资源类型而异。 所有资源类型都支持 metadata.name 和 metadata.namespace 字段。 使用不受支持的字段选择器会产生错误。 例如：

```bash
$ kubectl get ingress --field-selector foo.bar=baz
Error from server (BadRequest): Unable to find "ingresses" that match label selector "", field selector "foo.bar=baz": "foo.bar" is not a known field selector: only "metadata.name", "metadata.namespace"
```

### 支持的运算符

您可以将 `=`，`==` 和 `=` 运算符与字段选择器一起使用（`=` 和 `==` 表示相同的事情）。 例如，这个 kubectl 命令选择不在 `default` 命名空间中的所有 Kubernetes 服务：

```bash
$ kubectl get services --field-selector metadata.namespace!=default
```

### 链式选择器

与标签和其他选择器一样，字段选择器可以链接在一起作为逗号分隔列表。这个 `kubectl` 命令选择 `status.phase` 不等于 `Running` 并且 `spec.restartPolicy` 字段等于 `Always` 的所有Pod：

```bash
$ kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```

### 多资源类型

您可以跨多种资源类型使用字段选择器。 此kubectl命令选择不在默认命名空间中的所有Statefulsets和Services：

```bash
$ kubectl get statefulsets,services --field-selector metadata.namespace!=default
```

