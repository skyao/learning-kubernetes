---
title: "ContainerType"
linkTitle: "ContainerType"
weight: 20
date: 2025-03-31
description: >
  pod ContainerType
---

### ContainerType 定义

ContainerType 类型定义，有3个类型：

```go
// ContainerType signifies container type
type ContainerType int

const (
	// Containers is for normal containers
	Containers ContainerType = 1 << iota
	// InitContainers is for init containers
	InitContainers
	// EphemeralContainers is for ephemeral containers
	EphemeralContainers
)

// AllContainers specifies that all containers be visited
const AllContainers ContainerType = (InitContainers | Containers | EphemeralContainers)
```

### AllFeatureEnabledContainers() 方法

AllFeatureEnabledContainers() 方法 返回一个 ContainerType mask，其中包括所有容器类型，但受 feature gate 保护的容器类型除外。

```go
// AllContainers specifies that all containers be visited
const AllContainers ContainerType = (InitContainers | Containers | EphemeralContainers)

// AllFeatureEnabledContainers returns a ContainerType mask which includes all container
// types except for the ones guarded by feature gate.
func AllFeatureEnabledContainers() ContainerType {
	return AllContainers
}
```

> 备注： 没有什么 feature gate 啊

