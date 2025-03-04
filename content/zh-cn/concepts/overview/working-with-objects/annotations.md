---
title: "注解"
linkTitle: "注解"
weight: 60
date: 2021-02-01
description: >
  使用 Kubernetes 注解为对象附加任意的非标识的元数据
---

> https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/annotations/

你可以使用 Kubernetes 注解为对象附加任意的非标识的元数据。

## 为对象附加元数据

你可以使用标签或注解将元数据附加到 Kubernetes 对象。

-  标签可以用来选择对象和查找满足某些条件的对象集合。 

- 相反，注解不用于标识和选择对象。 

注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符。

注解和标签一样，是键/值对：

```json
"metadata": {
  "annotations": {
    "key1" : "value1",
    "key2" : "value2"
  }
}
```

**说明：** Map 中的键和值必须是字符串。 换句话说，你不能使用数字、布尔值、列表或其他类型的键或值。

## 语法和字符集

**注解（Annotations）** 存储的形式是键/值对。有效的注解键分为两部分： 可选的前缀和名称，以斜杠（`/`）分隔。 名称段是必需项，并且必须在 63 个字符以内，以字母数字字符（`[a-z0-9A-Z]`）开头和结尾， 并允许使用破折号（`-`），下划线（`_`），点（`.`）和字母数字。 前缀是可选的。如果指定，则前缀必须是 DNS 子域：一系列由点（`.`）分隔的 DNS 标签， 总计不超过 253 个字符，后跟斜杠（`/`）。 如果省略前缀，则假定注解键对用户是私有的。 由系统组件添加的注解 （例如，`kube-scheduler`，`kube-controller-manager`，`kube-apiserver`，`kubectl` 或其他第三方组件），必须为终端用户添加注解前缀。

`kubernetes.io/` 和 `k8s.io/` 前缀是为 Kubernetes 核心组件保留的。
