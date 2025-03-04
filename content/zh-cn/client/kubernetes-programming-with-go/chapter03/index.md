---
title: "[第三章]在Go中使用API资源"
linkTitle: "在Go中使用API资源"
weight: 30
date: 2021-02-01
description: >
  在Go中使用API资源

---

本书的前两章已经描述了Kubernetes API是如何设计的，以及如何使用HTTP请求访问它。具体来说，你已经看到由API管理的资源被组织成Group-Version-Resources，而客户端和API服务器之间交换的对象被Kubernetes API定义为Kinds。本章还显示，在交换过程中，这些数据可以用JSON、YAML或Protobuf编码，这取决于客户端设置的HTTP头。

在接下来的章节中，你将看到如何使用Go语言访问这个API。与Kubernetes API合作需要的两个重要Go库是apimachinery和api。

apimachinery 是一个通用库，它负责Go结构体和以 JSON（或YAML或Protobuf）格式编写的对象之间的数据序列化。这使得客户和API服务器的开发者有可能使用Go结构体编写数据，并在HTTP交换过程中透明地使用这些资源的JSON（或YAML或Protobuf）。

apimachinery 是通用的，因为它不包括任何Kubernetes API资源定义。它使Kubernetes API可以扩展，并使 apimachinery 可用于任何其他使用相同机制的API，即Kinds 和 Group-Version-Resources。

API 库则是 go 结构体的集合，需要在 go 中与Kubernetes API定义的资源一起工作。

## API库的源码和导入

API库的源代码可以从 https://github.com/kubernetes/api 。如果你想为这个库做贡献，请注意，源代码不是从这个库管理的，而是从中心库，https://github.com/kubernetes/kubernetes，在 `staging/src/k8s.io/api` 目录下，源代码从 kubernetes 库同步到 api 库中。

要把 API 库的包导入Go源代码，你需要使用 `k8s.io/api` 的前缀--例如：

```go
import "k8s.io/api/core/v1"
```

进入API库的包遵循API的 `Group-Version-Resource` 结构。当你想使用某个资源的结构体时，你需要通过这种模式导入与该资源的组和版本相关的包：

```go
import "k8s.io/api/<group>/<version>"
```

## 包的内容

让我们来检查包中包含的文件--例如，`k8s.io/api/apps/v1` 包。

### types.go

这个文件可以被认为是包的主文件，因为它定义了所有的 Kind 结构体和其他相关的子结构体。它还定义了这些结构体中的枚举字段的所有类型和常量。举个例子，考虑一下 Deployment Kind；Deployment 结构体首先被定义如下：

```go
type Deployment struct {
    metav1.TypeMeta
    metav1.ObjectMeta
    Spec      DeploymentSpec
    Status    DeploymentStatus
}
```

然后，相关的子结构体，DeploymentSpec 和 DeploymentStatus，都是用这个定义的：

```go
type DeploymentSpec struct {
     Replicas      *int32
     Selector      *metav1.LabelSelector
     Template      v1.PodTemplateSpec
     Strategy      DeploymentStrategy
     [...]
}

type DeploymentStatus struct {
     ObservedGeneration      int64
     Replicas                int32
     [...]
     Conditions              []DeploymentCondition
}
```

然后，以同样的方式继续处理在前一个结构体中作为类型使用的每一个结构体。

在 DeploymentCondition 结构体中使用的 DeploymentConditionType 类型（这里没列出），以及这个枚举的可能值都被定义：

```go
type DeploymentConditionType string
const (
     DeploymentAvailable DeploymentConditionType = "Available"
     DeploymentProgressing DeploymentConditionType = "Progressing"
     DeploymentReplicaFailure DeploymentConditionType = "ReplicaFailure"
)
```

你可以看到，每个 Kind 都嵌入了两个结构体：metav1.TypeMeta 和 metav1.ObjectMeta。它们是强制性的，以便被 API Machinery 识别。TypeMeta 结构体包含关于 Kind 的 GVK 的信息，而 ObjectMeta 包含 Kind 的元数据，比如它的 name 。

### register.go

这个文件定义了与这个包相关的分组（group）和版本（version），以及这个分组和版本中的 Kinds 列表。当你需要从这个分组-版本中指定一个资源的分组和版本时，可以使用公共变量 SchemeGroupVersion。

