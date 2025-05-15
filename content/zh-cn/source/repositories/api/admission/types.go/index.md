---
title: "types.go"
linkTitle: "types.go"
weight: 20
date: 2025-05-14
description: >
  Admission API 类型
---


### AdmissionReview

AdmissionReview 描述了准入审查请求/应答。

```go
// AdmissionReview describes an admission review request/response.
type AdmissionReview struct {
	metav1.TypeMeta `json:",inline"`
	// Request describes the attributes for the admission request.
	// +optional
	Request *AdmissionRequest `json:"request,omitempty" protobuf:"bytes,1,opt,name=request"`
	// Response describes the attributes for the admission response.
	// +optional
	Response *AdmissionResponse `json:"response,omitempty" protobuf:"bytes,2,opt,name=response"`
}
```


### AdmissionRequest

dmissionRequest 描述了准入请求的准入属性

```go
type AdmissionRequest struct {
	// UID 是单个请求/响应的唯一标识符。它允许我们区分其他方面相同的请求实例
	//（并行请求、先前请求未修改时的请求等）
	// UID 用于跟踪 KAS 和 WebHook 之间的往返（请求/响应），而不是用户请求
	// 它适合在 webhook 和 apiserver 之间关联日志条目，用于审计或调试
	UID types.UID `json:"uid" protobuf:"bytes,1,opt,name=uid"`
	// Kind 是被提交对象的完全限定类型（例如 v1.Pod 或 autoscaling.v1.Scale）
	Kind metav1.GroupVersionKind `json:"kind" protobuf:"bytes,2,opt,name=kind"`
	// Resource 是被请求的完全限定资源（例如 v1.pods）
	Resource metav1.GroupVersionResource `json:"resource" protobuf:"bytes,3,opt,name=resource"`
	// SubResource 是被请求的子资源（如果有）（例如 "status" 或 "scale"）
	// +optional
	SubResource string `json:"subResource,omitempty" protobuf:"bytes,4,opt,name=subResource"`

	// RequestKind 是原始 API 请求的完全限定类型（例如 v1.Pod 或 autoscaling.v1.Scale）
	// 如果指定且与 "kind" 中的值不同，则表示执行了等效匹配和转换
	//
	// 例如，如果 deployments 可以通过 apps/v1 和 apps/v1beta1 修改，并且 webhook 注册了规则
	// `apiGroups:["apps"], apiVersions:["v1"], resources: ["deployments"]` 和 `matchPolicy: Equivalent`，
	// 那么发送到 apps/v1beta1 deployments 的 API 请求会被转换并发送到 webhook，
	// 其中 `kind: {group:"apps", version:"v1", kind:"Deployment"}`（匹配 webhook 注册的规则），
	// 而 `requestKind: {group:"apps", version:"v1beta1", kind:"Deployment"}`（表示原始 API 请求的类型）
	//
	// 更多详情请参阅 webhook 配置类型中 "matchPolicy" 字段的文档
	// +optional
	RequestKind *metav1.GroupVersionKind `json:"requestKind,omitempty" protobuf:"bytes,13,opt,name=requestKind"`
	// RequestResource 是原始 API 请求的完全限定资源（例如 v1.pods）
	// 如果指定且与 "resource" 中的值不同，则表示执行了等效匹配和转换
	//
	// 例如，如果 deployments 可以通过 apps/v1 和 apps/v1beta1 修改，并且 webhook 注册了规则
	// `apiGroups:["apps"], apiVersions:["v1"], resources: ["deployments"]` 和 `matchPolicy: Equivalent`，
	// 那么发送到 apps/v1beta1 deployments 的 API 请求会被转换并发送到 webhook，
	// 其中 `resource: {group:"apps", version:"v1", resource:"deployments"}`（匹配 webhook 注册的资源），
	// 而 `requestResource: {group:"apps", version:"v1beta1", resource:"deployments"}`（表示原始 API 请求的资源）
	//
	// 更多详情请参阅 webhook 配置类型中 "matchPolicy" 字段的文档
	// +optional
	RequestResource *metav1.GroupVersionResource `json:"requestResource,omitempty" protobuf:"bytes,14,opt,name=requestResource"`
	// RequestSubResource 是原始 API 请求的子资源名称（如果有）（例如 "status" 或 "scale"）
	// 如果指定且与 "subResource" 中的值不同，则表示执行了等效匹配和转换
	// 更多详情请参阅 webhook 配置类型中 "matchPolicy" 字段的文档
	// +optional
	RequestSubResource string `json:"requestSubResource,omitempty" protobuf:"bytes,15,opt,name=requestSubResource"`

	// Name 是请求中呈现的对象名称。在 CREATE 操作中，客户端可以省略名称，
	// 依赖服务器生成名称。如果是这种情况，此字段将包含空字符串
	// +optional
	Name string `json:"name,omitempty" protobuf:"bytes,5,opt,name=name"`
	// Namespace 是与请求关联的命名空间（如果有）
	// +optional
	Namespace string `json:"namespace,omitempty" protobuf:"bytes,6,opt,name=namespace"`
	// Operation 是正在执行的操作。这可能与请求的操作不同
	// 例如，一个 patch 可能导致 CREATE 或 UPDATE 操作
	Operation Operation `json:"operation" protobuf:"bytes,7,opt,name=operation"`
	// UserInfo 是关于请求用户的信息
	UserInfo authenticationv1.UserInfo `json:"userInfo" protobuf:"bytes,8,opt,name=userInfo"`
	// Object 是来自传入请求的对象
	// +optional
	Object runtime.RawExtension `json:"object,omitempty" protobuf:"bytes,9,opt,name=object"`
	// OldObject 是现有对象。仅对 DELETE 和 UPDATE 请求填充
	// +optional
	OldObject runtime.RawExtension `json:"oldObject,omitempty" protobuf:"bytes,10,opt,name=oldObject"`
	// DryRun 表示此请求的修改肯定不会持久化
	// 默认为 false
	// +optional
	DryRun *bool `json:"dryRun,omitempty" protobuf:"varint,11,opt,name=dryRun"`
	// Options 是正在执行的操作的操作选项结构
	// 例如 `meta.k8s.io/v1.DeleteOptions` 或 `meta.k8s.io/v1.CreateOptions`。这可能与
	// 调用者提供的选项不同。例如，对于 patch 请求，执行的操作可能是 CREATE，
	// 在这种情况下，Options 将是 `meta.k8s.io/v1.CreateOptions`，即使调用者提供了 `meta.k8s.io/v1.PatchOptions`
	// +optional
	Options runtime.RawExtension `json:"options,omitempty" protobuf:"bytes,12,opt,name=options"`
}
```

