---
title: "Visitor"
linkTitle: "Visitor"
weight: 40
date: 2025-03-31
description: >
  pod Visitor 源码
---


## Visitor() 方法定义

Visitor() 方法会被每个对象名称调用，如果要继续访问，则返回 true

```go
// Visitor is called with each object name, and returns true if visiting should continue
type Visitor func(name string) (shouldContinue bool)
```

### skipEmptyNames() 方法

skipEmptyNames 检查 name 是否为空，如果是，则跳过 visitor() 方法的调用，直接返回 true。

```go
func skipEmptyNames(visitor Visitor) Visitor {
	return func(name string) bool {
		if len(name) == 0 {
			// continue visiting
			// 相当于如果 name 为空，则跳过 visitor() 方法直接返回 true
			return true
		}
		// delegate to visitor
		return visitor(name)
	}
}
```

## VisitPodSecretNames() 方法

VisitPodSecretNames() 方法 会调用 visit() 函数，并输入 pod spec 引用的每个 secret 的名称。

如果 visitor 返回 false，则访问将被短路。

不会访问传递引用（例如 pod -> pvc -> pv -> secret）。

如果访问完成，则返回 true；如果访问短路，则返回 false。

```go

// VisitPodSecretNames invokes the visitor function with the name of every secret
// referenced by the pod spec. If visitor returns false, visiting is short-circuited.
// Transitive references (e.g. pod -> pvc -> pv -> secret) are not visited.
// Returns true if visiting completed, false if visiting was short-circuited.
func VisitPodSecretNames(pod *api.Pod, visitor Visitor, containerType ContainerType) bool {
	// 跳过空名称
	visitor = skipEmptyNames(visitor)

	// 对于每个 ImagePullSecrets
	for _, reference := range pod.Spec.ImagePullSecrets {
		if !visitor(reference.Name) {
			return false
		}
	}

	// 调用 VisitContainers() 方法，检查 container 的 secret
	// 传入 ContainerVisitor 方法实现为调用 visitContainerSecretNames() 方法
	VisitContainers(&pod.Spec, containerType, func(c *api.Container, containerType ContainerType) bool {
		return visitContainerSecretNames(c, visitor)
	})

	// 对于每个 Volume，检查 Volume 的 secret
	var source *api.VolumeSource
	for i := range pod.Spec.Volumes {
		source = &pod.Spec.Volumes[i].VolumeSource
		switch {
		case source.AzureFile != nil:
			if len(source.AzureFile.SecretName) > 0 && !visitor(source.AzureFile.SecretName) {
				return false
			}
		case source.CephFS != nil:
			if source.CephFS.SecretRef != nil && !visitor(source.CephFS.SecretRef.Name) {
				return false
			}
		case source.Cinder != nil:
			if source.Cinder.SecretRef != nil && !visitor(source.Cinder.SecretRef.Name) {
				return false
			}
		case source.FlexVolume != nil:
			if source.FlexVolume.SecretRef != nil && !visitor(source.FlexVolume.SecretRef.Name) {
				return false
			}
		case source.Projected != nil:
			for j := range source.Projected.Sources {
				if source.Projected.Sources[j].Secret != nil {
					if !visitor(source.Projected.Sources[j].Secret.Name) {
						return false
					}
				}
			}
		case source.RBD != nil:
			if source.RBD.SecretRef != nil && !visitor(source.RBD.SecretRef.Name) {
				return false
			}
		case source.Secret != nil:
			if !visitor(source.Secret.SecretName) {
				return false
			}
		case source.ScaleIO != nil:
			if source.ScaleIO.SecretRef != nil && !visitor(source.ScaleIO.SecretRef.Name) {
				return false
			}
		case source.ISCSI != nil:
			if source.ISCSI.SecretRef != nil && !visitor(source.ISCSI.SecretRef.Name) {
				return false
			}
		case source.StorageOS != nil:
			if source.StorageOS.SecretRef != nil && !visitor(source.StorageOS.SecretRef.Name) {
				return false
			}
		case source.CSI != nil:
			if source.CSI.NodePublishSecretRef != nil && !visitor(source.CSI.NodePublishSecretRef.Name) {
				return false
			}
		}
	}
	return true
}
```

visitContainerSecretNames() 方法游历 container.EnvFrom 和 container.Env，检查 SecretRef 和 SecretKeyRef 的 name：

```go
func visitContainerSecretNames(container *api.Container, visitor Visitor) bool {
	for _, env := range container.EnvFrom {
		if env.SecretRef != nil {
			if !visitor(env.SecretRef.Name) {
				return false
			}
		}
	}
	for _, envVar := range container.Env {
		if envVar.ValueFrom != nil && envVar.ValueFrom.SecretKeyRef != nil {
			if !visitor(envVar.ValueFrom.SecretKeyRef.Name) {
				return false
			}
		}
	}
	return true
}
```

## VisitPodConfigmapNames() 方法

VisitPodConfigMapNames() 方法 会调用 visitor 函数，并输入 pod spec 引用的每个 config map 的名称。

如果 visitor 返回 false，则访问将被短路。

不会访问传递引用（例如 pod -> pvc -> pv -> secret）。

如果访问完成，则返回 true；如果访问短路，则返回 false。

```go
func VisitPodConfigmapNames(pod *api.Pod, visitor Visitor, containerType ContainerType) bool {
	visitor = skipEmptyNames(visitor)
	VisitContainers(&pod.Spec, containerType, func(c *api.Container, containerType ContainerType) bool {
		return visitContainerConfigmapNames(c, visitor)
	})
	var source *api.VolumeSource
	for i := range pod.Spec.Volumes {
		source = &pod.Spec.Volumes[i].VolumeSource
		switch {
		case source.Projected != nil:
			for j := range source.Projected.Sources {
				if source.Projected.Sources[j].ConfigMap != nil {
					if !visitor(source.Projected.Sources[j].ConfigMap.Name) {
						return false
					}
				}
			}
		case source.ConfigMap != nil:
			if !visitor(source.ConfigMap.Name) {
				return false
			}
		}
	}
	return true
}
```

visitContainerConfigmapNames() 方法游历 container.EnvFrom 和 container.Env，检查 ConfigMapRef 和 ConfigMapKeyRef 的 name：

```go
func visitContainerConfigmapNames(container *api.Container, visitor Visitor) bool {
	for _, env := range container.EnvFrom {
		if env.ConfigMapRef != nil {
			if !visitor(env.ConfigMapRef.Name) {
				return false
			}
		}
	}
	for _, envVar := range container.Env {
		if envVar.ValueFrom != nil && envVar.ValueFrom.ConfigMapKeyRef != nil {
			if !visitor(envVar.ValueFrom.ConfigMapKeyRef.Name) {
				return false
			}
		}
	}
	return true
}
```