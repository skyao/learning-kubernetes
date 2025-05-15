---
title: "types.go"
linkTitle: "types.go"
weight: 20
date: 2025-05-14
description: >
  apiserverinternal API 类型
---





### StorageVersion

```go
// Storage version of a specific resource.
type StorageVersion struct {
	metav1.TypeMeta `json:",inline"`
	// The name is <group>.<resource>.
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Spec is an empty spec. It is here to comply with Kubernetes API style.
	Spec StorageVersionSpec `json:"spec" protobuf:"bytes,2,opt,name=spec"`

	// API server instances report the version they can decode and the version they
	// encode objects to when persisting objects in the backend.
	Status StorageVersionStatus `json:"status" protobuf:"bytes,3,opt,name=status"`
}
```

### StorageVersionList

```go
// A list of StorageVersions.
type StorageVersionList struct {
	metav1.TypeMeta `json:",inline"`
	// Standard list metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ListMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	// Items holds a list of StorageVersion
	Items []StorageVersion `json:"items" protobuf:"bytes,2,rep,name=items"`
}
```

