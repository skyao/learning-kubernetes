---
title: "[第8章]用自定义资源定义扩展Kubernetes API"
linkTitle: "用自定义资源定义扩展Kubernetes API"
weight: 80
date: 2021-02-01
description: >
  client-go类库
---

在本书的第一部分，你了解到Kubernetes API是按分组组织的。这些分组包含一个或多个资源，每个资源都是有版本的。

要在Go中使用 Kubernetes API，有两个基本库。API Machinery 提供了与API通信的工具，独立于API提供的资源。API库提供了 Kubernetes API 提供的本地Kubernetes 资源的定义，以便与 API Machinery 一起使用。

client-go 库利用 API Machinery 和 API库来提供对 Kubernetes API 的访问。

Kubernetes API 通过其自定义资源定义（CRD）机制是可扩展的。

CustomResourceDefinition 是一种特定的 Kubernetes 资源，用于以动态方式定义由API提供的新的 Kubernetes 资源。

为 Kubernetes 定义新资源是用来代表特定领域的资源，例如数据库、CI/CD作业或证书。

结合自定义控制器，这些自定义资源可以由控制器在集群中实现。

像其他资源一样，你可以获得、列出、创建、删除和更新这类资源。CRD资源是一种非命名空间的资源。

该CRD资源被定义在 `apiextensions.k8s.io/v1` 分组和版本中。访问该资源的 HTTP 路径尊重访问非核心和非命名资源的标准格式，即 `/apis/<group>/<version>/<plural_resource_name>`，并且是 `/apis/apiextensions.k8s.io/v1/customresourcedefinitions/`。

这个资源的 go 定义并不像其他本地资源那样在 API 库中声明，而是在 `apiextensions-apiserver` 库中。要从Go资源中访问定义，你需要使用以下导入：

```go
import (
  "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
)
```

## 在Go中执行操作

要使用 Go 对 CRD 资源进行操作，你可以使用 clientset，类似于 client-go 库提供的 clientset，但包含在 `apiextensions-apiserver` 库中。

要使用这个 clientset ，你需要使用下面的导入：

```go
import (
  "k8s.io/apiextensions-apiserver/pkg/client/clientset/clientset"
)
```

你可以使用这个 CRD clientset，与使用 Client-go clientset 的方式完全相同。

作为一个例子，你可以在这里列出申报到一个集群中的CRD：

```go
import (
     "context"
     "fmt"
     "k8s.io/apiextensions-apiserver/pkg/client/clientset/clientset"
     metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)
// config is a standard rest.Config defined in client-go
clientset, err := clientset.NewForConfig(config)
if err != nil {
     return err
}
ctx := context.Background()
list, err := clientset.ApiextensionsV1().
     CustomResourceDefinitions().
     List(ctx, metav1.ListOptions{})
```

## CustomResourceDefinition的详细介绍

CustomReourceDefinition Go 结构体的定义是：

```go
type CustomResourceDefinition struct {
     metav1.TypeMeta
     metav1.ObjectMeta
     Spec CustomResourceDefinitionSpec
     Status CustomResourceDefinitionStatus
}
```

像其他 Kubernetes 资源一样，CRD 资源嵌入了 TypeMeta 和 ObjectMeta 结构。ObjectMeta 字段中的 Name 的值必须等于 `<Spec.Names.Plural> + “. ” + <Spec.Group>`。

像被 Controller 或 Operator 管理的资源一样，该资源包含一个 Spec 结构体来定义期望的状态，以及一个 Status 结构体来包含 Controller 或 Operator 描述的资源的状态。

```go
type CustomResourceDefinitionSpec struct {
     Group           string
     Scope           ResourceScope
     Names           CustomResourceDefinitionNames
     Versions        []CustomResourceDefinitionVersion
     Conversion      *CustomResourceConversion
     PreserveUnknownFields bool
}
```

Spec 的 Group 字段表示资源所在的分组的名称。

Scope 字段表示该资源是命名空间还是非命名空间。ResourceScope 类型包含两个值： ClusterScoped 和 NamespaceScoped。

### 命名资源

name 字段表示资源和相关信息的不同名称。这些信息将被 Kubernetes API 的发现机制使用，以便能够在发现调用的结果中包括 CRD。CustomResourceDefinitionNames 类型被定义为：

```go
type CustomResourceDefinitionNames struct {
     Plural         string
     Singular       string
     ShortNames     []string
     Kind           string
     ListKind       string
     Categories     []string
}
```

