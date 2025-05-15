---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-05-14
description: >
  authentication API
---


## 介绍

authentication API 的核心功能是处理 Kubernetes 集群的 身份认证，确保请求来源的合法性。

主要职责包括：

1. **验证请求者身份**：
   - 确认用户、ServiceAccount 或外部系统的身份凭证
2. **支持多种认证机制**：
   - X.509 客户端证书
   - Bearer Token（如 ServiceAccount Token）
   - 身份代理（OIDC、LDAP 等）
3. **提供认证配置管理**：
   - 管理 TLS 证书配置
   - 管理 ServiceAccount 的 Token 签发

### **核心 API 资源类型**

在 `authentication.k8s.io/v1` 和核心 API 组中定义了以下关键资源：

(1) `TokenReview`

```go
// 用于验证 Bearer Token 的有效性
type TokenReview struct {
    metav1.TypeMeta `json:",inline"`
    Spec   TokenReviewSpec   `json:"spec"`
    Status TokenReviewStatus `json:"status,omitempty"`
}
```

**关键字段**：

- `spec.token`：待验证的 Token 字符串
- `status.authenticated`：验证结果（true/false）
- `status.user`：认证通过后的用户信息（用户名、组等）

(2) `CertificateSigningRequest` (CSR)

```go
// 在 certificates.k8s.io/v1 中定义，但用于认证流程
type CertificateSigningRequest struct {
    metav1.TypeMeta `json:",inline"`
    Spec   CertificateSigningRequestSpec `json:"spec"`
}
```

**关键字段**：

- `spec.request`：PEM 编码的证书签名请求
- `spec.signerName`：指定签名用途（如 `kubernetes.io/kube-apiserver-client`）

(3) `ServiceAccount`

```go
// 在核心 v1 API 中定义，用于 Pod 身份认证
type ServiceAccount struct {
    metav1.TypeMeta `json:",inline"`
    Secrets []ObjectReference `json:"secrets,omitempty"` // 关联的 Token Secret
}
```
