---
title: "Drop"
linkTitle: "Drop"
weight: 70
date: 2025-03-31
description: >
  Drop 方法
---


## DropDisabledTemplateFields() 方法

DropDisabledTemplateFields() 方法 删除 pod 模板元数据和规范中禁用的字段。

对于包含 PodTemplateSpec 的所有资源，应在 PrepareForCreate/PrepareForUpdate 中调用此功能。

```go
// DropDisabledTemplateFields removes disabled fields from the pod template metadata and spec.
// This should be called from PrepareForCreate/PrepareForUpdate for all resources containing a PodTemplateSpec
func DropDisabledTemplateFields(podTemplate, oldPodTemplate *api.PodTemplateSpec) {
	var (
		podSpec           *api.PodSpec
		podAnnotations    map[string]string
		oldPodSpec        *api.PodSpec
		oldPodAnnotations map[string]string
	)
	if podTemplate != nil {
		podSpec = &podTemplate.Spec
		podAnnotations = podTemplate.Annotations
	}
	if oldPodTemplate != nil {
		oldPodSpec = &oldPodTemplate.Spec
		oldPodAnnotations = oldPodTemplate.Annotations
	}
	dropDisabledFields(podSpec, podAnnotations, oldPodSpec, oldPodAnnotations)
}
```

### dropDisabledFields() 方法

dropDisabledFields() 方法 会删除 pod 元数据和规范中禁用的字段。

```go

// dropDisabledFields removes disabled fields from the pod metadata and spec.
func dropDisabledFields(
	podSpec *api.PodSpec, podAnnotations map[string]string,
	oldPodSpec *api.PodSpec, oldPodAnnotations map[string]string,
) {
	// the new spec must always be non-nil
	if podSpec == nil {
		podSpec = &api.PodSpec{}
	}

	// 如果 feature 被禁用且未在 use，则删除 hostUsers 字段。
	if !utilfeature.DefaultFeatureGate.Enabled(features.UserNamespacesSupport) && !hostUsersInUse(oldPodSpec) {
		// 仅在 SecurityContext 不为 nil 时删除 podSpec 中的字段。
		// 如果为 nil，则不需要设置 hostUsers=nil（它也将为 nil）。
		if podSpec.SecurityContext != nil {
			podSpec.SecurityContext.HostUsers = nil
		}
	}

	// 如果 feature 被禁用且未在 use，则删除 SupplementalGroupsPolicy 字段。
	if !utilfeature.DefaultFeatureGate.Enabled(features.SupplementalGroupsPolicy) && !supplementalGroupsPolicyInUse(oldPodSpec) {
		// 仅在 SecurityContext 不为 nil 时删除 podSpec 中的字段。
		// 如果为 nil，则不需要设置 supplementalGroupsPolicy=nil（它也将为 nil）。
		if podSpec.SecurityContext != nil {
			podSpec.SecurityContext.SupplementalGroupsPolicy = nil
		}
	}

	dropDisabledPodLevelResources(podSpec, oldPodSpec)
	dropDisabledProcMountField(podSpec, oldPodSpec)

	dropDisabledNodeInclusionPolicyFields(podSpec, oldPodSpec)
	dropDisabledMatchLabelKeysFieldInTopologySpread(podSpec, oldPodSpec)
	dropDisabledMatchLabelKeysFieldInPodAffinity(podSpec, oldPodSpec)
	dropDisabledDynamicResourceAllocationFields(podSpec, oldPodSpec)
	dropDisabledClusterTrustBundleProjection(podSpec, oldPodSpec)

	if !utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) && !inPlacePodVerticalScalingInUse(oldPodSpec) {
		// Drop ResizePolicy fields. Don't drop updates to Resources field as template.spec.resources
		// field is mutable for certain controllers. Let ValidatePodUpdate handle it.
		for i := range podSpec.Containers {
			podSpec.Containers[i].ResizePolicy = nil
		}
		for i := range podSpec.InitContainers {
			podSpec.InitContainers[i].ResizePolicy = nil
		}
		for i := range podSpec.EphemeralContainers {
			podSpec.EphemeralContainers[i].ResizePolicy = nil
		}
	}

	if !utilfeature.DefaultFeatureGate.Enabled(features.SidecarContainers) && !restartableInitContainersInUse(oldPodSpec) {
		// Drop the RestartPolicy field of init containers.
		for i := range podSpec.InitContainers {
			podSpec.InitContainers[i].RestartPolicy = nil
		}
		// For other types of containers, validateContainers will handle them.
	}

	if !utilfeature.DefaultFeatureGate.Enabled(features.RecursiveReadOnlyMounts) && !rroInUse(oldPodSpec) {
		for i := range podSpec.Containers {
			for j := range podSpec.Containers[i].VolumeMounts {
				podSpec.Containers[i].VolumeMounts[j].RecursiveReadOnly = nil
			}
		}
		for i := range podSpec.InitContainers {
			for j := range podSpec.InitContainers[i].VolumeMounts {
				podSpec.InitContainers[i].VolumeMounts[j].RecursiveReadOnly = nil
			}
		}
		for i := range podSpec.EphemeralContainers {
			for j := range podSpec.EphemeralContainers[i].VolumeMounts {
				podSpec.EphemeralContainers[i].VolumeMounts[j].RecursiveReadOnly = nil
			}
		}
	}

	dropPodLifecycleSleepAction(podSpec, oldPodSpec)
	dropImageVolumes(podSpec, oldPodSpec)
	dropSELinuxChangePolicy(podSpec, oldPodSpec)
	dropContainerStopSignals(podSpec, oldPodSpec)
}
```