- **Plural**（复数）是资源的小写复数名称，在访问资源的URL中使用--例如，pods。这个值是必须的。
- **Singular**（单数）是资源的小写单数名称。例如，pod。这个值是可选的，如果不指定，则使用Kind字段的小写值。
- **ShortNames** 是资源的小写短名列表，可用于在 `kubectl get <shortname>` 等命令中调用该资源。举例来说，服务资源声明了svc的短名，所以你可以执行kubectl get svc 而不是 kubectl get services。
- **Kind** 是资源的 CamelCase 和单数名称，在资源序列化时使用--例如，Pod或ServiceAccount。这个值是必须的。
- **ListKind** 是该资源的列表序列化过程中使用的名称--例如，PodList。这个值是可选的，如果没有指定，将使用以List为后缀的Kind值。
- **Categories** 是该资源所属的分组资源的列表，可以被 `kubectl get <category>` 这样的命令使用。所有类别是最著名的类别，但也存在其他类别（api-extensions），你可以定义自己的类别名称。

### 资源版本的定义

到此为止提供的所有信息都对资源的所有版本有效。

**Versions** 字段包含特定版本的信息，是一个定义列表，每个版本的资源都有一个定义。CustomResourceDefinitionVersion 类型被定义为：

```go
type CustomResourceDefinitionVersion struct {
     Name                      string
     Served                    bool
     Storage                   bool
     Deprecated                bool
     DeprecationWarning        *string
     Schema                    *CustomResourceValidation
     Subresources              *CustomResourceSubresources
     AdditionalPrinterColumns  []CustomResourceColumnDefinition
}
```

**Name** 表示版本名称。Kubernetes 资源使用标准的版本格式：`v<number>[(alpha|beta)<number>]`，但你可以使用任何你想要的格式。

**Served** 布尔值表示该特定版本是否必须由 API server 提供服务。如果不是，该版本仍然被定义，并且可以作为 Storage 使用（见下文），但用户不能创建或获得该特定版本的资源实例。

**Storage** 布尔值表示该特定版本是否是用于持久化资源的版本。恰好有一个版本必须定义这个字段为 true 。你在第五章的 "转换" 部分已经看到，转换功能存在于同一资源的不同版本之间。用户可以在任何可用的版本中创建资源，API服务器会将其转换为 `Storage=true` 的版本，然后再将数据持久化在 etcd 中。

**Deprecated** 布尔值表示该资源的特定版本是否被废弃。如果为真，服务器将为这个版本的响应添加一个 Warning 头。

**DeprecationWarning** 是当 Deprecated 为 true 时返回给调用者的警告信息。如果 Deprecated 为真，且该字段为零，则会发送一个默认的警告信息。

**Schema** 描述了这个版本的资源的模式。该模式用于在创建或更新资源时验证发送到 API 的数据。这个字段是可选的。模式将在 "资源的模式" 一节中详细讨论。

**Subresources** 定义了将为这个版本的资源提供的子资源。这个字段是可选的。该字段的类型定义为：

```go
type CustomResourceSubresources struct {
     Status *CustomResourceSubresourceStatus
     Scale *CustomResourceSubresourceScale
}
```

- 如果 Status 不是 nil，"/status" 子资源将被提供。

- 如果 Scale 不是 nil，"/scale" 子资源将被提供。


AdditionalPrinterColumns是当用户要求表的输出格式时要返回的额外列的列表。这个字段是可选的。关于表格输出格式的更多信息可以在第二章 "以表格形式获取结果" 部分找到。你将在本章的 "附加打印机列" 部分看到如何定义附加打印机列。

## 资源的模式

特定版本的资源的模式是用 OpenAPI v3 模式格式来定义的。这种格式的完整描述不在本书的范围之内。这里有一个简短的介绍，可以帮助你为你的资源定义模式。

自定义资源通常有 Spec 部分和 Status 部分。为此，你必须声明对象类型的顶层模式，并使用属性声明字段。在这个例子中，顶层模式将有Spec和Status属性。然后你可以递归地描述每个属性。

Spec 和 Status 字段也将是对象类型的，并且包含属性。可接受的数据类型是：

- string
- number
- integer
- boolean
- array
- object

可以对字符串和数字类型给出具体的格式：

- **string**: date, date-time, byte, int-or-string
- **number**: float, double, int32, int64

在对象中，必须的字段用 required 属性表示。

你可以声明属性只接受一组值，使用枚举属性。要声明值的 map，你需要使用对象类型并指定 additionalProperties 属性。这个属性可以接受一个 true 值（表示条目可以有任何类型），或者通过给出一个类型作为值来定义 map 的条目类型：

