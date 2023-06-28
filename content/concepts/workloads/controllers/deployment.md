---
title: "Deployments"
linkTitle: "Deployments"
weight: 200
date: 2021-02-01
description: >
  Deployment 为 Pod 和 ReplicaSet 提供声明式的更新能力
---

> https://kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/

你负责描述 Deployment 中的 目标状态，而 Deployment 控制器（Controller） 以受控速率更改实际状态， 使其变为期望状态。你可以定义 Deployment 以创建新的 ReplicaSet，或删除现有 Deployment， 并通过新的 Deployment 收养其资源。
