---
title: "[第9章]使用自定义资源"
linkTitle: "使用自定义资源"
weight: 90
date: 2021-02-01
description: >
  使用自定义资源
---

在上一章中，你已经看到了如何使用 CustomResourceDefinition 资源声明一个由 Kubernetes API 提供的新的自定义资源，以及如何使用 kubectl 创建这个自定义资源的新实例。但就目前而言，你还没有任何 Go 库可以让你处理自定义资源的实例。

本章探讨了在 Go 中处理自定义资源的各种可能性：

- 为自定义资源的专用 Clientset 生成代码。
- 使用 API  Machinery 的非结构化（**unstructured**）包和 Client-go 库的 dynamic 包。

## 生成 Clientset

存储库 https://github.com/kubernetes/code-generator 包含 Go 代码生成器。第三章的 "包的内容" 部分包含了对这些生成器的一个非常快速的概述，其中对API库的内容进行了探讨。

要使用这些生成器，你需要首先为自定义资源所定义的种类编写 Go 结构体。在这个例子中，您将为 MyResource 和 MyResourceList 类型编写结构体。

为了与 API 库中的组织结构体保持一致，您将在 types.go 文件中编写这些类型，放置在 `pkg/apis/<group>/<version>/` 目录中。

同样地，为了与生成器正常工作，你项目的根目录必须在你项目的 Go 包之后，以 Go 包命名的目录。例如，如果项目的包是 `github.com/myid/myproject`（在项目的 go.mod 文件的第一行定义），项目的根目录必须在 `github.com/myid/myproject/` 目录下。

作为例子，让我们开始一个新项目。你可以在你选择的目录中执行这些命令，一般是包含你所有 Go 项目的目录。

```bash
$ mkdir -p github.com/myid/myresource-crd
$ cd github.com/myid/myresource-crd
$ go mod init github.com/myid/myresource-crd
$ mkdir -p pkg/apis/mygroup.example.com/v1alpha1/
$ cd pkg/apis/mygroup.example.com/v1alpha1/
```

然后，在这个目录中，你可以创建 `types.go` 文件，其中包含 kind 的结构体定义。下面是与前一章 CRD 中定义的模式相匹配的结构体定义。

```go
package v1alpha1
import (
     metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
     "k8s.io/apimachinery/pkg/util/intstr"
)
type MyResource struct {
     metav1.TypeMeta   `json:",inline"`
     metav1.ObjectMeta `json:"metadata,omitempty"`
     Spec MyResourceSpec `json:"spec"`
}
type MyResourceSpec struct {
     Image  string             `json:"image"`
     Memory resource.Quantity `json:"memory"`
}
type MyResourceList struct {
     metav1.TypeMeta `json:",inline"`
     metav1.ListMeta `json:"metadata,omitempty"`
     Items []MyResource `json:"items"`
}
```

现在，你需要运行两个生成器：

- `deepcopy-gen` - 这将为每个 Kind 结构体生成一个 DeepCopyObject() 方法，这是这些类型实现 `runtime.Object` 接口所需要的。
- `client-gen` - 这将为该分组/版本生成 clientset。

## 使用deepcopy-gen

### 安装 deepcopy-gen

要安装 deepcopy-gen 可执行文件，你可以使用 go install 命令：

```go
go install k8s.io/code-generator/cmd/deepcopy-gen@v0.24.4
```

你可以使用 @latest 标签来使用 Kubernetes 代码的最新版本，或者选择一个特定的版本。

### 添加注解

deepcopy-gen 生成器需要注释才能工作。它首先需要 `//+k8s:deepcopy-gen=package` 注解在包级别上被定义。这个注解要求 deepcopy-gen 为包的所有结构体生成 deepcopy 方法。

为此，你可以在 types.go 所在的目录中创建一个 doc.go 文件，以添加这个注释：

```go
// pkg/apis/mygroup.example.com/v1alpha1/doc.go
// +k8s:deepcopy-gen=package
package v1alpha1
```

默认情况下，deepcopy-gen 将生成 DeepCopy() 和 DeepCopyInto() 方法，但没有 DeepCopyObject()。为此，你需要在每个种类结构体之前添加另一个注释（`//+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`）。

```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type MyResource struct {
[...]
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type MyResourceList struct {
[...]
```

