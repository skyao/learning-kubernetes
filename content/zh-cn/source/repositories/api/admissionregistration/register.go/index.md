---
title: "register.go"
linkTitle: "register.go"
weight: 30
date: 2025-05-14
description: >
  AdmissionRegistration API 类型注册
---

注册类型

```go
// Adds the list of known types to the given scheme.
func addKnownTypes(scheme *runtime.Scheme) error {
	scheme.AddKnownTypes(SchemeGroupVersion,
		&ValidatingWebhookConfiguration{},
		&ValidatingWebhookConfigurationList{},
		&MutatingWebhookConfiguration{},
		&MutatingWebhookConfigurationList{},
		&ValidatingAdmissionPolicy{},
		&ValidatingAdmissionPolicyList{},
		&ValidatingAdmissionPolicyBinding{},
		&ValidatingAdmissionPolicyBindingList{},
	)
	metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
	return nil
}
```