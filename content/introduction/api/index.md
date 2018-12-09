---
date: 2018-12-08T09:00:00+08:00
title: Kubernetes API
weight: 140
description : "Kubernetes API"
---

[API conventions doc](https://git.k8s.io/community/contributors/devel/api-conventions.md) 中描述了整体API约定。

[API Reference](https://kubernetes.io/docs/reference) 中描述了API端点，资源类型和示例。

[Controlling API Access doc](https://kubernetes.io/docs/reference/access-authn-authz/controlling-access/) 文档中讨论了对API的远程访问。

Kubernetes API还可用作系统声明性配置模式的基础。kubectl 命令行工具可用于创建，更新，删除和获取API对象。

Kubernetes还根据API资源存储其序列化状态（当前在etcd中）。

Kubernetes本身被分解为多个组件，这些组件通过其API进行交互。

### API 变更

根据我们的经验，任何成功的系统都需要随着新用例的出现或现有用例的变化而增长和变化。因此，我们希望Kubernetes API能够不断变化和发展。但是，我们打算在很长一段时间内不破坏与现有客户端的兼容性。通常，可以预期新的API资源和新的资源字段的添加是频繁的。消除资源或字段将需要遵循 [API弃用策略](https://kubernetes.io/docs/reference/using-api/deprecation-policy/)。

[API更改文档](https://git.k8s.io/community/contributors/devel/api_changes.md) 详细说明了可兼容更改的组成以及如何变更API。

### OpenAPI 和 Swagger 定义

使用 Swagger v1.2 和 OpenAPI 记录完整的API详细信息。Kubernetes apiserver（又名“master”）公开了API，可用于检索位于 `/swaggerapi` 的 Swagger v1.2 Kubernetes API规范。

从 Kubernetes 1.10开始，OpenAPI规范在单个 `/openapi/v2` 端点中提供。格式分隔的端点（`/swagger.json`, `/swagger-2.0.0.json`, `/swagger-2.0.0.pb-v1`, `/swagger-2.0.0.pb-v1.gz`）已弃用，将被删除 在Kubernetes 1.14。

通过设置HTTP标头指定请求的格式：

| Header          | 可能的值                                                     |
| --------------- | ------------------------------------------------------------ |
| Accept          | `application/json`, `application/com.github.proto-openapi.spec.v2@v1.0+protobuf` ( 对于 `*/*` 或者未传递这个header，默认 content-type 是 `application/json` ) |
| Accept-Encoding | `gzip` (不传递这个header是可以被接受的)                      |

获取OpenAPI规范的示例：

| 在 1.10 之前                | 从 Kubernetes 1.10 开始                                      |
| --------------------------- | ------------------------------------------------------------ |
| GET /swagger.json           | GET /openapi/v2 **Accept**: application/json                 |
| GET /swagger-2.0.0.pb-v1    | GET /openapi/v2 **Accept**: application/com.github.proto-openapi.spec.v2@v1.0+protobuf |
| GET /swagger-2.0.0.pb-v1.gz | GET /openapi/v2 **Accept**: application/com.github.proto-openapi.spec.v2@v1.0+protobuf **Accept-Encoding**: gzip |

Kubernetes为API实现了另一种基于Protobuf的序列化格式，主要用于集群内通信，在 [设计提案](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/protobuf.md) 中有记录，每个模式的IDL文件都位于定义API对象的Go包中。

### API 版本

为了更容易消除字段或重构资源表示，Kubernetes支持多个API版本，每个API版本位于不同的API路径，例如 `/api/v1`或 `/apis/extensions/v1beta1`。

我们选择在API级别而不是在资源或字段级别进行版本控制，以确保API提供清晰，一致的系统资源和行为视图，并允许控制对生命周期尾期和/或实验API的访问。JSON和Protobuf序列化模式遵循相同的模式更改指南 - 以下所有描述都涵盖两种格式。

请注意，API版本控制和软件版本控制仅间接相关。API和发布版本控制提议描述了API版本控制和软件版本控制之间的关系。

不同的API版本意味着不同级别的稳定性和支持。[API Changes documentation](https://git.k8s.io/community/contributors/devel/api_changes.md#alpha-beta-and-stable-versions) 中更详细地描述了每个级别的标准。 他们总结在这里：

- Alpha 级别

	- 版本名称包含 `alpha`（例如`v1alpha1`）。
	- 可能是buggy。 启用该功能可能会导致错误。默认情况下禁用。
	- 可以随时删除对功能的支持，恕不另行通知。
	- API可能会在以后的软件版本中以不兼容的方式更改，恕不另行通知。
	- 由于错误风险增加和缺乏长期支持，建议仅在短期测试集群中使用。

- Beta 级别

	- 版本名称包含 `beta`（例如`v2beta3`）。
	- 代码经过了充分测试。启用该功能被认为是安全的。默认情况下启用。
	- 虽然细节可能会有所变化，但不会删除对整体功能的支持。
	- 在随后的beta版或稳定版中，对象的模式和/或语义可能以不兼容的方式发生变化。发生这种情况时，我们将提供迁移到下一版本的说明。这可能需要删除，编辑和重新创建API对象。编辑过程可能需要一些思考。对于依赖该功能的应用程序，这可能需要停机时间。
	- 建议仅用于非关键业务用途，因为后续版本中可能存在不兼容的更改。如果您有多个可以独立升级的集群，您可以放宽此限制。
	- 请尝试我们的测试版功能并提供反馈！ 一旦他们退出测试版，我们可能无法进行更多更改。

- Stable 级别

	- 版本名称是 `vX`，其中 `X` 是整数。
	- 稳定版本的功能将会在许多后续版本的发布软件中出现。

### API 分组

为了更容易扩展Kubernetes API，我们实现了API分组。 API分组在REST路径和序列化对象的apiVersion字段中指定。

目前有几个API分组正在使用中：

- core group（通常称为 legacy group）位于REST路径 `/api/v1` 并使用 `apiVersion：v1`。

- 命名组位于REST路径 `/apis/$GROUP_NAME/$VERSION`，并使用 `apiVersion: $GROUP_NAME/$VERSION`（例如 `apiVersion: batch/v1`）。 在 [Kubernetes API reference](https://kubernetes.io/docs/reference/) 中可以看到支持的API分组的完整列表。

支持两种使用 [自定义资源](https://kubernetes.io/docs/concepts/api-extension/custom-resources/) 扩展API的路径：

- [CustomResourceDefinition](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/) 适用于具有非常基本的CRUD需求的用户。
- 需要全套Kubernetes API语义的用户可以实现自己的apiserver并使用 [聚合器](https://kubernetes.io/docs/tasks/access-kubernetes-api/configure-aggregation-layer/) 无缝链接到客户端。

### 启用API 分组

某些资源和API组在默认情况下启用。可以通过在 apiserver 上设置 `--runtime-config` 来启用或禁用它们。 `--runtime-config` 接受逗号分隔值 例如：要禁用 batch/v1，请设置 `--runtime-config=batch/v1=false`，要启用 batch/v2alpha1，请设置 `--runtime-config =batch/v2alpha1`。该标志接受逗号分隔的一组key=value对，描述 apiserver 的运行时配置。

重要信息：启用或禁用分组或资源需要重新启动 apiserver 和 controller-manager 以获取 `--runtime-config` 更改。

### 启用分组中的资源

DaemonSets，Deployments，HorizontalPodAutoscalers，Ingress，Jobs和ReplicaSet 在默认情况下是启用的。可以通过在 apiserver 上设置 `--runtime-config` 来启用其他扩展资源。`--runtime-config` 接受逗号分隔值。例如：要禁用 deployment 和 ingress，请设置 `--runtime-config=extensions/v1beta1/deployments=false，extensions/v1beta1/ ingress=false`


### 参考资料

- [The Kubernetes API](https://kubernetes.io/docs/concepts/overview/kubernetes-api/): 官方文档的介绍

