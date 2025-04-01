---
title: "ValidationOptions"
linkTitle: "ValidationOptions"
weight: 60
date: 2025-03-31
description: >
  ValidationOptions
---


## 预备方法

### 检查 Huge Pages 的 usesIndivisibleHugePagesValues() 方法

如果其中一个容器使用非整数倍的 huge page 单位大小，则 usesIndivisibleHugePagesValues() 方法 返回 true

```go
// usesIndivisibleHugePagesValues returns true if the one of the containers uses non-integer multiple
// of huge page unit size
func usesIndivisibleHugePagesValues(podSpec *api.PodSpec) bool {
	// 初始化一个布尔变量，用于记录是否找到非整数倍的 huge page 单位大小
	foundIndivisibleHugePagesValue := false
	// 遍历 podSpec 中的所有容器
	VisitContainers(podSpec, AllContainers, func(c *api.Container, containerType ContainerType) bool {
		// 检查容器是否使用非整数倍的 huge page 单位大小
		if checkContainerUseIndivisibleHugePagesValues(*c) {
			// 如果找到，则将 foundIndivisibleHugePagesValue 设置为 true
			foundIndivisibleHugePagesValue = true
		}
		// 如果还没有找到非整数倍的 huge page 单位大小，则继续遍历
		return !foundIndivisibleHugePagesValue // continue visiting if we haven't seen an invalid value yet
	})

	// 如果找到非整数倍的 huge page 单位大小，则返回 true
	if foundIndivisibleHugePagesValue {
		return true
	}

	// 游历检查 pod spec 中的 Overhead
	for resourceName, quantity := range podSpec.Overhead {
		if helper.IsHugePageResourceName(resourceName) {
			if !helper.IsHugePageResourceValueDivisible(resourceName, quantity) {
				return true
			}
		}
	}

	return false
}
```

checkContainerUseIndivisibleHugePagesValues() 方法

```go
func checkContainerUseIndivisibleHugePagesValues(container api.Container) bool {
	for resourceName, quantity := range container.Resources.Limits {
		if helper.IsHugePageResourceName(resourceName) {
			if !helper.IsHugePageResourceValueDivisible(resourceName, quantity) {
				return true
			}
		}
	}

	for resourceName, quantity := range container.Resources.Requests {
		if helper.IsHugePageResourceName(resourceName) {
			if !helper.IsHugePageResourceValueDivisible(resourceName, quantity) {
				return true
			}
		}
	}

	return false
}
```

### 检查 Topology Spread Constraints 的 hasInvalidTopologySpreadConstraintLabelSelector() 方法

hasInvalidTopologySpreadConstraintLabelSelector 如果 spec.TopologySpreadConstraints 有任何标签选择器无效的条目，则返回 true

```go
// hasInvalidTopologySpreadConstraintLabelSelector return true if spec.TopologySpreadConstraints have any entry with invalid labelSelector
func hasInvalidTopologySpreadConstraintLabelSelector(spec *api.PodSpec) bool {
	for _, constraint := range spec.TopologySpreadConstraints {
		errs := metavalidation.ValidateLabelSelector(constraint.LabelSelector, metavalidation.LabelSelectorValidationOptions{AllowInvalidLabelValueInSelector: false}, nil)
		if len(errs) != 0 {
			return true
		}
	}
	return false
}
```

TopologySpreadConstraints = 拓扑扩展限制

### 检查 Volumes 的 hasNonLocalProjectedTokenPath() 方法

hasNonLocalProjectedTokenPath 如果 spec.Volumes 有任何条目具有非本地投影标记路径（non-local projected token path），则返回 true

```go
// hasNonLocalProjectedTokenPath return true if spec.Volumes have any entry with non-local projected token path
func hasNonLocalProjectedTokenPath(spec *api.PodSpec) bool {
	for _, volume := range spec.Volumes {
		if volume.Projected != nil {
			for _, source := range volume.Projected.Sources {
				if source.ServiceAccountToken == nil {
					continue
				}
				errs := apivalidation.ValidateLocalNonReservedPath(source.ServiceAccountToken.Path, nil)
				if len(errs) != 0 {
					return true
				}
			}
		}
	}
	return false
}
```

## GetValidationOptionsFromPodSpecAndMeta() 方法

GetValidationOptionsFromPodSpecAndMeta() 方法 根据 pod spec和元数据返回验证选项:

```go
// GetValidationOptionsFromPodSpecAndMeta returns validation options based on pod specs and metadata
func GetValidationOptionsFromPodSpecAndMeta(podSpec, oldPodSpec *api.PodSpec, podMeta, oldPodMeta *metav1.ObjectMeta) apivalidation.PodValidationOptions {
	// 默认的 pod 验证选项基于 feature gate
	opts := apivalidation.PodValidationOptions{
		AllowInvalidPodDeletionCost: !utilfeature.DefaultFeatureGate.Enabled(features.PodDeletionCost),
		// Do not allow pod spec to use non-integer multiple of huge page unit size default
		AllowIndivisibleHugePagesValues:                   false,
		AllowInvalidLabelValueInSelector:                  false,
		AllowInvalidTopologySpreadConstraintLabelSelector: false,
		AllowNamespacedSysctlsForHostNetAndHostIPC:        false,
		AllowNonLocalProjectedTokenPath:                   false,
		AllowPodLifecycleSleepActionZeroValue:             utilfeature.DefaultFeatureGate.Enabled(features.PodLifecycleSleepActionAllowZero),
		PodLevelResourcesEnabled:                          utilfeature.DefaultFeatureGate.Enabled(features.PodLevelResources),
		AllowInvalidLabelValueInRequiredNodeAffinity:      false,
		AllowSidecarResizePolicy:                          utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling),
	}

	// 如果旧的 spec 使用宽松的验证或启用了 RelaxedEnvironmentVariableValidation feature gate，
	// 我们必须允许它
	opts.AllowRelaxedEnvironmentVariableValidation = useRelaxedEnvironmentVariableValidation(podSpec, oldPodSpec)
	opts.AllowRelaxedDNSSearchValidation = useRelaxedDNSSearchValidation(oldPodSpec)

	opts.AllowOnlyRecursiveSELinuxChangePolicy = useOnlyRecursiveSELinuxChangePolicy(oldPodSpec)

	if oldPodSpec != nil {
		// 如果旧的 spec 使用非整数倍的 huge page 单位大小，我们必须允许它
		opts.AllowIndivisibleHugePagesValues = usesIndivisibleHugePagesValues(oldPodSpec)

		opts.AllowInvalidLabelValueInSelector = hasInvalidLabelValueInAffinitySelector(oldPodSpec)
		opts.AllowInvalidLabelValueInRequiredNodeAffinity = hasInvalidLabelValueInRequiredNodeAffinity(oldPodSpec)
		// 如果旧的 spec 有无效的标签选择器在 topologySpreadConstraint，我们必须允许它
		opts.AllowInvalidTopologySpreadConstraintLabelSelector = hasInvalidTopologySpreadConstraintLabelSelector(oldPodSpec)
		// 如果旧的 spec 有无效的投影标记卷路径，我们必须允许它
		opts.AllowNonLocalProjectedTokenPath = hasNonLocalProjectedTokenPath(oldPodSpec)

		// 如果旧的 spec 有无效的 sysctl 与 hostNet 或 hostIPC，我们必须允许它在更新时
		if oldPodSpec.SecurityContext != nil && len(oldPodSpec.SecurityContext.Sysctls) != 0 {
			for _, s := range oldPodSpec.SecurityContext.Sysctls {
				err := apivalidation.ValidateHostSysctl(s.Name, oldPodSpec.SecurityContext, nil)
				if err != nil {
					opts.AllowNamespacedSysctlsForHostNetAndHostIPC = true
					break
				}
			}
		}

		opts.AllowPodLifecycleSleepActionZeroValue = opts.AllowPodLifecycleSleepActionZeroValue || podLifecycleSleepActionZeroValueInUse(oldPodSpec)
		// 如果旧的 pod 有在可重启的 init 容器上设置的 resize 策略，我们必须允许它
		opts.AllowSidecarResizePolicy = opts.AllowSidecarResizePolicy || hasRestartableInitContainerResizePolicy(oldPodSpec)
	}
	if oldPodMeta != nil && !opts.AllowInvalidPodDeletionCost {
		// 这是一个更新，所以只验证现有的对象是否有效
		_, err := helper.GetDeletionCostFromPodAnnotations(oldPodMeta.Annotations)
		opts.AllowInvalidPodDeletionCost = err != nil
	}

	return opts
}
```

useRelaxedEnvironmentVariableValidation() 方法, 如果旧的 spec 使用宽松的验证或启用了 RelaxedEnvironmentVariableValidation feature gate，则返回 true