## 运行 deepcopy-gen

生成器需要一个文件，包含在生成的文件开头添加的文本（一般是许可证）。为此，你可以创建一个空文件（或者用你喜欢的内容，别忘了这些文字应该是Go注释），命名为 `hack/boilerplate.go.txt`。

你需要运行 go mod tidy 来使生成器工作（如果你喜欢 vendor Go dependencies，也可以运行 `go mod vendor`）。最后，你可以运行 deepcopy-gen 命令，它将生成一个文件 `pkg/apis/mygroup.example.com/v1alpha1/zz_generated.deepcopy.go`：

```go
$ go mod tidy
$ deepcopy-gen --input-dirs github.com/myid/myresource-crd/pkg/apis/mygroup.example.com/v1alpha1
     -O zz_generated.deepcopy
     --output-base ../../..
     --go-header-file ./hack/boilerplate.go.txt
```

注意 ".../.../... " 作为 output-base 。它将把 output-base 放在你为该项目创建目录的目录中：

```bash
$ mkdir -p github.com/myid/myresource-crd
```

如果你创建的子目录数量与这三个不同，你将需要对其进行调整。

在这一点上，你的项目应该有以下文件结构：

```
├── go.mod
├── hack
│   └── boilerplate.go.txt
├── pkg
│   └── apis
│       └── mygroup.example.com
│           └── v1alpha1
│               ├── doc.go
│               ├── types.go
│               └── zz_generated.deepcopy.go
```

## 使用 client-gen

### 安装 client-go

要安装 client-gen 可执行文件，你可以使用 go install命令：

```go
go install k8s.io/code-generator/cmd/client-gen@v0.24.4
```

你可以使用 @latest 标签来使用 Kubernetes 代码的最新版本，如果你想以可复制的方式运行该命令，则可以选择一个特定的版本。

### 添加注解

你需要向 types.go 文件中定义的结构体添加注释，以表明你想为哪些类型定义 clientset。要使用的注释是 `// +genclient`。

- `// +genclient`（没有选项）将要求 client-gen 为一个有命名空间的资源生成 Clientset。
- `// +genclient:nonNamespaced` 将为非namespaced资源生成 Clientset。
- `+genclient:onlyVerbs=create,get` 将只生成这些动词，而不是默认生成的所有动词。
- `+genclient:skipVerbs=watch` 将生成除这些动词以外的所有动词，而不是默认的所有动词。
- `+genclient:noStatus` - 如果注释的结构体中存在 Status 字段，将生成一个 updateStatus 函数。通过这个选项，你可以禁止生成 updateStatus 函数（注意，如果 Status 字段不存在，就没有必要）。

你所创建的自定义资源是有命名空间的，所以你可以使用注解而不使用选项：

```go
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +genclient
type MyResource struct {
[...]
```

### 添加 AddToScheme 函数

生成的代码依赖于定义在 resource 包中的 AddToScheme 函数。为了与 API 库中的惯例保持一致，你需要在 `register.go` 文件中编写这个函数，放在目录 `**pkg/apis/<group>/<version>/**` 中。

通过从 Kubernetes API 库的本地 Kubernetes 资源中获取作为模板的 register.go 文件，你会得到以下文件。唯一的变化是 group name（❶）、version name（❷）和要注册到该 schema 的资源列表（❸）。

```go
package v1alpha1
import (
     metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
     "k8s.io/apimachinery/pkg/runtime"
     "k8s.io/apimachinery/pkg/runtime/schema"
)
const GroupName = "mygroup.example.com"          ❶
var SchemeGroupVersion = schema.GroupVersion{
     Group: GroupName,
     Version: "v1alpha1",                         ❷
}
var (
     SchemeBuilder      = runtime.NewSchemeBuilder(addKnownTypes)
     localSchemeBuilder = &SchemeBuilder
     AddToScheme        = localSchemeBuilder.AddToScheme
)
func addKnownTypes(scheme *runtime.Scheme) error {
     scheme.AddKnownTypes(SchemeGroupVersion,
          &MyResource{},                         ❸
          &MyResourceList{},
     )
     metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
     return nil
}
```

### 运行 client-go

client-gen 需要一个包含文本（通常是许可证）的文件，添加在生成文件的开头。你将使用与 deepcopy-gen 相同的文件： `hack/boilerplate.go.txt`。