## DropDisabledPodFields() 方法

DropDisabledPodFields 删除 Pod 元数据和规范中禁用的字段。

对于包含 Pod 的所有资源，应在 PrepareForCreate/PrepareForUpdate 中调用此功能。

```go
// DropDisabledPodFields removes disabled fields from the pod metadata and spec.
// This should be called from PrepareForCreate/PrepareForUpdate for all resources containing a Pod
func DropDisabledPodFields(pod, oldPod *api.Pod) {
	var (
		podSpec           *api.PodSpec
		podStatus         *api.PodStatus
		podAnnotations    map[string]string
		oldPodSpec        *api.PodSpec
		oldPodStatus      *api.PodStatus
		oldPodAnnotations map[string]string
	)
	if pod != nil {
		podSpec = &pod.Spec
		podStatus = &pod.Status
		podAnnotations = pod.Annotations
	}
	if oldPod != nil {
		oldPodSpec = &oldPod.Spec
		oldPodStatus = &oldPod.Status
		oldPodAnnotations = oldPod.Annotations
	}
	dropDisabledFields(podSpec, podAnnotations, oldPodSpec, oldPodAnnotations)
	dropDisabledPodStatusFields(podStatus, oldPodStatus, podSpec, oldPodSpec)
}
```

### dropDisabledPodStatusFields() 方法

dropDisabledPodStatusFields() 方法 删除 pod 状态中已禁用的字段

```go

// dropDisabledPodStatusFields removes disabled fields from the pod status
func dropDisabledPodStatusFields(podStatus, oldPodStatus *api.PodStatus, podSpec, oldPodSpec *api.PodSpec) {
	// 新的状态总是非 nil
	if podStatus == nil {
		podStatus = &api.PodStatus{}
	}

	if !utilfeature.DefaultFeatureGate.Enabled(features.InPlacePodVerticalScaling) && !inPlacePodVerticalScalingInUse(oldPodSpec) {
		// 删除 Resources 字段
		dropResourcesField := func(csl []api.ContainerStatus) {
			for i := range csl {
				csl[i].Resources = nil
			}
		}
		dropResourcesField(podStatus.ContainerStatuses)
		dropResourcesField(podStatus.InitContainerStatuses)
		dropResourcesField(podStatus.EphemeralContainerStatuses)

		// 删除 AllocatedResources 字段
		dropAllocatedResourcesField := func(csl []api.ContainerStatus) {
			for i := range csl {
				csl[i].AllocatedResources = nil
			}
		}
		dropAllocatedResourcesField(podStatus.ContainerStatuses)
		dropAllocatedResourcesField(podStatus.InitContainerStatuses)
		dropAllocatedResourcesField(podStatus.EphemeralContainerStatuses)
	}

	if !utilfeature.DefaultFeatureGate.Enabled(features.DynamicResourceAllocation) && !dynamicResourceAllocationInUse(oldPodSpec) {
		podStatus.ResourceClaimStatuses = nil
	}

	if !utilfeature.DefaultFeatureGate.Enabled(features.RecursiveReadOnlyMounts) && !rroInUse(oldPodSpec) {
		for i := range podStatus.ContainerStatuses {
			podStatus.ContainerStatuses[i].VolumeMounts = nil
		}
		for i := range podStatus.InitContainerStatuses {
			podStatus.InitContainerStatuses[i].VolumeMounts = nil
		}
		for i := range podStatus.EphemeralContainerStatuses {
			podStatus.EphemeralContainerStatuses[i].VolumeMounts = nil
		}
	}

	if !utilfeature.DefaultFeatureGate.Enabled(features.ResourceHealthStatus) {
		setAllocatedResourcesStatusToNil := func(csl []api.ContainerStatus) {
			for i := range csl {
				csl[i].AllocatedResourcesStatus = nil
			}
		}
		setAllocatedResourcesStatusToNil(podStatus.ContainerStatuses)
		setAllocatedResourcesStatusToNil(podStatus.InitContainerStatuses)
		setAllocatedResourcesStatusToNil(podStatus.EphemeralContainerStatuses)
	}

	// drop ContainerStatus.User field to empty (disable SupplementalGroupsPolicy)
	if !utilfeature.DefaultFeatureGate.Enabled(features.SupplementalGroupsPolicy) && !supplementalGroupsPolicyInUse(oldPodSpec) {
		dropUserField := func(csl []api.ContainerStatus) {
			for i := range csl {
				csl[i].User = nil
			}
		}
		dropUserField(podStatus.InitContainerStatuses)
		dropUserField(podStatus.ContainerStatuses)
		dropUserField(podStatus.EphemeralContainerStatuses)
	}

	if !utilfeature.DefaultFeatureGate.Enabled(features.PodObservedGenerationTracking) && !podObservedGenerationTrackingInUse(oldPodStatus) {
		podStatus.ObservedGeneration = 0
		for i := range podStatus.Conditions {
			podStatus.Conditions[i].ObservedGeneration = 0
		}
	}
}
```