```go
func useRelaxedEnvironmentVariableValidation(podSpec, oldPodSpec *api.PodSpec) bool {
	if utilfeature.DefaultFeatureGate.Enabled(features.RelaxedEnvironmentVariableValidation) {
		return true
	}

	var oldPodEnvVarNames, podEnvVarNames sets.Set[string]
	if oldPodSpec != nil {
		oldPodEnvVarNames = gatherPodEnvVarNames(oldPodSpec)
	}

	if podSpec != nil {
		podEnvVarNames = gatherPodEnvVarNames(podSpec)
	}

	for env := range podEnvVarNames {
		if relaxedEnvVarUsed(env, oldPodEnvVarNames) {
			return true
		}
	}

	return false
}
```

useRelaxedDNSSearchValidation() 方法, 如果旧的 spec 使用宽松的验证或启用了 RelaxedDNSSearchValidation feature gate，则返回 true

```go
func useRelaxedDNSSearchValidation(oldPodSpec *api.PodSpec) bool {
	// Return true early if feature gate is enabled
	if utilfeature.DefaultFeatureGate.Enabled(features.RelaxedDNSSearchValidation) {
		return true
	}

	// Return false early if there is no DNSConfig or Searches.
	if oldPodSpec == nil || oldPodSpec.DNSConfig == nil || oldPodSpec.DNSConfig.Searches == nil {
		return false
	}

	return hasDotOrUnderscore(oldPodSpec.DNSConfig.Searches)
}
```

hasDotOrUnderscore() 方法检查是否存在点或下划线：

```go
// Helper function to check if any domain is a dot or contains an underscore.
func hasDotOrUnderscore(searches []string) bool {
	for _, domain := range searches {
		if domain == "." || strings.Contains(domain, "_") {
			return true
		}
	}
	return false
}
```

gatherPodEnvVarNames() 方法收集 pod 环境变量名称：

```go
func gatherPodEnvVarNames(podSpec *api.PodSpec) sets.Set[string] {
	podEnvVarNames := sets.Set[string]{}

	for _, c := range podSpec.Containers {
		for _, env := range c.Env {
			podEnvVarNames.Insert(env.Name)
		}

		for _, env := range c.EnvFrom {
			podEnvVarNames.Insert(env.Prefix)
		}
	}

	for _, c := range podSpec.InitContainers {
		for _, env := range c.Env {
			podEnvVarNames.Insert(env.Name)
		}

		for _, env := range c.EnvFrom {
			podEnvVarNames.Insert(env.Prefix)
		}
	}

	for _, c := range podSpec.EphemeralContainers {
		for _, env := range c.Env {
			podEnvVarNames.Insert(env.Name)
		}

		for _, env := range c.EnvFrom {
			podEnvVarNames.Insert(env.Prefix)
		}
	}

	return podEnvVarNames
}
```

relaxedEnvVarUsed() 检查是否使用宽松的环境变量名称：

```go
func relaxedEnvVarUsed(name string, oldPodEnvVarNames sets.Set[string]) bool {
	// A length of 0 means this is not an update request,
	// or the old pod does not exist in the env.
	// We will let the feature gate decide whether to use relaxed rules.
	if oldPodEnvVarNames.Len() == 0 {
		return false
	}

	if len(validation.IsEnvVarName(name)) == 0 || len(validation.IsRelaxedEnvVarName(name)) != 0 {
		// It's either a valid name by strict rules or an invalid name under relaxed rules.
		// Either way, we'll use strict rules to validate.
		return false
	}

	// The name in question failed strict rules but passed relaxed rules.
	if oldPodEnvVarNames.Has(name) {
		// This relaxed-rules name was already in use.
		return true
	}

	return false
}
```

## GetValidationOptionsFromPodTemplate() 方法

GetValidationOptionsFromPodTemplate() 方法 将返回指定模板的 pod 验证选项。

```go
// GetValidationOptionsFromPodTemplate will return pod validation options for specified template.
func GetValidationOptionsFromPodTemplate(podTemplate, oldPodTemplate *api.PodTemplateSpec) apivalidation.PodValidationOptions {
	var newPodSpec, oldPodSpec *api.PodSpec
	var newPodMeta, oldPodMeta *metav1.ObjectMeta
	// 我们必须小心这里关于 nil 指针的问题
	// 特别是 replication controller 容易传递 nil
	if podTemplate != nil {
		newPodSpec = &podTemplate.Spec
		newPodMeta = &podTemplate.ObjectMeta
	}
	if oldPodTemplate != nil {
		oldPodSpec = &oldPodTemplate.Spec
		oldPodMeta = &oldPodTemplate.ObjectMeta
	}
	return GetValidationOptionsFromPodSpecAndMeta(newPodSpec, oldPodSpec, newPodMeta, oldPodMeta)
}
```