它还声明了一个函数 AddToScheme，它可以用来将分组、版本和 Kinds 添加到 Scheme 中。Scheme 是 API Machinery 中的一个抽象概念，用于在Go结构体和Group-Version-Kinds 之间建立映射。这将在第五章 "API Machinery" 中进一步讨论。

### doc.go

这个文件和下面的文件包含高级信息，你不需要理解这些信息就可以开始用 Go 编写你的第一个 Kubernetes 资源，但它们会帮助你理解如何在接下来的章节中用自定义资源定义声明新资源。

doc.go 文件包含以下生成文件的说明：

```go
// +k8s:deepcopy-gen=package
// +k8s:protobuf-gen=package
// +k8s:openapi-gen=true
```

第一条指令被 deepcopy-gen 生成器用来生成 zz_generated.deepcopy.go 文件。第二条指令被 go-to-protobuf 生成器用来生成这些文件：generated.pb.go 和 generated.proto。第三条指令被 genswaggertypedocs 生成器用来生成 type_swagger_doc_generated.go 文件。

### generated.pb.go 和 generated.proto

这些文件是由 go-to-protobuf 生成器生成的。当把数据序列化为 Protobuf 格式时，它们被 API Machinery 使用。

### types_swagger_doc_generated.go

这个文件是由 genswaggertypedocs 生成器生成的。它在生成 Kubernetes API 的完整 swagger 定义时使用。

### zz_generated.deepcopy.go

这个文件是由 deepcopy-gen 生成器生成的。它包含了为包中定义的每个类型生成的 DeepCopyObject 方法的定义。这个方法对于结构体 实现 **runtime.Object** 接口是必要的，这个接口是在 API Machinery 中定义的，API Machinery  期望所有的 Kind 结构体将实现这个 **runtime.Object**  接口。

该文件中的接口定义如下：

```go
type Object interface {
     GetObjectKind() schema.ObjectKind
     DeepCopyObject() Object
}
```

另一个必要的方法，GetObjectKind，被自动添加到嵌入 TypeMeta 结构体的结构体中 -- 所有 Kind 结构体都是如此。TypeMeta 结构体的方法定义如下：

```go
func (obj *TypeMeta) GetObjectKind() schema.ObjectKind {
  return obj
}
```

## core/v1中的特定内容

core/v1包除了定义核心资源的结构体外，还定义了特定类型的实用方法，当你把这些类型纳入你的代码中时，这些方法会很有用。

### ObjectReference

ObjectReference 可以用来以一种独特的方式引用任何对象。该结构体定义如下：

```go
type ObjectReference struct {
     APIVersion          string
     Kind                string
     Namespace           string
     Name                string
     UID                 types.UID
     ResourceVersion     string
     FieldPath           string
}
```

为这种类型定义了三种方法：

- `SetGroupVersionKind(gvk schema.GroupVersionKind) ` - 这个方法将根据作为参数传递的 GroupVersionKind 值来设置字段 APIVersion 和 Kind。

- `GroupVersionKind() schema.GroupVersionKind` - 该方法将根据 ObjectReference 的字段 APIVersion 和 Kind 返回一个 GroupVersionKind 值。

- `GetObjectKind() schema.ObjectKind` - 这个方法将把 ObjectReference 对象转换成 ObjectKind。前面的两个方法实现了这个 ObjectKind 接口。因为 ObjectReference 上的 DeepCopyObject 方法也被定义了，所以 ObjectReference 将实现 runtime.Object 接口。

### ResourceList

ResourceList 类型被定义为一个map，其键是 ResourceName，值是 Quantity。这被用于各种 Kubernetes 资源，以定义资源的 limits 和 requests。

在YAML中，一个使用的例子是当你为一个 container 定义资源的 requests 和 limits 时，如下所示：

```go
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: runtime
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```

在Go中，你可以把 requests 部分写成：

```go
requests := corev1.ResourceList{
     corev1.ResourceMemory:
        *resource.NewQuantity(64*1024*1024, resource.BinarySI),
     corev1.ResourceCPU:
        *resource.NewMilliQuantity(250, resource.DecimalSI),
}
```

下一章将更详细地描述如何使用 resource.Quantity 类型来定义数量。ResourceList 类型有以下方法：

- `Cpu() *resource.Quantity` - 返回 map 中 CPU 键的数量，以十进制格式（1250m等）。

- `Memory() *resource.Quantity` - 返回 map 中 Memory 键的数量，以二进制格式表示（512 Ki, 64 Mi, etc.）

