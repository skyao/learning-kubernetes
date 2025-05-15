---
title: "register.go"
linkTitle: "register.go"
weight: 30
date: 2025-05-14
description: >
  Admission API 类型注册
---

就注册了一个类型 AdmissionReview：

```go
// Adds the list of known types to the given scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&AdmissionReview{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```