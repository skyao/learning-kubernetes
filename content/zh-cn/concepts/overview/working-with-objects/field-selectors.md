---
title: "字段选择器"
linkTitle: "字段选择器"
weight: 60
date: 2021-02-01
description: >
  字段选择器（Field selectors）允许你根据一个或多个资源字段的值筛选 Kubernetes 资源。 
---

> https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/field-selectors/

`kubectl` 命令将筛选出 [`status.phase`](https://kubernetes.io/zh-cn/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase) 字段值为 `Running` 的所有 Pod：

```shell
kubectl get pods --field-selector status.phase=Running
```

**说明：** 字段选择器本质上是资源“过滤器（Filters）”。默认情况下，字段选择器/过滤器是未被应用的， 这意味着指定类型的所有资源都会被筛选出来。 这使得以下的两个 `kubectl` 查询是等价的：

```bash
kubectl get pods
kubectl get pods --field-selector ""
```

## 链式选择器

同[标签](https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/labels/)和其他选择器一样， 字段选择器可以通过使用逗号分隔的列表组成一个选择链。 下面这个 `kubectl` 命令将筛选 `status.phase` 字段不等于 `Running` 同时 `spec.restartPolicy` 字段等于 `Always` 的所有 Pod：

```shell
kubectl get pods --field-selector=status.phase!=Running,spec.restartPolicy=Always
```

## 多种资源类型

你能够跨多种资源类型来使用字段选择器。 下面这个 `kubectl` 命令将筛选出所有不在 `default` 命名空间中的 StatefulSet 和 Service：

```bash
kubectl get statefulsets,services --all-namespaces --field-selector metadata.namespace!=default
```

> 笔记：原来还能这么用，汗，之前不知道，我都是一个一个查的。
