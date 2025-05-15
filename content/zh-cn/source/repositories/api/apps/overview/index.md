---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-05-14
description: >
  apps API
---


## 介绍

apps API 的核心功能是提供对 应用层工作负载 的声明式管理，主要包括：

- 部署管理：滚动更新、回滚、扩缩容
- 状态维护：确保应用实例数与声明的一致
- 版本控制：管理应用更新时的版本切换
- 与底层资源（Pod）的协调

主要管理的资源类型：

1. 无状态应用（Deployment）
2. 有状态应用（StatefulSet）
3. 守护进程（DaemonSet）
4. 一次性任务（ReplicaSet，通常由 Deployment 自动管理）

## 核心 API 资源类型

在 apps/v1 中定义了以下关键资源（所有资源均为 namespace-scoped）：

1. Deployment
2. DaemonSet
3. StatefulSet
4. ReplicaSet



## 源码学习路径

类型定义：

- staging/src/k8s.io/api/apps/v1/types.go

控制器实现：

- Deployment: pkg/controller/deployment/
- StatefulSet: pkg/controller/statefulset/
- DaemonSet: pkg/controller/daemon/

客户端操作：

- staging/src/k8s.io/client-go/kubernetes/typed/apps/v1/

## 设计意义

1. 抽象与自动化：将 Pod 管理从手动操作转为声明式管理
2. 版本化运维：支持灰度发布、金丝雀发布等高级部署模式
3. 状态分离：区分应用定义（Spec）和运行时状态（Status）

