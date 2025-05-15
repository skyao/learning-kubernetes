---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-05-14
description: >
  AdmissionRegistration API  
---


### 介绍

AdmissionRegistration API 核心功能：用于注册和管理 Kubernetes 的 动态准入控制 Webhook，允许用户在不修改 API Server 代码的情况下扩展准入控制逻辑。

主要分为两类：

1. MutatingAdmissionWebhook
   - 用于 **修改** API 请求对象（如自动注入 Sidecar 容器）
2. ValidatingAdmissionWebhook
   - 用于 **验证** API 请求对象（如检查资源是否符合安全策略）

## 核心类型

在 `admissionregistration.k8s.io/v1` 中定义了以下关键资源：

1. MutatingWebhookConfiguration
   - 用于注册多个 **修改型 Webhook**。
   - 每个 Webhook 指定：
     - 触发规则（匹配哪些 API 请求）
     - 调用的 Webhook 服务地址
     - 失败策略（如拒绝请求或忽略错误）
2. ValidatingWebhookConfiguration
   - 用于注册多个 **验证型 Webhook**。
   - 结构与 `MutatingWebhookConfiguration` 类似，但 Webhook 不能修改对象。

## 核心 API 方法

AdmissionRegistration API 通过标准的 Kubernetes REST 接口提供以下操作：

| HTTP 方法 | 路径                                                         | 功能                                    |
| :-------- | :----------------------------------------------------------- | :-------------------------------------- |
| `GET`     | `/apis/admissionregistration.k8s.io/v1/mutatingwebhookconfigurations` | 列出所有 MutatingWebhookConfiguration   |
| `POST`    | 同上                                                         | 创建新的 MutatingWebhookConfiguration   |
| `PUT`     | `/apis/admissionregistration.k8s.io/v1/mutatingwebhookconfigurations/{name}` | 更新指定配置                            |
| `DELETE`  | 同上                                                         | 删除配置                                |
| `GET`     | `/apis/admissionregistration.k8s.io/v1/validatingwebhookconfigurations` | 列出所有 ValidatingWebhookConfiguration |
| `POST`    | 同上                                                         | 创建新的 ValidatingWebhookConfiguration |

------

### 关键字段解析

#### (1) `WebhookClientConfig`

```go
type WebhookClientConfig struct {
    URL      *string           `json:"url,omitempty"`      // Webhook 服务 URL（直接调用）
    Service  *ServiceReference `json:"service,omitempty"`  // 通过 Service 调用
    CABundle []byte            `json:"caBundle,omitempty"`  // CA 证书（用于 TLS 验证）
}
```

- 支持两种调用方式：
  - 直接指定 URL（如 `https://webhook.example.com:443/admit`）
  - 通过 Kubernetes Service 调用（推荐）

(2) `RuleWithOperations`

```go
type RuleWithOperations struct {
    Operations []OperationType `json:"operations"`  // 操作类型：CREATE, UPDATE, DELETE, *
    Rule       Rule            `json:"rule"`        // 资源匹配规则
}
```

- 示例：匹配所有 Pod 的 CREATE/UPDATE 操作：

  

  ```go
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  ```

(3) `FailurePolicy`

```go
type FailurePolicyType string
const (
    Ignore FailurePolicyType = "Ignore"  // 失败时忽略（继续请求）
    Fail   FailurePolicyType = "Fail"    // 失败时拒绝请求
)
```



## 工作原理

1. **API Server 接收请求**（如创建 Pod）
2. **检查匹配的 Webhook 规则**
   - 通过 `rules` 字段过滤
3. **调用 Webhook 服务**
   - 发送 `AdmissionReview` 请求
4. **处理响应**
   - 根据 `allowed` 和 `patch` 字段决定是否允许请求