- `storage() *resource.Quantity` - 返回 map 中 **Storage** 键的数量，以二进制格式表示（512 Mi, 1 Gi等）。

- `Pods() *resource.Quantity` - 返回 map 的 Pods 键的数量，以十进制格式（1，10等）。

- `StorageEphemeral() *resource.Quantity` - 返回 map 中 StorageEphemeral 键的数量，以二进制格式表示（512 Mi, 1 Gi, etc.)

对于这些方法中的每一个，如果键没有在 map 中定义，将返回一个零值的 Quantity。

另一个方法，在内部被前面的方法使用，可以被用来获得非标准格式的数量：

- `Name(name ResourceName, defaultFormat resource.Format) *resource.Quantity` - 这返回 Name 键的数量，以 defaultFormat 格式。

ResourceName 类型的定义枚举值是 ResourceCPU、ResourceMemory、ResourceStorage、ResourceEphemeralStorage 和 ResourcePods 。

### Taint / 污点

**Taint** 资源是为了应用于节点，以确保不容忍这些污点的 pod 不会被安排到这些节点。Taint 结构体定义如下：

```go
type Taint struct {
    Key            string
    Value          string
    Effect         TaintEffect
    TimeAdded      *metav1.Time
}
```

TaintEffect 枚举可以得到以下值：

```go
TaintEffectNoSchedule          = "NoSchedule"
TaintEffectPreferNoSchedule    = "PreferNoSchedule"
TaintEffectNoExecute           = "NoExecute"
```

众所周知的 Taint 键，在特殊情况下被控制平面使用，在这个包中定义如下：

```go
TaintNodeNotReady
   = "node.kubernetes.io/not-ready"
TaintNodeUnreachable
   = "node.kubernetes.io/unreachable"
TaintNodeUnschedulable
   = "node.kubernetes.io/unschedulable"
TaintNodeMemoryPressure
   = "node.kubernetes.io/memory-pressure"
TaintNodeDiskPressure
   = "node.kubernetes.io/disk-pressure"
TaintNodeNetworkUnavailable
   ="node.kubernetes.io/network-unavailable"
TaintNodePIDPressure
   = "node.kubernetes.io/pid-pressure"
TaintNodeOutOfService
   = "node.kubernetes.io/out-of-service"
```

以下两个方法是在 Taint 上定义的：

- `MatchTaint(taintToMatch *Taint) bool `- 如果两个污点的 key 和  Effect 值相同，这个方法将返回 true。

- `ToString() string` - 这个方法将返回 Taint 的字符串表示，格式是这样的： `<key>=<value>:<effect>, <key>=<value>:, <key>:<effect>, 或 <key>`.

### Toleration / 宽容性

Toleration 资源旨在应用于 Pod，使其能够容忍特定节点的污点。Toleration 结构体定义如下：

```go
type Toleration struct {
     Key                  string
     Operator             TolerationOperator
     Value                string
     Effect               TaintEffect
     TolerationSeconds    *int64
}
```

TolerationOperator 枚举有以下值：

```go
TolerationOpExists    = "Exists"
TolerationOpEqual     = "Equal"
```

TaintEffect 枚举可以有这些值：

```go
TaintEffectNoSchedule              = "NoSchedule"
TaintEffectPreferNoSchedule        = "PreferNoSchedule"
TaintEffectNoExecute               = "NoExecute"
```

以下两种方法是在 Toleration 结构体上定义的：

- `MatchToleration(tolerationToMatch *Toleration) bool` - 如果两个 Toleration 的 Key、Operator、Value 和 Effect值相同，该方法返回true。

- `ToleratesTaint(taint *Taint) bool` - 如果容忍度能容忍污点，该方法返回 true。容忍器容忍污点的规则如下：
  - **Effect**： 对于空的 Toleration Effect，所有的 Taint 效果将匹配；否则，Toleration 和 Taint Effect 必须匹配。
  - **Operator**： 如果 TolerationOperator 是Exists，所有的 Taint 值将匹配；否则，TolerationOperator 是 Equal，Toleration 和 Taint 值必须匹配。
  - **Key**： 对于空的 Toleration Key，TolerationOperator 必须是Exists，所有 Taint 键（有任何值）都将匹配；否则，Toleration 和 Taint 键必须匹配。

### 知名标签

控制平面在节点上添加标签；使用的知名键及其用法可以在 `core/v1 `包的 `well_known_labels.go` 文件中找到。以下是最著名的。

