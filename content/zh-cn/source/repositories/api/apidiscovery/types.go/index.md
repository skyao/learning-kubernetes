---
title: "types.go"
linkTitle: "types.go"
weight: 20
date: 2025-05-14
description: >
  apidiscovery API 类型
---





### APIGroupDiscoveryList

```go

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +k8s:prerelease-lifecycle-gen:introduced=1.30

// APIGroupDiscoveryList is a resource containing a list of APIGroupDiscovery.
// This is one of the types able to be returned from the /api and /apis endpoint and contains an aggregated
// list of API resources (built-ins, Custom Resource Definitions, resources from aggregated servers)
// that a cluster supports.
type APIGroupDiscoveryList struct {
	v1.TypeMeta `json:",inline"`
	// ResourceVersion will not be set, because this does not have a replayable ordering among multiple apiservers.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	v1.ListMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	// items is the list of groups for discovery. The groups are listed in priority order.
	Items []APIGroupDiscovery `json:"items" protobuf:"bytes,2,rep,name=items"`
}
```

### APIGroupDiscovery

```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +k8s:prerelease-lifecycle-gen:introduced=1.30

// APIGroupDiscovery holds information about which resources are being served for all version of the API Group.
// It contains a list of APIVersionDiscovery that holds a list of APIResourceDiscovery types served for a version.
// Versions are in descending order of preference, with the first version being the preferred entry.
type APIGroupDiscovery struct {
	v1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// The only field completed will be name. For instance, resourceVersion will be empty.
	// name is the name of the API group whose discovery information is presented here.
	// name is allowed to be "" to represent the legacy, ungroupified resources.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	v1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
	// versions are the versions supported in this group. They are sorted in descending order of preference,
	// with the preferred version being the first entry.
	// +listType=map
	// +listMapKey=version
	Versions []APIVersionDiscovery `json:"versions,omitempty" protobuf:"bytes,2,rep,name=versions"`
}
```

