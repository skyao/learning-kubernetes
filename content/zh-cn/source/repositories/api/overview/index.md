---
title: "概述"
linkTitle: "概述"
weight: 1
date: 2025-05-14
description: >
  api 仓库概述
---

## api 信息

### api 列表

- admission
- admissionregistration
- apidiscovery
- apiserverinternal
- apps
- authentication
- authorization
- autoscaling
- batch
- certificates
- coordination
- core
- discovery
- doc.go
- events
- extensions
- flowcontrol
- imagepolicy
- networking
- node
- policy
- rbac
- resource
- scheduling
- storage
- storagemigration

### API 版本

大部分 api 都有多个版本，如：

- v1
- v1alpha1
- v1beta1

代码学习中，我们以 v1 版本为主。


## api 代码结构

基本上每个 api 文件夹都有如下的文件列表：

- doc.go
- generated.pb.go
- register.go
- types.go
- types_swagger_doc_generated.go
- zz_generated.deepcopy.go
- zz_generated.prerelease-lifecycle.go


### doc.go

这是包的文档文件，包含包的用途说明和基本的文档注释

通常会包含 +k8s:openapi-gen=true 这样的代码生成标记

定义了包的帮助文档和代码生成器的元数据。

打开看源码，发现除了 license 之外，只定义了 package 名字和该 package 相关的注解，如：

```bash
// +k8s:deepcopy-gen=package
// +k8s:protobuf-gen=package
// +k8s:openapi-gen=false
// +k8s:prerelease-lifecycle-gen=true
// +groupName=admission.k8s.io

package v1
```

### generated.pb.go

generated.pb.go 是 Protocol Buffers 的生成文件（由 generated.proto 生成），包含了 AdmissionReview 等类型的 Protocol Buffers 序列化相关代码，用于 Kubernetes API 的 gRPC 通信和存储序列化。

代码都是从 generated.proto 文件生成的，没啥好看的，直接看 generated.proto 文件好了。

### generated.proto

generated.proto 是Protocol Buffers 的定义文件，定义了当前 API 类型的 protobuf 消息格式。

generated.proto 文件会被 protoc 编译器用来生成 generated.pb.go

### register.go

register.go 包含 API 类型的注册逻辑，将 API 中的类型注册到 Kubernetes API 的 Scheme 中。

register.go 定义了如何将 Go 类型与 API 版本/组进行映射，包含 AddToScheme 函数，其他组件可以调用它来注册这些类型。

### types.go

最重要的文件，包含核心 API 类型的 Go 定义，定义了 API 相关的各种结构体，包含字段定义、标签注释和验证逻辑，有详细的字段注释说明每个字段的用途

### types_swagger_doc_generated.go

自动生成的 Swagger 文档，包含 API 类型的详细文档，用于生成 Kubernetes API 参考文档。

这些注释会被 Kubernetes API 文档工具使用。

### zz_generated.deepcopy.go

自动生成的 DeepCopy 方法实现，包含 api 类型的深拷贝方法，由 deepcopy-gen 工具根据 types.go 中的标记生成。

Kubernetes 运行时需要这些方法来安全地复制对象。

### zz_generated.prerelease-lifecycle.go

自动生成的 API 生命周期相关代码，包含 API 版本的弃用和移除策略，由 prerelease-lifecycle-gen 工具生成。

用于管理 API 版本的生命周期（alpha/beta/stable 状态转换）。