节点上运行的 kubelet 会填充这些标签：

```go
LabelOSStable      = "kubernetes.io/os"
LabelArchStable    = "kubernetes.io/arch"
LabelHostname      = "kubernetes.io/hostname"
```

当节点在云提供商上运行时，可以设置这些标签，代表（虚拟）机器的实例类型、它的区域和它的地区：

```go
LabelInstanceTypeStable
    = "node.kubernetes.io/instance-type"
LabelTopologyZone
    = "topology.kubernetes.io/zone"
LabelTopologyRegion
    = "topology.kubernetes.io/region"
```

## 用Go写Kubernetes资源

你需要在 Go 中编写 Kubernetes 资源，以创建或更新集群中的资源，可以使用HTTP请求，也可以更广泛地使用 client-go 库。client-go 库将在以后的章节中讨论；但现在，让我们专注于用 Go 编写资源。

要创建或更新资源，你需要为与该资源相关的 Kind 创建结构。例如，要创建 Deployment ，你需要创建 Deployment kind；为此，要初始化 Deployment 结构体，它定义在 API 库的 `apps/v1` 包中。

### 导入包

在使用这个结构体之前，需要导入定义这个结构体的包。正如本章开头所见，包名的模式是 `k8s.io/api/<group>/<version>`。路径的最后部分是一个版本号，但你不应该把它和 Go 模块的版本号混淆。

区别在于，当你导入一个 Go 模块的特定版本时（例如 `k8s.io/klog/v2`），你将使用版本前的部分作为前缀来访问包的符号，而不需要定义任何别名。原因是在库中，v2目录并不存在，而是代表一个分支名称，进入包的文件以 package klog一行开始，而不是 package v2。

相反，在使用 Kubernetes API 库时，版本号是其中一个包的真实名称，进入这个包的文件确实以 package v1 开始。

如果你不为导入定义别名，你将不得不使用版本名来使用这个包的符号。但是，在阅读代码时，单单是版本号是没有意义的，如果你从同一个文件中包含了几个包，最后会出现几个 v1 包名，这是不可能的。

惯例是用组名定义一个别名，或者，如果你想和同一个组的几个版本一起工作，或者你想让别名更清楚地指向一个API group/verson，你可以用 group/verson 创建别名：

```go
import (
    corev1 "k8s.io/api/core/v1"
    appsv1 "k8s.io/api/apps/v1"
)
```

现在你可以实例化一个 Deployment 结构体：

```go
myDep := appsv1.Deployment{}
```

为了编译程序，将需要获取该库。为此，请使用：

```go
$ go get k8s.io/api@latest
```

或者，如果你想使用特定版本的 Kubernetes API（例如，Kubernetes 1.23），请使用：

```go
$ go get k8s.io/api@v0.23
```

所有与 Kinds 相关的结构体首先嵌入两个通用结构体： TypeMeta 和 ObjectMeta。这两个结构体都在 API machinery 的 `/pkg/apis/meta/v1` 包中声明。

### TypeMeta字段

TypeMeta 结构体定义如下：

```go
type TypeMeta struct {
     Kind           string
     APIVersion     string
}
```

你一般不需要自己为这些字段设置值，因为 API machinery 通过维护 Scheme--即 `Group-Version-Kinds` 和 Go 结构体之间的映射，从结构体的类型推断出这些值。请注意，APIVersion 值是将 Group-Version 写成一个包含 `<group>/<version>` 的单一字段的另一种方式（或者对于传统的核心组，只有v1）。

### ObjectMeta字段

ObjectMeta 结构体定义如下（已废弃的字段和内部字段已被删除）：

```go
Type ObjectMeta {
    Name                string
    GenerateName        string
    Namespace           string
    UID                 types.UID
    ResourceVersion     string
    Generation          int64
    Labels              map[string]string
    Annotations         map[string]string
    OwnerReferences     []OwnerReference
    [...]
}
```

API machinery  的 `/pkg/apis/meta/v1` 包为这个结构体的字段定义了 Getters 和 Setters。由于 ObjectMeta 被嵌入到资源结构体中，你可以在资源对象本身中使用这些方法。

#### 名称

这个结构体中最重要的信息是资源的名称。你可以使用 Name 字段来指定资源的确切名称，或者使用 GenerateName 字段来请求 Kubernetes API 为你选择一个独特的名称；它是通过向 GenerateName 值添加一个后缀来建立的，以使其独一无二。

