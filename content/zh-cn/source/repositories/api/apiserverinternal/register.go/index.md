---
title: "register.go"
linkTitle: "register.go"
weight: 30
date: 2025-05-14
description: >
  apiserverinternal API 类型注册
---

注册类型 APIGroupDiscoveryList 和 APIGroupDiscovery：

```go
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&StorageVersion{},
		&StorageVersionList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```