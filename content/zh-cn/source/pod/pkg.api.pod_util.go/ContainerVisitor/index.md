---
title: "ContainerVisitor"
linkTitle: "ContainerVisitor"
weight: 30
date: 2025-03-31
description: >
  pod ContainerVisitor 源码
---

### ContainerType 定义

ContainerVisitor 类型定义：

```go
// ContainerVisitor is called with each container spec, and returns true
// if visiting should continue.
type ContainerVisitor func(container *api.Container, containerType ContainerType) (shouldContinue bool)
```

每个容器规格都会调用 ContainerVisitor，如果应该继续访问，则返回 true 。

### VisitContainers() 方法

VisitContainers() 方法使用指向给定 pod spec 中每个容器规格的指针调用访问者函数，容器规格的类型在掩码中设置。

如果 visitor 返回 false，则表示访问短路。如果访问完成，VisitContainers 返回 true；如果访问短路，则返回 false。

```go
// VisitContainers invokes the visitor function with a pointer to every container
// spec in the given pod spec with type set in mask. If visitor returns false,
// visiting is short-circuited. VisitContainers returns true if visiting completes,
// false if visiting was short-circuited.
func VisitContainers(podSpec *api.PodSpec, mask ContainerType, visitor ContainerVisitor) bool {
	// 如果 ContainerType 是 InitContainers 
	if mask&InitContainers != 0 {
		// 对于每个 InitContainer
		for i := range podSpec.InitContainers {
			//调用 visitor() 方法
			if !visitor(&podSpec.InitContainers[i], InitContainers) {
				// 只要任何一个返回 false，则中断并返回 false
				return false
			}
		}
	}

	// 如果 ContainerType 是普通 Containers
	if mask&Containers != 0 {
		// 对于每个 普通 Container
		for i := range podSpec.Containers {
			if !visitor(&podSpec.Containers[i], Containers) {
				return false
			}
		}
	}

	// 如果 ContainerType 是普通 EphemeralContainers
	if mask&EphemeralContainers != 0 {
		// 对于每个 EphemeralContainers
		for i := range podSpec.EphemeralContainers {
			if !visitor((*api.Container)(&podSpec.EphemeralContainers[i].EphemeralContainerCommon), EphemeralContainers) {
				return false
			}
		}
	}
	return true
}
```