你可以在资源对象上使用  **GetName() string**  和 SetName(name string) 方法来访问其嵌入的 ObjectMeta 的 Name 字段，例如：

```go
configmap := corev1.ConfigMap{}
configmap.SetName("config")
```

#### 命名空间

带命名空间的资源需要被放置到一个特定的命名空间。你可能认为你需要在命名空间字段中指出这个命名空间，但是，当你创建或更新一个资源时，你将定义放置该资源的命名空间，作为请求路径的一部分。第2章已经表明，在 project1 命名空间中创建 pod 的请求是：

```bash
$ curl $HOST/api/v1/namespaces/project1/pods
```


如果你在 Pod 结构中指定了不同于 project1 的命名空间，你会得到错误： *“the namespace of the provided object does not match the namespace sent on the request.”*。由于这些原因，在创建资源时，没有必要设置 Namespace 字段。

#### UID、ResourceVersion 和 Generation

UID 是集群中过去、现在和未来资源的唯一标识符。它由控制平面设置，在资源的生命周期内从不更新。必须用它来引用一个资源，而不是它的 kind, name, 和 namespace，后者可以描述不同时期的各种资源。

ResourceVersion 是一个不透明的值，代表资源的版本。每当资源被更新时，ResourceVersion 就会改变。

这个 ResourceVersion 用于优化并发控制：如果从 Kubernetes API 获得资源的特定版本，修改它然后把它送回 API 更新；API会检查你送回的资源的ResourceVersion 是最后一个。如果在此期间，另一个进程修改了该资源，ResourceVersion 将被修改，你的请求将被拒绝；在这种情况下，你有责任读取新的版本并再次更新它。这与悲观主义的并发控制不同，在悲观主义的并发控制中，你需要在读取资源之前获得一个锁，并在更新之后释放它。

Generation 是一个序列号，可以被资源的控制器用来表示所需状态（Spec）的版本。只有当资源的 Spec 部分被更新时，它才会被更新，而不是其他部分（标签、注解、状态）。控制器通常使用 Status 部分的 ObservedGeneration 字段来指示哪一代被最后处理并反映在状态中。

#### 标签和注解

标签（Labels）和注解（Annotations）在Go中被定义为 map，其中的键和值都是字符串。

即使标签和注解有非常不同的用途，它们也可以用同样的方式来填充。我们将在本节讨论如何填充标签字段，但这也适用于注释。

如果你知道你想添加为标签的键和值，写标签字段的最简单方法是直接写 map，例如：

```go
mylabels := map[string]string{
     "app.kubernetes.io/component": "my-component",
     "app.kubernetes.io/name":      "a-name",
}
```

也可以在现有的 map 上添加标签，比如说：

```go
mylabels["app.kubernetes.io/part-of"] = "my-app"
```

如果你需要从动态值建立标签字段，API Machinery  中提供的标签包提供了一个 Set 类型，可能会有所帮助。

```go
import "k8s.io/apimachinery/pkg/labels"
mylabels := labels.Set{}
```

提供函数和方法来操作这种类型：

- 函数 `ConvertSelectorToLabelsMap` 将选择器字符串转换为Set。

- 函数 `Conflicts` 检查两个 Set 是否有冲突的标签。冲突的标签是指具有相同键但不同值的标签。

- 函数 `Merge` 将把两个 Set 合并成一个 Set。如果两个集合之间有冲突的标签，第二个集合中的标签将被用于产生的集合中

- 函数 `Equals` 检查这两个 Set 是否有相同的键和值。

- 方法 `Has` 表明 Set 是否包含一个键。

- 方法 `Get` 返回 Set 中给定键的值，如果 Set 中没有定义该键，则返回空字符串。

你可以用值来实例化 Set，你可以用单个的值来填充它，就像你用 map 做的一样：

```go
mySet := labels.Set{
     "app.kubernetes.io/component": "my-component",
     "app.kubernetes.io/name":      "a-name",
}
mySet["app.kubernetes.io/part-of"] = "my-app"
```

你可以使用以下方法来访问一个资源的标签和注释：

- GetLabels() map[string]string
- SetLabels(labels map[string]string)
- GetAnnotations() map[string]string
- SetAnnotations(annotations map[string]string)

#### OwnerReferences

当你想表明这个资源被另一个资源所拥有，并且你希望这个资源在所有者不存在时被垃圾收集器收集时，就会在 Kubernetes 资源上设置 OwnerReference。