你可以运行 client-gen 命令，它将在 `pkg/clientset/clientset` 目录下生成文件：

```bash
client-gen \
     --clientset-name clientset
     --input-base ""
     --input github.com/myid/myresource-crd/pkg/apis/mygroup.example.com/v1alpha1
     --output-package github.com/myid/myresource-crd/pkg/clientset
     --output-base ../../..
     --go-header-file hack/boilerplate.go.txt
```

注意 `".../.../..."` 作为 output-base 。它将把 output-base 放在你为该项目创建目录的目录中：

```go
$ mkdir -p github.com/myid/myresource-crd
```

如果你创建的子目录数量与三个不同，你将需要对其进行调整。

注意，当你更新自定义资源的定义时，你将需要再次运行这个命令。建议把这个命令放在 Makefile 中，以便在每次修改定义自定义资源的文件时自动运行它。

### 使用生成的 clientset

现在，clientset 已经生成，并且类型实现了 runtime.Object 接口，你可以像处理本地 Kubernetes 资源一样处理自定义资源。例如，这段代码将使用专用的 clientset 来列出默认命名空间上的自定义资源：

```go
import (
     "context"
     "fmt"
     "github.com/myid/myresource-crd/pkg/clientset/clientset"
     metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
     "k8s.io/client-go/tools/clientcmd"
)
config, err :=
     clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
          clientcmd.NewDefaultClientConfigLoadingRules(),
          nil,
     ).ClientConfig()
if err != nil {
     return err
}
clientset, err := clientset.NewForConfig(config)
if err != nil {
     return err
}
list, err := clientset.MygroupV1alpha1().
     MyResources("default").
     List(context.Background(), metav1.ListOptions{})
if err != nil {
     return err
}
for _, res := range list.Items {
     fmt.Printf("%s\n", res.GetName())
}
```

### 使用生成的 fake Clientset

客户端生成工具也会生成 fake Clientset，你可以像使用 Client-go 库中的 fake Clientset 一样使用它。更多信息，请参见第7章的 "fake Clientset" 部分。

### 使用非结构化包和动态客户端

Unstructured 和 UnstructuredList 类型被定义在API Machinery 的 unstructured 包中。要使用的导入方式如下：

```go
import (
     "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
)
```

这些类型可以用来表示任何 Kubernetes Kind，无论是列表还是非列表。

#### 非结构化类型

非结构化类型被定义为一个结构体，包含一个唯一的对象字段：

```go
type Unstructured struct {
     // Object is a JSON compatible map with
     // string, float, int, bool, []interface{}, or
     // map[string]interface{}
     // children.
     Object map[string]interface{}
}
```

使用这种类型，可以定义任何 Kubernetes 资源，而不必使用类型结构体（例如，API 库中的 Pod 结构体）。

该类型定义了 Getters 和 Setters 方法，以访问 TypeMeta 和 ObjectMeta 字段中的通用字段，这些字段对代表 Kubernetes Kinds 的所有结构体都是通用的。

#### 访问 TypeMeta 字段的 Getters 和 Setters

APIVersion 和 Kind Getters/Setters 可以被用来直接获取和设置 TypeMeta 的 apiVersion 和 Kind 字段。

GroupVersionKind Getters/Setters 可用于将对象中指定的 apiVersion 和 Kind 转换为 GroupVersionKind 值。

```go
GetAPIVersion() string
GetKind() string
GroupVersionKind() schema.GroupVersionKind
SetAPIVersion(version string)
SetKind(kind string)
SetGroupVersionKind(gvk schema.GroupVersionKind)
```

#### 访问 ObjectMeta 字段的Getters和Setters

Getters 和 Setters 被定义为 ObjectMeta 结构体的所有字段。该结构体的细节可以在第三章的 "ObjectMeta字段" 部分找到。

作为例子，访问 Name 字段的 getter 和 setter是 `GetName() string` 和 `SetName(name string)`。

#### 创建和转换的方法

- `NewEmptyInstance() runtime.Unstructured` - 这将返回一个新的实例，只有 apiVersion 和 kind 字段是从接收者那里复制的。

- `MarshalJSON() ([]byte, error) `- 这返回接收器的JSON表示。