```go
type: object
additionalProperties:
  type: string
```

或者

```go
type: object
additionalProperties:
  type: object
  properties:
    code:
      type: integer
    text:
      type: string
```

在声明数组时，你必须定义数组中的 item 的类型：

```go
type: array
items:
  type: string
```

作为例子，这里是一个自定义资源的模式，包含 Spec 中的三个字段（image, replicas, and port），以及 Status 中的一个 state 字段。

```go
schema:
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        properties:
          image:
            type: string
          replicas:
            type: integer
          port:
            type: string
            format: int-or-string
        required: [image,replicas]
      status:
        type: object
        properties:
          state:
            type: string
            enum: [waiting,running]
```

## 部署自定义资源定义

要将新的资源定义部署到集群中，你需要在 CRD 对象上执行 Create 操作。你可以使用 apiextensions-apiserver 库提供的客户端，并使用你在前几节看到的结构体在 Go 中编写资源定义，或者你可以创建一个 YAML 文件并使用 kubectl "apply " 它。

举个例子，你将在 `mygroup.example.com` 分组中创建一个名为 `myresources` 的新资源，其版本为 `v1alpha1`，使用 YAML 格式和 kubectl 来部署它。

```go
apiVersion: apiextensions.k8s.io/v1        ❶
kind: CustomResourceDefinition             ❷
metadata:
  name: myresources.mygroup.example.com    ❸
spec:
  group: mygroup.example.com               ❹
  scope: Namespaced                        ❺
  names:
    plural: myresources                    ❻
    singular: myresource                   ❼
    shortNames:
    - my                                   ❽
    - myres
    kind: MyResource
    categories:
    - all                                  ❾
  versions:
  - name: v1alpha1                         ❿
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object                       ⓫
```

- ❶ CRD 资源的分组和版本

- ❷ 该 CRD 资源的种类（kind）

- ❸ 新资源的完整名称，包括其组别

- ❹ 新资源所属的组别

- ❺ 新资源可以在特定的命名空间创建

- ❻ 新资源的复数名称，在访问该新资源的路径中使用

- ❼ 资源的单数名称，你可以使用 kubectl get myresource

- ❾ 新资源的短名，你可以使用 kubectl get my, kubectl get myres

- ❾ 将资源添加到所有类别中；运行 kubectl get all 时，会出现这类资源

- ❿ v1alpha1版本是新资源唯一定义的版本。

- ⓫ 将新资源模式定义为一个对象，没有字段

现在，你可以使用 kubectl "apply" 这个资源，使用以下命令：

```bash
$ kubectl apply -f myresource.yaml

customresourcedefinition.apiextensions.k8s.io/myresources.mygroup.example.com created
```

从这开始，你可以使用这个新资源。

例如，你可以使用 HTTP 请求获得整个集群或特定命名空间的资源列表（在运行这些命令之前，你需要从其他终端执行 kubectl proxy）：

```go
$ curl
http://localhost:8001/apis/mygroup.example.com/v1alpha1/myresources
{"apiVersion":"mygroup.example.com/v1alpha1","items":[],"kind":"MyResourceList","metadata":{"continue":"","resourceVersion":"186523407"}}
$ curl
http://localhost:8001/apis/mygroup.example.com/v1alpha1/namespaces/default/myresources
{"apiVersion":"mygroup.example.com/v1alpha1","items":[],"kind":"MyResourceList","metadata":{"continue":"","resourceVersion":"186524840"}}
```

或者，你可以用 kubectl 获得这些列表：

```bash
$ kubectl get myresources
No resources found in default namespace.
```

你可以使用 YAML 格式定义一个新的资源，并使用 kubectl 将其 "apply" 于集群：

```bash
kubectl apply -f - <<EOF
apiVersion: mygroup.example.com/v1alpha1
kind: MyResource
metadata:
  name: myres1
EOF

$ kubectl get myresources
NAME     AGE
myres1   10s
```

额外的打印列

TBD

## 总结

在本章中，你已经看到由 Kubernetes API 提供的资源列表是可以通过在集群中创建 CustomResourceDefinition（CRD）资源来扩展的。你已经看到了CRD资源在 Go 中的结构体，以及如何使用专用的 clientset 来操作 CRD 资源。

之后，你看到了如何用 YAML 编写 CRD，以及如何使用 kubectl 将其部署到集群中。然后，一些字段被添加到 CRD 方案中，以及一些额外的打印列，以便在 `kubectl get` 的输出中显示这些字段。

最后，你已经看到，一旦 CRD 在集群中被创建，你就可以开始使用新的相关种类的资源。