这在开发 controller 和 operator 时被广泛使用。controller 或operator 创建一些资源来实现另一个资源所描述的规范，它将一个 OwnerReference 放入所创建的资源中，指向给出规范的资源。

例如，Deployment controller 根据在 Deployment 资源中发现的规格创建 ReplicaSet 资源。当你删除 Deployment 时，相关的 ReplicaSet 资源会被垃圾收集器删除，而无需 controller 的任何干预。

OwnerReference 类型定义如下：

```go
type OwnerReference struct {
     APIVersion          string
     Kind                string
     Name                string
     UID                 types.UID
     Controller          *bool
     BlockOwnerDeletion  *bool
}
```

要知道要引用的对象的 UID，你需要从 Kubernetes API 中使用 get（或list）请求获得该对象。

##### 设置 APIVersion 和 Kind

使用 Client-go 库（第4章展示了如何使用它），APIVersion 和 Kind 将不会被设置；你需要在复制对象之前在被引用对象上设置它们，或者直接在 ownerReference 中设置：

```go
import (
      corev1 "k8s.io/api/core/v1"
     metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
// Get the object to reference
pod, err := clientset.CoreV1().Pods(myns).
    Get(context.TODO(), mypodname, metav1.GetOptions{})
If err != nil {
    return err
}
// Solution 1: set the APIVersion and Kind of the Pod
// then copy all information from the pod
pod.SetGroupVersionKind(
    corev1.SchemeGroupVersion.WithKind("Pod"),
)
ownerRef := metav1.OwnerReference{
     APIVersion: pod.APIVersion,
     Kind:       pod.Kind,
     Name:       pod.GetName(),
     UID:        pod.GetUID(),
}
// Solution 2: Copy name and uid from pod
// then set APIVersion and Kind on the OwnerReference
ownerRef := metav1.OwnerReference{
     Name: pod.GetName(),
     UID:  pod.GetUID(),
}
ownerRef.APIVersion, ownerRef.Kind =
    corev1.SchemeGroupVersion.WithKind("Pod").
        ToAPIVersionAndKind()
```

APIVersion包含与 Group 和 Version 相同的信息。你可以从 schema.GroupVersion 类型的 SchemeGroupVersion 变量中获取信息，该变量定义在与资源相关的API库包中（这里 `k8s.io/api/core/v1` 为 Pod 资源）。然后你可以添加 Kind 来创建一个 `schema.GroupVersionKind`。

对于第一个解决方案，你可以在引用的对象上使用 SetGroupVersionKind 方法，从 GroupVersionKind 中设置 APIVersion 和 Kind。对于第二个解决方案，使用 GroupVersionKind 上的 ToAPIVersionAndKind 方法来获取相应的 APIVersion 和 Kind 值，然后再把它们移到 OwnerReference 中。

第5章描述了 **API Machinery** 以及与 Group、Version 和 Kinds 相关的所有类型和方法。OwnerReference 结构体还包含两个可选的布尔字段： Controller 和 BlockOwnerDeletion。

##### 设置Controller

Controller 字段表示被引用的对象是否是管理 Controller（或 operator）。Controller 或 operator 必须在拥有的资源上将此值设置为 "true"，以表明它正在管理这个拥有的资源。

Kubernetes API 将拒绝在同一资源上添加两个 Controller 设置为 true 的 OwnerReferences。这样一来，一个资源就不可能由两个不同的 Controller 管理。

请注意，这与被各种资源所拥有是不同的。一个资源可以有不同的所有者；在这种情况下，当所有的所有者都被删除时，该资源将被删除，与哪个所有者是控制器无关，如果有的话。

Controller 的这种唯一性对于那些可以 "采用" 资源的 Controller 来说是很有用的。例如，ReplicaSet 可以采用与 ReplicaSet 的选择器相匹配的现有Pod，但前提是该 Pod 还没有被另一个 ReplicaSet 或另一个 Controller 控制。

这个字段的值是一个指向布尔值的指针。你可以声明一个布尔值，并在设置控制器字段时影响其地址，或者使用 Kubernetes Utils Library 的 BoolPtr 函数：

```go
// Solution 1: declare a value and use its address
controller := true
ownerRef.Controller = &controller

// Solution 2: use the BoolPtr function
import (
     "k8s.io/utils/pointer"
)
ownerRef.Controller = pointer.BoolPtr(true)
```