### AdmissionResponse

```go
// AdmissionResponse 描述了一个准入响应
type AdmissionResponse struct {
	// UID 是单个请求/响应的标识符
	// 必须从对应的 AdmissionRequest 中复制过来
	UID types.UID `json:"uid" protobuf:"bytes,1,opt,name=uid"`

	// Allowed 表示是否允许该准入请求
	Allowed bool `json:"allowed" protobuf:"varint,2,opt,name=allowed"`

	// Result 包含关于为什么拒绝准入请求的额外细节
	// 如果 "Allowed" 为 "true"，则不会参考此字段
	// +optional
	Result *metav1.Status `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`

	// patch 内容体。目前我们只支持实现了 RFC 6902 的 "JSONPatch"
	// +optional
	Patch []byte `json:"patch,omitempty" protobuf:"bytes,4,opt,name=patch"`

	// patch 类型。目前我们只允许 "JSONPatch"
	// +optional
	PatchType *PatchType `json:"patchType,omitempty" protobuf:"bytes,5,opt,name=patchType"`

	// AuditAnnotations 是由远程准入控制器设置的非结构化键值映射（例如 error=image-blacklisted）
	// MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook 准入控制器会在键前加上
	// 准入 webhook 名称（例如 imagepolicy.example.com/error=image-blacklisted）
	// 准入 webhook 提供 AuditAnnotations 来为此请求的审计日志添加上下文
	// +optional
	AuditAnnotations map[string]string `json:"auditAnnotations,omitempty" protobuf:"bytes,6,opt,name=auditAnnotations"`

	// warnings 是要返回给请求 API 客户端的警告消息列表
	// 警告消息描述了 API 请求客户端应该纠正或注意的问题
	// 尽可能将警告限制在 120 个字符以内
	// 超过 256 个字符的警告和大量警告可能会被截断
	// +optional
	Warnings []string `json:"warnings,omitempty" protobuf:"bytes,7,rep,name=warnings"`
}

// PatchType is the type of patch being used to represent the mutated object
type PatchType string

// PatchType constants.
const (
	PatchTypeJSONPatch PatchType = "JSONPatch"
)
```