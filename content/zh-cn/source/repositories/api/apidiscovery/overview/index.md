---
title: "概述"
linkTitle: "概述"
weight: 10
date: 2025-05-14
description: >
  APIDiscovery API
---


### 介绍

APIDiscovery API 的核心功能是提供 Kubernetes API 的 动态发现机制，使客户端能够自动发现集群支持的 API 资源及其详细信息。

主要解决以下问题：

1. API 资源发现：客户端无需硬编码即可知道集群支持哪些 API（如 pods、deployments 等）
2. 版本兼容性：发现不同 API 版本（如 v1、v1beta1）及其差异
3. API 聚合支持：支持自定义 API Server（Aggregated API Server）的自动注册和发现
4. 高效缓存：客户端可以缓存发现信息，减少不必要的 API 调用

## 核心 API 资源类型

在 `apidiscovery.k8s.io/v2`（最新版本）中定义了以下关键结构：

(1) APIGroupDiscovery

```go
// 描述一个 API 组的所有可用版本和资源
type APIGroupDiscovery struct {
    metav1.TypeMeta `json:",inline"`
    metav1.GroupVersion `json:",inline"`  // 例如 "apps/v1"
    Versions []APIVersionDiscovery `json:"versions"`  // 该组下的所有版本
}
```

(2) APIVersionDiscovery

```go
// 描述一个 API 版本下的所有资源
type APIVersionDiscovery struct {
    Version string `json:"version"`  // 例如 "v1"
    Resources []APIResourceDiscovery `json:"resources"`  // 该版本下的资源列表
    Freshness metav1.ConditionStatus `json:"freshness"`  // 表示数据是否最新
}
```

(3) APIResourceDiscovery

```go
// 描述单个 API 资源的详细信息
type APIResourceDiscovery struct {
    Resource string `json:"resource"`  // 例如 "pods"
    ResponseKind *metav1.GroupVersionKind `json:"responseKind"`  // 返回的 Kind
    Scope ScopeType `json:"scope"`  // 作用域（Namespaced/Cluster）
    SingularResource string `json:"singularResource"`  // 单数形式（如 "pod"）
    Verbs []string `json:"verbs"`  // 支持的动词（get/list/watch 等）
    ShortNames []string `json:"shortNames"`  // 缩写（如 "po"）
    Categories []string `json:"categories"`  // 所属分类（如 "all"）
    Subresources []APISubresourceDiscovery `json:"subresources"`  // 子资源
}
```

## 核心 API 方法

APIDiscovery API 通过以下标准 HTTP 端点提供服务：

| HTTP 方法 | 路径                                             | 功能                             |
| :-------- | :----------------------------------------------- | :------------------------------- |
| `GET`     | `/apis/apidiscovery.k8s.io/v2`                   | 获取 APIDiscovery API 自身的描述 |
| `GET`     | `/apis/apidiscovery.k8s.io/v2/apigroups`         | 获取所有 API 组的发现信息        |
| `GET`     | `/apis/apidiscovery.k8s.io/v2/apigroups/{group}` | 获取特定 API 组的发现信息        |



## 工作原理

1. 客户端首次查询

   ```bash
   kubectl get --raw /apis/apidiscovery.k8s.io/v2beta1/apigroups
   ```

2. 解析响应

   - 获取所有 API 组、版本、资源及其能力

3. 建立本地缓存

   - 客户端缓存发现结果，后续只检查 `Freshness`

4. 处理 API 变更

   - 当 API 扩展（如 CRD 注册）时，APIServer 自动更新发现信息

------

## 实际应用场景

(1) kubectl 自动补全

```bash
# kubectl 依赖发现 API 实现自动补全
kubectl get <TAB>  # 显示所有可用资源类型
```

(2) 客户端库初始化

```go
// client-go 使用发现 API 初始化动态客户端
discoveryClient := clientset.Discovery()
resources, err := discoveryClient.ServerResources()
```

(3) 自定义控制器开发

```go
// 动态发现新的 CRD
discoveryClient.Discovery().ServerPreferredResources()
```

## 源码学习路径

1. **类型定义**：
   `staging/src/k8s.io/api/apidiscovery/v2/types.go`
2. **服务端实现**：
   `pkg/controlplane/apiserver/discovery/`
3. **客户端缓存逻辑**：
   `staging/src/k8s.io/client-go/discovery/`