- `UnmarshalJSON(b []byte) error` - 这将用传递的JSON表示法填充接收器。

- `UnstructuredContent() map[string]interface{} ` - 这将返回接收器的Object字段的值。

- `SetUnstructuredContent()`

  content map[string]interface{}，

  )

  这将设置接收方的 Object 字段的值。

- `IsList() bool` - 如果接收方描述的是一个列表，通过检查项目字段是否存在，并且是一个数组，返回true。

- `ToList() (*UnstructuredList, error) `- 这将接收器转换为一个非结构化列表。

#### 访问非meta字段的助手

以下 helper 可以用来获取和设置非结构化实例的 Object 字段中的特定字段的值。

> 注意，这些 helper 是函数，而不是非结构化类型的方法。

它们都接受：

- 第一个参数 obj 是 `map[string]interface{}` 类型，用于传递非结构化实例的 Object 字段、
- 最后一个参数 fields，类型为 `...string`，用于传递导航到对象的键。注意，不支持数组/slice 的语法。


**Setters** 接受第二个参数，给出给定对象中特定字段的设置值。Getters 返回三个值：

- 要求的字段的值，如果可能的话
- 一个布尔值，表示是否已经找到了所请求的字段

- 如果找到了字段，但不属于请求的类型，则为错误。


helper 函数的名称是：

- RemoveNestedField - 这将删除请求的字段

- NestedFieldCopy, NestedFieldNoCopy - 返回请求字段的副本或原始值

- NestedBool, NestedFloat64, NestedInt64, NestedString, SetNestedField - 获取和设置 bool/float64/int64/string 字段。

- NestedMap, SetNestedMap - 获取和设置 `map[string]interface{}` 类型的字段。

- NestedSlice, SetNestedSlice - 获取并设置 `[]interface{}` 类型的字段。

- NestedStringMap, SetNestedStringMap - 获取和设置 `map[string]string` 类型的字段。

- NestedStringSlice, SetNestedStringSlice - 获取和设置 `[]string` 类型的字段。

例子:

作为例子，下面是一些定义 MyResource 实例的代码：

```go
import (
     myresourcev1alpha1 "github.com/myid/myresource-crd/pkg/apis/mygroup.example.com/v1alpha1"
     "k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
)
func getResource() (*unstructured.Unstructured, error) {
     myres := unstructured.Unstructured{}
     myres.SetGroupVersionKind(
          myresourcev1alpha1.SchemeGroupVersion.
               WithKind("MyResource"))
     myres.SetName("myres1")
     myres.SetNamespace("default")
     err := unstructured.SetNestedField(
          myres.Object,
          "nginx",
          "spec", "image",
     )
     if err != nil {
          return err
     }
     // Use int64
     err = unstructured.SetNestedField(
          myres.Object,
          int64(1024*1024*1024),
          "spec", "memory",
     )
     if err != nil {
          return err
     }
     // or use string
     err = unstructured.SetNestedField(
          myres.Object,
          "1024Mo",
          "spec", "memory",
     )
     if err != nil {
          return err
     }
     return &myres, nil
}
```

#### UnstructuredList 类型

UnstructuredList 类型被定义为一个结构体，包含一个 Object 字段和一个非结构化实例的 slice 作为项目：

```go
type UnstructuredList struct {
     Object map[string]interface{}
     // Items is a list of unstructured objects.
     Items []Unstructured
}
```

TBD

### Dynamic Client

正如你在第6章中所看到的，Client-go 为客户端提供了与 Kubernetes API 协同工作的能力：Clientet 用于访问类型化资源，REST 客户端用于对API进行低级别的REST调用，Discovery 客户端用于获取API提供的资源信息。

它提供了另一个客户端，即 Dynamic Client / 动态客户端，用于处理非类型的资源，用非结构化类型描述。

#### 获取动态客户端

**dynamic** 包提供了创建 `dynamic.Interface` 类型的动态客户端的函数。

## 总结

本章探讨了在 Go 中使用自定义资源的各种解决方案。

第一个解决方案是根据自定义资源的定义生成Go代码，为这个特定的自定义资源定义生成一个客户端，这样你就可以像处理本地 Kubernetes 资源一样处理自定义资源实例。

另一个解决方案是使用 Client-go 库的动态客户端，并依靠非结构化类型来定义自定义资源。
