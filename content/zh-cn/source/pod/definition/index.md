---
title: "pod的定义"
linkTitle: "定义"
weight: 20
date: 2025-04-01
description: >
  pod的结构体定义和属性
---

## 概述

pod的结构体定义和属性，来自 cursor 的提示，感谢 cursor 的强大。

Pod 是 Kubernetes 中最小的可部署单元，它的定义位于 `staging/src/k8s.io/api/core/v1/types.go` 文件中。一个 Pod 包含以下主要部分：

1. 基本结构：
```go
type Pod struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`
    Spec PodSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`
    Status PodStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

2. 主要组件：
- `metadata`: 包含 Pod 的元数据信息，如名称、命名空间、标签等
- `spec`: Pod 的规格定义，包含以下重要字段：
  - `containers`: Pod 中的容器列表（至少需要一个容器）
  - `initContainers`: 初始化容器列表
  - `volumes`: 可挂载的存储卷列表
  - `restartPolicy`: 重启策略（Always、OnFailure、Never）
  - `nodeSelector`: 节点选择器
  - `serviceAccountName`: 服务账号名称
  - `securityContext`: 安全上下文
  - `imagePullSecrets`: 镜像拉取密钥
  - `affinity`: 亲和性设置
  - `tolerations`: 容忍设置
  - 等等...

- `status`: Pod 的当前状态信息，包含：
  - `phase`: Pod 的生命周期阶段（Pending、Running、Succeeded、Failed、Unknown）
  - `conditions`: Pod 的条件状态
  - `message`: 状态描述信息
  - `reason`: 状态原因
  - `hostIP`: 主机 IP
  - `podIP`: Pod IP
  - `containerStatuses`: 容器状态列表
  - `initContainerStatuses`: 初始化容器状态列表
  - `qosClass`: 服务质量等级（Guaranteed、Burstable、BestEffort）

3. 生命周期：
Pod 有五个主要阶段：
- `Pending`: Pod 已被系统接受，但容器镜像尚未创建
- `Running`: Pod 已绑定到节点，所有容器已启动
- `Succeeded`: 所有容器已成功终止
- `Failed`: 所有容器已终止，至少有一个容器失败
- `Unknown`: 无法获取 Pod 状态

4. 条件状态：
Pod 可以有以下条件状态：
- `ContainersReady`: 所有容器是否就绪
- `PodInitialized`: 所有初始化容器是否成功启动
- `PodReady`: Pod 是否可以服务请求
- `PodScheduled`: Pod 的调度状态
- `DisruptionTarget`: Pod 是否即将被终止
- `PodReadyToStartContainers`: Pod 沙箱是否配置成功
- `PodResizePending`: Pod 是否正在调整大小
- `PodResizeInProgress`: Pod 大小调整是否正在进行

这就是 Kubernetes 中 Pod 的基本定义和结构。Pod 是 Kubernetes 中最小的可部署单元，它可以包含一个或多个容器，这些容器共享网络命名空间和存储卷。

## pod 结构体

pod的结构体定义在 `staging/src/k8s.io/api/core/v1/pod.go` 文件中:

```go
type Pod struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object's metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#metadata
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Specification of the desired behavior of the pod.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
	// +optional
	Spec PodSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// Most recently observed status of the pod.
	// This data may not be up to date.
	// Populated by the system.
	// Read-only.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#spec-and-status
	// +optional
	Status PodStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```

> 备注: `staging/src/k8s.io/api` 的代码会被同步到仓库 [https://github.com/kubernetes/api](https://github.com/kubernetes/api) 中

## PodSpec 结构体

PodSpec 的定义在同一个文件中:

```go
// PodSpec is a description of a pod.
type PodSpec struct {
	// List of volumes that can be mounted by containers belonging to the pod.
	// More info: https://kubernetes.io/docs/concepts/storage/volumes
	// +optional
	// +patchMergeKey=name
	// +patchStrategy=merge,retainKeys
	// +listType=map
	// +listMapKey=name
	Volumes []Volume `json:"volumes,omitempty" patchStrategy:"merge,retainKeys" patchMergeKey:"name" protobuf:"bytes,1,rep,name=volumes"`
	// List of initialization containers belonging to the pod.
	// Init containers are executed in order prior to containers being started. If any
	// init container fails, the pod is considered to have failed and is handled according
	// to its restartPolicy. The name for an init container or normal container must be
	// unique among all containers.
	// Init containers may not have Lifecycle actions, Readiness probes, Liveness probes, or Startup probes.
	// The resourceRequirements of an init container are taken into account during scheduling
	// by finding the highest request/limit for each resource type, and then using the max of
	// that value or the sum of the normal containers. Limits are applied to init containers
	// in a similar fashion.
	// Init containers cannot currently be added or removed.
	// Cannot be updated.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/init-containers/
	// +patchMergeKey=name
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=name
	InitContainers []Container `json:"initContainers,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,20,rep,name=initContainers"`
	// List of containers belonging to the pod.
	// Containers cannot currently be added or removed.
	// There must be at least one container in a Pod.
	// Cannot be updated.
	// +patchMergeKey=name
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=name
	Containers []Container `json:"containers" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,2,rep,name=containers"`
	// List of ephemeral containers run in this pod. Ephemeral containers may be run in an existing
	// pod to perform user-initiated actions such as debugging. This list cannot be specified when
	// creating a pod, and it cannot be modified by updating the pod spec. In order to add an
	// ephemeral container to an existing pod, use the pod's ephemeralcontainers subresource.
	// +optional
	// +patchMergeKey=name
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=name
	EphemeralContainers []EphemeralContainer `json:"ephemeralContainers,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,34,rep,name=ephemeralContainers"`
	// Restart policy for all containers within the pod.
	// One of Always, OnFailure, Never. In some contexts, only a subset of those values may be permitted.
	// Default to Always.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy
	// +optional
	RestartPolicy RestartPolicy `json:"restartPolicy,omitempty" protobuf:"bytes,3,opt,name=restartPolicy,casttype=RestartPolicy"`
	// Optional duration in seconds the pod needs to terminate gracefully. May be decreased in delete request.
	// Value must be non-negative integer. The value zero indicates stop immediately via
	// the kill signal (no opportunity to shut down).
	// If this value is nil, the default grace period will be used instead.
	// The grace period is the duration in seconds after the processes running in the pod are sent
	// a termination signal and the time when the processes are forcibly halted with a kill signal.
	// Set this value longer than the expected cleanup time for your process.
	// Defaults to 30 seconds.
	// +optional
	TerminationGracePeriodSeconds *int64 `json:"terminationGracePeriodSeconds,omitempty" protobuf:"varint,4,opt,name=terminationGracePeriodSeconds"`
	// Optional duration in seconds the pod may be active on the node relative to
	// StartTime before the system will actively try to mark it failed and kill associated containers.
	// Value must be a positive integer.
	// +optional
	ActiveDeadlineSeconds *int64 `json:"activeDeadlineSeconds,omitempty" protobuf:"varint,5,opt,name=activeDeadlineSeconds"`
	// Set DNS policy for the pod.
	// Defaults to "ClusterFirst".
	// Valid values are 'ClusterFirstWithHostNet', 'ClusterFirst', 'Default' or 'None'.
	// DNS parameters given in DNSConfig will be merged with the policy selected with DNSPolicy.
	// To have DNS options set along with hostNetwork, you have to specify DNS policy
	// explicitly to 'ClusterFirstWithHostNet'.
	// +optional
	DNSPolicy DNSPolicy `json:"dnsPolicy,omitempty" protobuf:"bytes,6,opt,name=dnsPolicy,casttype=DNSPolicy"`
	// NodeSelector is a selector which must be true for the pod to fit on a node.
	// Selector which must match a node's labels for the pod to be scheduled on that node.
	// More info: https://kubernetes.io/docs/concepts/configuration/assign-pod-node/
	// +optional
	// +mapType=atomic
	NodeSelector map[string]string `json:"nodeSelector,omitempty" protobuf:"bytes,7,rep,name=nodeSelector"`

	// ServiceAccountName is the name of the ServiceAccount to use to run this pod.
	// More info: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
	// +optional
	ServiceAccountName string `json:"serviceAccountName,omitempty" protobuf:"bytes,8,opt,name=serviceAccountName"`
	// DeprecatedServiceAccount is a deprecated alias for ServiceAccountName.
	// Deprecated: Use serviceAccountName instead.
	// +k8s:conversion-gen=false
	// +optional
	DeprecatedServiceAccount string `json:"serviceAccount,omitempty" protobuf:"bytes,9,opt,name=serviceAccount"`
	// AutomountServiceAccountToken indicates whether a service account token should be automatically mounted.
	// +optional
	AutomountServiceAccountToken *bool `json:"automountServiceAccountToken,omitempty" protobuf:"varint,21,opt,name=automountServiceAccountToken"`

	// NodeName indicates in which node this pod is scheduled.
	// If empty, this pod is a candidate for scheduling by the scheduler defined in schedulerName.
	// Once this field is set, the kubelet for this node becomes responsible for the lifecycle of this pod.
	// This field should not be used to express a desire for the pod to be scheduled on a specific node.
	// https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/#nodename
	// +optional
	NodeName string `json:"nodeName,omitempty" protobuf:"bytes,10,opt,name=nodeName"`
	// Host networking requested for this pod. Use the host's network namespace.
	// If this option is set, the ports that will be used must be specified.
	// Default to false.
	// +k8s:conversion-gen=false
	// +optional
	HostNetwork bool `json:"hostNetwork,omitempty" protobuf:"varint,11,opt,name=hostNetwork"`
	// Use the host's pid namespace.
	// Optional: Default to false.
	// +k8s:conversion-gen=false
	// +optional
	HostPID bool `json:"hostPID,omitempty" protobuf:"varint,12,opt,name=hostPID"`
	// Use the host's ipc namespace.
	// Optional: Default to false.
	// +k8s:conversion-gen=false
	// +optional
	HostIPC bool `json:"hostIPC,omitempty" protobuf:"varint,13,opt,name=hostIPC"`
	// Share a single process namespace between all of the containers in a pod.
	// When this is set containers will be able to view and signal processes from other containers
	// in the same pod, and the first process in each container will not be assigned PID 1.
	// HostPID and ShareProcessNamespace cannot both be set.
	// Optional: Default to false.
	// +k8s:conversion-gen=false
	// +optional
	ShareProcessNamespace *bool `json:"shareProcessNamespace,omitempty" protobuf:"varint,27,opt,name=shareProcessNamespace"`
	// SecurityContext holds pod-level security attributes and common container settings.
	// Optional: Defaults to empty.  See type description for default values of each field.
	// +optional
	SecurityContext *PodSecurityContext `json:"securityContext,omitempty" protobuf:"bytes,14,opt,name=securityContext"`
	// ImagePullSecrets is an optional list of references to secrets in the same namespace to use for pulling any of the images used by this PodSpec.
	// If specified, these secrets will be passed to individual puller implementations for them to use.
	// More info: https://kubernetes.io/docs/concepts/containers/images#specifying-imagepullsecrets-on-a-pod
	// +optional
	// +patchMergeKey=name
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=name
	ImagePullSecrets []LocalObjectReference `json:"imagePullSecrets,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,15,rep,name=imagePullSecrets"`
	// Specifies the hostname of the Pod
	// If not specified, the pod's hostname will be set to a system-defined value.
	// +optional
	Hostname string `json:"hostname,omitempty" protobuf:"bytes,16,opt,name=hostname"`
	// If specified, the fully qualified Pod hostname will be "<hostname>.<subdomain>.<pod namespace>.svc.<cluster domain>".
	// If not specified, the pod will not have a domainname at all.
	// +optional
	Subdomain string `json:"subdomain,omitempty" protobuf:"bytes,17,opt,name=subdomain"`
	// If specified, the pod's scheduling constraints
	// +optional
	Affinity *Affinity `json:"affinity,omitempty" protobuf:"bytes,18,opt,name=affinity"`
	// If specified, the pod will be dispatched by specified scheduler.
	// If not specified, the pod will be dispatched by default scheduler.
	// +optional
	SchedulerName string `json:"schedulerName,omitempty" protobuf:"bytes,19,opt,name=schedulerName"`
	// If specified, the pod's tolerations.
	// +optional
	// +listType=atomic
	Tolerations []Toleration `json:"tolerations,omitempty" protobuf:"bytes,22,opt,name=tolerations"`
	// HostAliases is an optional list of hosts and IPs that will be injected into the pod's hosts
	// file if specified.
	// +optional
	// +patchMergeKey=ip
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=ip
	HostAliases []HostAlias `json:"hostAliases,omitempty" patchStrategy:"merge" patchMergeKey:"ip" protobuf:"bytes,23,rep,name=hostAliases"`
	// If specified, indicates the pod's priority. "system-node-critical" and
	// "system-cluster-critical" are two special keywords which indicate the
	// highest priorities with the former being the highest priority. Any other
	// name must be defined by creating a PriorityClass object with that name.
	// If not specified, the pod priority will be default or zero if there is no
	// default.
	// +optional
	PriorityClassName string `json:"priorityClassName,omitempty" protobuf:"bytes,24,opt,name=priorityClassName"`
	// The priority value. Various system components use this field to find the
	// priority of the pod. When Priority Admission Controller is enabled, it
	// prevents users from setting this field. The admission controller populates
	// this field from PriorityClassName.
	// The higher the value, the higher the priority.
	// +optional
	Priority *int32 `json:"priority,omitempty" protobuf:"bytes,25,opt,name=priority"`
	// Specifies the DNS parameters of a pod.
	// Parameters specified here will be merged to the generated DNS
	// configuration based on DNSPolicy.
	// +optional
	DNSConfig *PodDNSConfig `json:"dnsConfig,omitempty" protobuf:"bytes,26,opt,name=dnsConfig"`
	// If specified, all readiness gates will be evaluated for pod readiness.
	// A pod is ready when all its containers are ready AND
	// all conditions specified in the readiness gates have status equal to "True"
	// More info: https://git.k8s.io/enhancements/keps/sig-network/580-pod-readiness-gates
	// +optional
	// +listType=atomic
	ReadinessGates []PodReadinessGate `json:"readinessGates,omitempty" protobuf:"bytes,28,opt,name=readinessGates"`
	// RuntimeClassName refers to a RuntimeClass object in the node.k8s.io group, which should be used
	// to run this pod.  If no RuntimeClass resource matches the named class, the pod will not be run.
	// If unset or empty, the "legacy" RuntimeClass will be used, which is an implicit class with an
	// empty definition that uses the default runtime handler.
	// More info: https://git.k8s.io/enhancements/keps/sig-node/585-runtime-class
	// +optional
	RuntimeClassName *string `json:"runtimeClassName,omitempty" protobuf:"bytes,29,opt,name=runtimeClassName"`
	// EnableServiceLinks indicates whether information about services should be injected into pod's
	// environment variables, matching the syntax of Docker links.
	// Optional: Defaults to true.
	// +optional
	EnableServiceLinks *bool `json:"enableServiceLinks,omitempty" protobuf:"varint,30,opt,name=enableServiceLinks"`
	// PreemptionPolicy is the Policy for preempting pods with lower priority.
	// One of Never, PreemptLowerPriority.
	// Defaults to PreemptLowerPriority if unset.
	// +optional
	PreemptionPolicy *PreemptionPolicy `json:"preemptionPolicy,omitempty" protobuf:"bytes,31,opt,name=preemptionPolicy"`
	// Overhead represents the resource overhead associated with running a pod for a given RuntimeClass.
	// This field will be autopopulated at admission time by the RuntimeClass admission controller. If
	// the RuntimeClass admission controller is enabled, overhead must not be set in Pod create requests.
	// The RuntimeClass admission controller will reject Pod create requests which have the overhead already
	// set. If RuntimeClass is configured and selected in the PodSpec, Overhead will be set to the value
	// defined in the corresponding RuntimeClass, otherwise it will remain unset and treated as zero.
	// More info: https://git.k8s.io/enhancements/keps/sig-node/688-pod-overhead/README.md
	// +optional
	Overhead ResourceList `json:"overhead,omitempty" protobuf:"bytes,32,opt,name=overhead"`
	// TopologySpreadConstraints describes how a group of pods ought to spread across topology
	// domains. Scheduler will schedule pods in a way which abides by the constraints.
	// All topologySpreadConstraints are ANDed.
	// +optional
	// +patchMergeKey=topologyKey
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=topologyKey
	// +listMapKey=whenUnsatisfiable
	TopologySpreadConstraints []TopologySpreadConstraint `json:"topologySpreadConstraints,omitempty" patchStrategy:"merge" patchMergeKey:"topologyKey" protobuf:"bytes,33,opt,name=topologySpreadConstraints"`
	// If true the pod's hostname will be configured as the pod's FQDN, rather than the leaf name (the default).
	// In Linux containers, this means setting the FQDN in the hostname field of the kernel (the nodename field of struct utsname).
	// In Windows containers, this means setting the registry value of hostname for the registry key HKEY_LOCAL_MACHINE\\SYSTEM\\CurrentControlSet\\Services\\Tcpip\\Parameters to FQDN.
	// If a pod does not have FQDN, this has no effect.
	// Default to false.
	// +optional
	SetHostnameAsFQDN *bool `json:"setHostnameAsFQDN,omitempty" protobuf:"varint,35,opt,name=setHostnameAsFQDN"`
	// Specifies the OS of the containers in the pod.
	// Some pod and container fields are restricted if this is set.
	//
	// If the OS field is set to linux, the following fields must be unset:
	// -securityContext.windowsOptions
	//
	// If the OS field is set to windows, following fields must be unset:
	// - spec.hostPID
	// - spec.hostIPC
	// - spec.hostUsers
	// - spec.securityContext.appArmorProfile
	// - spec.securityContext.seLinuxOptions
	// - spec.securityContext.seccompProfile
	// - spec.securityContext.fsGroup
	// - spec.securityContext.fsGroupChangePolicy
	// - spec.securityContext.sysctls
	// - spec.shareProcessNamespace
	// - spec.securityContext.runAsUser
	// - spec.securityContext.runAsGroup
	// - spec.securityContext.supplementalGroups
	// - spec.securityContext.supplementalGroupsPolicy
	// - spec.containers[*].securityContext.appArmorProfile
	// - spec.containers[*].securityContext.seLinuxOptions
	// - spec.containers[*].securityContext.seccompProfile
	// - spec.containers[*].securityContext.capabilities
	// - spec.containers[*].securityContext.readOnlyRootFilesystem
	// - spec.containers[*].securityContext.privileged
	// - spec.containers[*].securityContext.allowPrivilegeEscalation
	// - spec.containers[*].securityContext.procMount
	// - spec.containers[*].securityContext.runAsUser
	// - spec.containers[*].securityContext.runAsGroup
	// +optional
	OS *PodOS `json:"os,omitempty" protobuf:"bytes,36,opt,name=os"`

	// Use the host's user namespace.
	// Optional: Default to true.
	// If set to true or not present, the pod will be run in the host user namespace, useful
	// for when the pod needs a feature only available to the host user namespace, such as
	// loading a kernel module with CAP_SYS_MODULE.
	// When set to false, a new userns is created for the pod. Setting false is useful for
	// mitigating container breakout vulnerabilities even allowing users to run their
	// containers as root without actually having root privileges on the host.
	// This field is alpha-level and is only honored by servers that enable the UserNamespacesSupport feature.
	// +k8s:conversion-gen=false
	// +optional
	HostUsers *bool `json:"hostUsers,omitempty" protobuf:"bytes,37,opt,name=hostUsers"`

	// SchedulingGates is an opaque list of values that if specified will block scheduling the pod.
	// If schedulingGates is not empty, the pod will stay in the SchedulingGated state and the
	// scheduler will not attempt to schedule the pod.
	//
	// SchedulingGates can only be set at pod creation time, and be removed only afterwards.
	//
	// +patchMergeKey=name
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=name
	// +optional
	SchedulingGates []PodSchedulingGate `json:"schedulingGates,omitempty" patchStrategy:"merge" patchMergeKey:"name" protobuf:"bytes,38,opt,name=schedulingGates"`
	// ResourceClaims defines which ResourceClaims must be allocated
	// and reserved before the Pod is allowed to start. The resources
	// will be made available to those containers which consume them
	// by name.
	//
	// This is an alpha field and requires enabling the
	// DynamicResourceAllocation feature gate.
	//
	// This field is immutable.
	//
	// +patchMergeKey=name
	// +patchStrategy=merge,retainKeys
	// +listType=map
	// +listMapKey=name
	// +featureGate=DynamicResourceAllocation
	// +optional
	ResourceClaims []PodResourceClaim `json:"resourceClaims,omitempty" patchStrategy:"merge,retainKeys" patchMergeKey:"name" protobuf:"bytes,39,rep,name=resourceClaims"`
	// Resources is the total amount of CPU and Memory resources required by all
	// containers in the pod. It supports specifying Requests and Limits for
	// "cpu" and "memory" resource names only. ResourceClaims are not supported.
	//
	// This field enables fine-grained control over resource allocation for the
	// entire pod, allowing resource sharing among containers in a pod.
	// TODO: For beta graduation, expand this comment with a detailed explanation.
	//
	// This is an alpha field and requires enabling the PodLevelResources feature
	// gate.
	//
	// +featureGate=PodLevelResources
	// +optional
	Resources *ResourceRequirements `json:"resources,omitempty" protobuf:"bytes,40,opt,name=resources"`
}
```

## PodStatus 结构体

PodStatus 的定义在同一个文件中:

```go