##### 设置BlockOwnerDeletion

OwnerReference 对于 Controller 或其他进程来说是非常有用的，因为他们不需要关心自有资源的删除问题：当所有者被删除时，自有资源将被 Kubernetes 垃圾收集器删除。

这种行为是可配置的。在资源上使用删除操作时，你可以使用传播策略（**Propagation Policy** ）选项：

1. **Orphan**：向 Kubernetes API 表示将拥有的资源变成孤儿，因此它们不会被垃圾收集器删除。
2. **Background**： 指示 Kubernetes API 在所有者资源被删除后立即从 DELETE 操作中返回，而不是等待自有资源被垃圾收集器删除。
3. **Foreground**： 指示 Kubernetes API 在所有者和 BlockOwnerDeletion 设置为 "true "的自有资源被删除后，从 DELETE 操作中返回。Kubernetes API 将不会等待其他拥有的资源被删除。

因此，如果你正在编写一个 Controller 或其他进程，需要等待所有拥有的资源被删除，该进程将需要在所有拥有的资源上将 BlockOwnerDeletion 字段设置为true，并在删除所有者资源时使用 Foreground 传播策略。

### 规格和状态

在常见的 Type 和 Metadata 之后，资源定义一般由两部分组成：Spec 和 Status。

请注意，并非所有资源都是如此。例如，核心的 ConfigMap 和 Secret 资源，仅举几例，不包含 Spec 和 Status 部分。更普遍的是，包含配置数据的资源，不由任何 Controller 管理，不包含这些字段。

Spec 是用户将定义的部分，它表示用户所期望的状态。管理该资源的 Controller 将读取Spec，它将根据 Spec 在集群上创建、更新或删除资源，并将其操作的状态检索到资源的 Status 部分。Controller 用于读取 Spec、应用于集群并检索状态的这个过程被称为 Reconcile Loop。

### 与编写 YAML 清单 的比较

当您编写 Kubernetes 清单 以用于kubectl时：

- 清单以 apiVersion 和 kind 开始。
- metadata 字段包含该资源的所有元数据。

- Spec 和 Status 字段（或其他）紧随其后。


当您用 Go 编写 Kubernetes 结构体时，会发生以下情况：

- 结构体的类型决定了apiVersion 和 Kind；不需要指定它们。
- metadata 可以通过嵌入metav1.ObjectMeta 结构体来定义，也可以通过在资源上使用元数据设置器（*metadata setters* ）。

- Spec 和 Status 字段（或其他）紧随其后，使用它们自己的 Go 结构体或其他类型。


举个例子，下面是用 YAML 清单和 Go 定义 Pod 的方法。在 YAML 中：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
  - component: my-component,
spec:
  containers:
  - image: nginx
    name: nginx
