---
title: "types.go"
linkTitle: "types.go"
weight: 20
date: 2025-05-14
description: >
  authorization API 类型
---





### TokenReview

```go

```

### TokenRequest

```go
// TokenRequest requests a token for a given service account.
type TokenRequest struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Spec holds information about the request being evaluated
	Spec TokenRequestSpec `json:"spec" protobuf:"bytes,2,opt,name=spec"`

	// Status is filled in by the server and indicates whether the token can be authenticated.
	// +optional
	Status TokenRequestStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

### SelfSubjectReview

```go
// SelfSubjectReview contains the user information that the kube-apiserver has about the user making this request.
// When using impersonation, users will receive the user info of the user being impersonated.  If impersonation or
// request header authentication is used, any extra keys will have their case ignored and returned as lowercase.
type SelfSubjectReview struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	// Status is filled in by the server with the user attributes.
	Status SelfSubjectReviewStatus `json:"status,omitempty" protobuf:"bytes,2,opt,name=status"`
}
```
