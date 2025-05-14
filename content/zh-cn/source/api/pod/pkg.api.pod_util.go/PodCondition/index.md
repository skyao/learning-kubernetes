---
title: "PodCondition"
linkTitle: "PodCondition"
weight: 50
date: 2025-03-31
description: >
  pod PodCondition 源码
---


## IsPodReady() 方法

如果 pod ready 则返回 true

```go
// IsPodReady returns true if a pod is ready; false otherwise.
func IsPodReady(pod *api.Pod) bool {
	return IsPodReadyConditionTrue(pod.Status)
}
```

### IsPodReadyConditionTrue() 方法

IsPodReadyConditionTrue 检查 pod 的 PodStatus, 如果为 ConditionTrue 则返回 true：

```go
// IsPodReadyConditionTrue returns true if a pod is ready; false otherwise.
func IsPodReadyConditionTrue(status api.PodStatus) bool {
	condition := GetPodReadyCondition(status)
	return condition != nil && condition.Status == api.ConditionTrue
}
```

### GetPodReadyCondition() 方法

GetPodReadyCondition() 方法 从给定状态中提取 pod 就绪条件并返回。如果条件不存在，则返回 nil。

```go
// GetPodReadyCondition extracts the pod ready condition from the given status and returns that.
// Returns nil if the condition is not present.
func GetPodReadyCondition(status api.PodStatus) *api.PodCondition {
	_, condition := GetPodCondition(&status, api.PodReady)
	return condition
}
```

### GetPodCondition() 方法

GetPodCondition() 方法从给定的状态中提取所提供的 condition 并返回。

如果 condition 不存在，则返回 nil 和-1，

如果 condition 存在，则返回所定位 condition 的索引。

```go
// GetPodCondition extracts the provided condition from the given status and returns that.
// Returns nil and -1 if the condition is not present, and the index of the located condition.
func GetPodCondition(status *api.PodStatus, conditionType api.PodConditionType) (int, *api.PodCondition) {
	if status == nil {
		// 如果 pod 的 status 为空
		return -1, nil
	}
	// 游历 status.Conditions
	for i := range status.Conditions {
		// 如果等于指定的 conditionType
		if status.Conditions[i].Type == conditionType {
			// 返回 index 和 对应 conditionType 的 condition
			// 这里只会找到的第一个符合 conditionType 要求的 condition
			return i, &status.Conditions[i]
		}
	}
	return -1, nil
}
```

## UpdatePodCondition() 方法

UpdatePodCondition 更新现有 pod condition 或创建新的 pod condition。

如果状态已更改，则将 LastTransitionTime 设置为 now。

如果 pod condition 已更改或已添加，则返回 true。

```go
// UpdatePodCondition updates existing pod condition or creates a new one. Sets LastTransitionTime to now if the
// status has changed.
// Returns true if pod condition has changed or has been added.
func UpdatePodCondition(status *api.PodStatus, condition *api.PodCondition) bool {
	// 设置 LastTransitionTime 为当前时间
	condition.LastTransitionTime = metav1.Now()
	// Try to find this pod condition.
	conditionIndex, oldCondition := GetPodCondition(status, condition.Type)

	//如果没有找到，说明之前没有设置这个类型的 condition
	if oldCondition == nil {
		// We are adding new pod condition.
		status.Conditions = append(status.Conditions, *condition)
		return true
	}
	// We are updating an existing condition, so we need to check if it has changed.
	if condition.Status == oldCondition.Status {
		// 如果 condition 的 status 和 oldCondition 的 status 相同，则将 LastTransitionTime 设置为 oldCondition 的 LastTransitionTime
		condition.LastTransitionTime = oldCondition.LastTransitionTime
	}

	// 检查传入的 condition 和现有的 conditon 是否完全等同
	isEqual := condition.Status == oldCondition.Status &&
		condition.Reason == oldCondition.Reason &&
		condition.Message == oldCondition.Message &&
		condition.LastProbeTime.Equal(&oldCondition.LastProbeTime) &&	
		condition.LastTransitionTime.Equal(&oldCondition.LastTransitionTime)

	// 更新 condition
	status.Conditions[conditionIndex] = *condition
	// Return true if one of the fields have changed.
	return !isEqual
}
```