```

在 Go 中，当你为元数据使用设置器时：

```yaml
pod := corev1.Pod{
   Spec: corev1.PodSpec{
      Containers: []corev1.Container{
        {
            Name:  "runtime",
            Image: "nginx",
         },
      },
   },
}
pod.SetName("my-pod")
pod.SetLabels(map[string]string{
     "component": "my-component",
})
```

或者，在 Go 中，当你将 `metav1.ObjectMeta` 结构体嵌入到 Pod 结构体中时：

```go
pod2 := corev1.Pod{
     ObjectMeta: metav1.ObjectMeta{
          Name: "nginx",
          Labels: map[string]string{
               "component": "mycomponent",
          },
     },
     Spec: corev1.PodSpec{
          Containers: []corev1.Container{
               {
                    Name:  "runtime",
                    Image: "nginx",
               },
          },
     },
}
```

### 完整的例子

这个完整的例子使用到此为止学到的概念，使用 POST 请求在集群上创建一个 Pod。

- ➊ 使用 Go 结构体建立一个 Pod 对象，如本章前面所示
- ➋ 使用序列化器将 Pod 对象序列化为 JSON 格式（详见第五章）。

- ➌ 建立一个 HTTP POST 请求，其主体包含要创建的 Pod，并以 JSON 形式序列化

- ➍ 用建立的请求调用 Kubernetes API

- ➎ 从响应中获取主体

- ➏ 根据响应的状态代码：


如果请求返回 2xx 状态代码：

- ➐ 将响应主体反序列化为一个Pod Go结构体
- ➑ 将创建的Pod对象显示为JSON格式的信息；


否则：

- ➒ 将响应体反序列化为一个状态Go结构体。

- ➓ 将Status对象显示为JSON格式，以获取信息：

```go
package main
import (
  "bytes"
  "encoding/json"
  "fmt"
  "io"
  "net/http"
  corev1 "k8s.io/api/core/v1"
  metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
  "k8s.io/apimachinery/pkg/runtime"
  "k8s.io/apimachinery/pkg/runtime/schema"
  "k8s.io/apimachinery/pkg/runtime/serializer/json"
)
func createPod() error {
  pod := createPodObject() ➊
  serializer := getJSONSerializer()
  postBody, err := serializePodObject(serializer, pod) ➋
  if err != nil {
    return err
  }
  reqCreate, err := buildPostRequest(postBody) ➌
  if err != nil {
    return err
  }
  client := &http.Client{}
  resp, err := client.Do(reqCreate) ➍
  if err != nil {
    return err
  }
  defer resp.Body.Close()
  body, err := io.ReadAll(resp.Body) ➎
  if err != nil {
    return err
  }
  if resp.StatusCode < 300 { ➏
    createdPod, err := deserializePodBody(serializer, body) ➐
    if err != nil {
      return err
    }
    json, err := json.MarshalIndent(createdPod, "", "  ")
    if err != nil {
      return err
    }
    fmt.Printf("%s\n", json) ➑
  } else {
    status, err := deserializeStatusBody(serializer, body) ➒
    if err != nil {
      return err
    }
    json, err := json.MarshalIndent(status, "", "  ")
    if err != nil {
      return err
    }
    fmt.Printf("%s\n", json) ➓
  }
  return nil
}
func createPodObject() *corev1.Pod { ➊
     pod := corev1.Pod{
          Spec: corev1.PodSpec{
               Containers: []corev1.Container{
                    {
                         Name:  "runtime",
                         Image: "nginx",
                    },
               },
          },
     }
     pod.SetName("my-pod")
     pod.SetLabels(map[string]string{
          "app.kubernetes.io/component": "my-component",
          "app.kubernetes.io/name":      "a-name",
     })
     return &pod
}
func serializePodObject( ➋
     serializer runtime.Serializer,
     pod *corev1.Pod,
) (
     io.Reader,
     error,
) {
     var buf bytes.Buffer
     err := serializer.Encode(pod, &buf)
     if err != nil {
          return nil, err
     }
     return &buf, nil
}
func buildPostRequest( ➌
     body io.Reader,
) (
     *http.Request,
     error,
) {
     reqCreate, err := http.NewRequest(
          "POST",
          "http://127.0.0.1:8001/api/v1/namespaces/default/pods",
          body,
     )
     if err != nil {
          return nil, err
     }
     reqCreate.Header.Add(
"Accept",
"application/json",
)
     reqCreate.Header.Add(
"Content-Type",
"application/json",
)
     return reqCreate, nil
}
func deserializePodBody( ➐
     serializer runtime.Serializer,
     body []byte,
) (
     *corev1.Pod,
     error,
) {
     var result corev1.Pod
     _, _, err := serializer.Decode(body, nil, & result)
     if err != nil {
          return nil, err
     }
     return &result, nil
}
func deserializeStatusBody( ➒
     serializer runtime.Serializer,
     body []byte,
) (
     *metav1.Status,
     error,
) {
     var status metav1.Status
     _, _, err := serializer.Decode(body, nil, & status)
     if err != nil {
          return nil, err
     }
     return & status, nil
}
func getJSONSerializer() runtime.Serializer {
     scheme := runtime.NewScheme()
     scheme.AddKnownTypes(
          schema.GroupVersion{
               Group:   "",
               Version: "v1",
          },
          &corev1.Pod{},
          &metav1.Status{},
     )
     return json.NewSerializerWithOptions(
          json.SimpleMetaFactory{},
          nil,
          scheme,
          json.SerializerOptions{},
     )
}
```

## 总结

在本章中，你已经发现了在 Go 中与 Kubernetes 合作的第一个库-- API库。它本质上是一个 Go 结构体的集合，用于声明 Kubernetes 资源。本章还探讨了 API Machinery  中定义的所有资源共有的元数据字段的定义。

在本章的最后，有一个使用 API Machinery  构建Pod定义的程序实例，然后通过使用HTTP请求调用API服务器在集群中创建这个Pod。

下一章将探讨其他基本库--API Machinery  和 client-go，你将不再需要建立HTTP请求。