// PodStatus represents information about the status of a pod. Status may trail the actual
// state of a system, especially if the node that hosts the pod cannot contact the control
// plane.
type PodStatus struct {
	// If set, this represents the .metadata.generation that the pod status was set based upon.
	// This is an alpha field. Enable PodObservedGenerationTracking to be able to use this field.
	// +featureGate=PodObservedGenerationTracking
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty" protobuf:"varint,17,opt,name=observedGeneration"`
	// The phase of a Pod is a simple, high-level summary of where the Pod is in its lifecycle.
	// The conditions array, the reason and message fields, and the individual container status
	// arrays contain more detail about the pod's status.
	// There are five possible phase values:
	//
	// Pending: The pod has been accepted by the Kubernetes system, but one or more of the
	// container images has not been created. This includes time before being scheduled as
	// well as time spent downloading images over the network, which could take a while.
	// Running: The pod has been bound to a node, and all of the containers have been created.
	// At least one container is still running, or is in the process of starting or restarting.
	// Succeeded: All containers in the pod have terminated in success, and will not be restarted.
	// Failed: All containers in the pod have terminated, and at least one container has
	// terminated in failure. The container either exited with non-zero status or was terminated
	// by the system.
	// Unknown: For some reason the state of the pod could not be obtained, typically due to an
	// error in communicating with the host of the pod.
	//
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-phase
	// +optional
	Phase PodPhase `json:"phase,omitempty" protobuf:"bytes,1,opt,name=phase,casttype=PodPhase"`
	// Current service state of pod.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-conditions
	// +optional
	// +patchMergeKey=type
	// +patchStrategy=merge
	// +listType=map
	// +listMapKey=type
	Conditions []PodCondition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type" protobuf:"bytes,2,rep,name=conditions"`
	// A human readable message indicating details about why the pod is in this condition.
	// +optional
	Message string `json:"message,omitempty" protobuf:"bytes,3,opt,name=message"`
	// A brief CamelCase message indicating details about why the pod is in this state.
	// e.g. 'Evicted'
	// +optional
	Reason string `json:"reason,omitempty" protobuf:"bytes,4,opt,name=reason"`
	// nominatedNodeName is set only when this pod preempts other pods on the node, but it cannot be
	// scheduled right away as preemption victims receive their graceful termination periods.
	// This field does not guarantee that the pod will be scheduled on this node. Scheduler may decide
	// to place the pod elsewhere if other nodes become available sooner. Scheduler may also decide to
	// give the resources on this node to a higher priority pod that is created after preemption.
	// As a result, this field may be different than PodSpec.nodeName when the pod is
	// scheduled.
	// +optional
	NominatedNodeName string `json:"nominatedNodeName,omitempty" protobuf:"bytes,11,opt,name=nominatedNodeName"`

	// hostIP holds the IP address of the host to which the pod is assigned. Empty if the pod has not started yet.
	// A pod can be assigned to a node that has a problem in kubelet which in turns mean that HostIP will
	// not be updated even if there is a node is assigned to pod
	// +optional
	HostIP string `json:"hostIP,omitempty" protobuf:"bytes,5,opt,name=hostIP"`

	// hostIPs holds the IP addresses allocated to the host. If this field is specified, the first entry must
	// match the hostIP field. This list is empty if the pod has not started yet.
	// A pod can be assigned to a node that has a problem in kubelet which in turns means that HostIPs will
	// not be updated even if there is a node is assigned to this pod.
	// +optional
	// +patchStrategy=merge
	// +patchMergeKey=ip
	// +listType=atomic
	HostIPs []HostIP `json:"hostIPs,omitempty" protobuf:"bytes,16,rep,name=hostIPs" patchStrategy:"merge" patchMergeKey:"ip"`

	// podIP address allocated to the pod. Routable at least within the cluster.
	// Empty if not yet allocated.
	// +optional
	PodIP string `json:"podIP,omitempty" protobuf:"bytes,6,opt,name=podIP"`

	// podIPs holds the IP addresses allocated to the pod. If this field is specified, the 0th entry must
	// match the podIP field. Pods may be allocated at most 1 value for each of IPv4 and IPv6. This list
	// is empty if no IPs have been allocated yet.
	// +optional
	// +patchStrategy=merge
	// +patchMergeKey=ip
	// +listType=map
	// +listMapKey=ip
	PodIPs []PodIP `json:"podIPs,omitempty" protobuf:"bytes,12,rep,name=podIPs" patchStrategy:"merge" patchMergeKey:"ip"`

	// RFC 3339 date and time at which the object was acknowledged by the Kubelet.
	// This is before the Kubelet pulled the container image(s) for the pod.
	// +optional
	StartTime *metav1.Time `json:"startTime,omitempty" protobuf:"bytes,7,opt,name=startTime"`

	// Statuses of init containers in this pod. The most recent successful non-restartable
	// init container will have ready = true, the most recently started container will have
	// startTime set.
	// Each init container in the pod should have at most one status in this list,
	// and all statuses should be for containers in the pod.
	// However this is not enforced.
	// If a status for a non-existent container is present in the list, or the list has duplicate names,
	// the behavior of various Kubernetes components is not defined and those statuses might be
	// ignored.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-and-container-status
	// +listType=atomic
	InitContainerStatuses []ContainerStatus `json:"initContainerStatuses,omitempty" protobuf:"bytes,10,rep,name=initContainerStatuses"`

	// Statuses of containers in this pod.
	// Each container in the pod should have at most one status in this list,
	// and all statuses should be for containers in the pod.
	// However this is not enforced.
	// If a status for a non-existent container is present in the list, or the list has duplicate names,
	// the behavior of various Kubernetes components is not defined and those statuses might be
	// ignored.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-and-container-status
	// +optional
	// +listType=atomic
	ContainerStatuses []ContainerStatus `json:"containerStatuses,omitempty" protobuf:"bytes,8,rep,name=containerStatuses"`

	// The Quality of Service (QOS) classification assigned to the pod based on resource requirements
	// See PodQOSClass type for available QOS classes
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/#quality-of-service-classes
	// +optional
	QOSClass PodQOSClass `json:"qosClass,omitempty" protobuf:"bytes,9,rep,name=qosClass"`

	// Statuses for any ephemeral containers that have run in this pod.
	// Each ephemeral container in the pod should have at most one status in this list,
	// and all statuses should be for containers in the pod.
	// However this is not enforced.
	// If a status for a non-existent container is present in the list, or the list has duplicate names,
	// the behavior of various Kubernetes components is not defined and those statuses might be
	// ignored.
	// More info: https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle#pod-and-container-status
	// +optional
	// +listType=atomic
	EphemeralContainerStatuses []ContainerStatus `json:"ephemeralContainerStatuses,omitempty" protobuf:"bytes,13,rep,name=ephemeralContainerStatuses"`

	// Status of resources resize desired for pod's containers.
	// It is empty if no resources resize is pending.
	// Any changes to container resources will automatically set this to "Proposed"
	// Deprecated: Resize status is moved to two pod conditions PodResizePending and PodResizeInProgress.
	// PodResizePending will track states where the spec has been resized, but the Kubelet has not yet allocated the resources.
	// PodResizeInProgress will track in-progress resizes, and should be present whenever allocated resources != acknowledged resources.
	// +featureGate=InPlacePodVerticalScaling
	// +optional
	Resize PodResizeStatus `json:"resize,omitempty" protobuf:"bytes,14,opt,name=resize,casttype=PodResizeStatus"`

	// Status of resource claims.
	// +patchMergeKey=name
	// +patchStrategy=merge,retainKeys
	// +listType=map
	// +listMapKey=name
	// +featureGate=DynamicResourceAllocation
	// +optional
	ResourceClaimStatuses []PodResourceClaimStatus `json:"resourceClaimStatuses,omitempty" patchStrategy:"merge,retainKeys" patchMergeKey:"name" protobuf:"bytes,15,rep,name=resourceClaimStatuses"`
}
```

## reference 文档

pod 的 reference 文档:

https://kubernetes.io/zh-cn/docs/reference/kubernetes-api/workload-resources/pod-v1/