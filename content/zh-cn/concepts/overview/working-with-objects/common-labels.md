---
title: "推荐使用的标签"
linkTitle: "推荐使用的标签"
weight: 90
date: 2021-02-01
description: >
  推荐的标签使管理应用程序变得更容易但不是任何核心工具所必需的。
---

> https://kubernetes.io/zh-cn/docs/concepts/overview/working-with-objects/common-labels/

一组通用的标签可以让多个工具之间相互操作，用所有工具都能理解的通用方式描述对象。

除了支持工具外，推荐的标签还以一种可以查询的方式描述了应用程序。

共享标签和注解都使用同一个前缀：`app.kubernetes.io`。没有前缀的标签是用户私有的。 共享前缀可以确保共享标签不会干扰用户自定义的标签。
