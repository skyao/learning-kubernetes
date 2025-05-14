---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-05-14
description: >
  Admission API  
---


### 介绍

Admission API 主要用于 动态准入控制（Dynamic Admission Control），即在 Kubernetes API 请求被处理之前或之后，对其进行 拦截、验证或修改。它的主要功能包括：

- 验证（Validation）：检查资源的合法性（如 Pod 的镜像是否符合安全策略）。
- 变更（Mutation）：修改请求的内容（如自动注入 Sidecar 容器）。
- 审计（Auditing）：记录 API 请求的详细信息，用于安全审计。

Admission API 通常与 Admission Webhooks 配合使用，允许用户自定义准入逻辑（如通过 Webhook 调用外部服务）。

## 核心类型

Admission API 主要定义在 `types.go` 中，核心结构体包括：

1. AdmissionReview

    - 用于封装 **准入请求（Request）** 和 **准入响应（Response）**。

    - 是 Webhook 和 API Server 之间的通信格式。

2. AdmissionRequest

    表示一个 **准入请求**，包含所有需要验证或修改的资源信息。

3. AdmissionResponse

    表示 **准入控制的结果**，决定是否允许该请求，并可以返回修改内容。

## HTTP 方法

Admission API 本身不直接提供 REST API，而是 **由 API Server 调用 Webhook** 时使用。其交互流程如下：

1. **API Server 收到请求**（如 `kubectl create -f pod.yaml`）。
2. **API Server 构造 `AdmissionReview`**，发送给配置的 Webhook。
3. **Webhook 处理请求**，返回 `AdmissionReview{ Response: ... }`。
4. **API Server 根据响应决定是否继续处理**。

## 使用场景

Admission API 通常用于：

1. **强制安全策略**（如禁止特权容器）。
2. **资源默认值注入**（如自动设置 `requests/limits`）。
3. **Sidecar 注入**（如 Istio 自动注入 `istio-proxy`）。
4. **自定义验证逻辑**（如检查 Ingress 域名是否合法）。
