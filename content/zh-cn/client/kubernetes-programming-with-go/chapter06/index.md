---
title: "[第6章]client-go类库"
linkTitle: "client-go类库"
weight: 60
date: 2021-02-01
description: >
  client-go类库
---

前几章探讨了 Kubernetes API 库和 API Machinery，前者是用于处理Kubernetes API对象的Go结构体的集合，后者则提供了用于处理遵循 Kubernetes API 对象约定的API对象的实用程序。具体来说，你已经看到API Machinery 提供了 Scheme 和 RESTMapper 的抽象。

本章探讨了 Client-go 库，它是一个高级别库，开发者可以使用 Go 语言与 Kubernetes API 进行交互。Client-go 库汇集了 Kubernetes API 和 API Machinery，提供了一个预先配置了 Kubernetes API 对象的 Scheme 和一个用于 Kubernetes API 的 RESTMapper 实现。它还提供了一套客户端，用于以简单的方式对Kubernetes API 的资源执行操作。

要使用这个库，你需要从其中导入包，前缀为 `k8s.io/client-go`。例如，要使用包 kubernetes，让我们使用以下内容：

```go
import (
     "k8s.io/client-go/kubernetes"
)
```

你还需要下载一个 client-go 库的版本。为此，你可以使用 `go get` 命令来获得你要使用的版本：

```bash
go get k8s.io/client-go@v0.24.4
```

Client-go 库的版本与 Kubernetes 的版本是一致的--0.24.4版本对应服务器的1.24.4版本。

Kubernetes 是向后兼容的，所以你可以在较新版本的集群中使用旧版本的Client-go，但你很可能希望得到一个最新的版本，以便能够使用当前的功能，因为只有bug修复被回传到以前的 Client-go 版本，而不是新功能。

## 连接到集群

连接到 Kubernetes API 服务器之前的第一步是让配置连接到它--即服务器的地址、证书、连接参数等。

rest 包提供了一个 `rest.Config` 结构体，它包含了一个应用程序连接到 REST API 服务器所需的所有配置信息。

### 集群内配置

默认情况下，在 Kubernetes Pod 上运行的容器包含连接到API服务器所需的所有信息：

- Pod 使用的 ServiceAccount 提供的令牌和根证书可以在这个目录中找到： 

  `/var/run/secrets/kubernetes.io/serviceaccount/`

- 请注意，可以通过在 Pod 使用的 ServiceAccount 中设置 `automountServiceAccountToken: false`，或直接在 Pod 的 Spec 中设置`automountServiceAccountToken: false` 来禁用这种行为。

- 环境变量，`KUBERNETES_SERVICE_HOST` 和 `KUBERNETES_SERVICE_PORT`，定义在容器环境中，由 kubelet 添加，定义了联系API server 的主机和端口。

当一个应用程序专门在 Pod 的容器内运行时，你可以使用以下函数来创建一个适当的 `rest.Config` 结构体，利用刚才描述的信息：
```go
import "k8s.io/client-go/rest"
func InClusterConfig() (*Config, error)
```

### 集群外的配置

Kubernetes 工具通常依赖于 kubeconfig 文件--即一个包含一个或几个 Kubernetes 集群的连接配置的文件。

你可以通过使用 clientcmd 包中的以下函数之一，根据这个 kubeconfig 文件的内容建立一个 `rest.Config` 结构体。

#### 从内存中的kubeconfig

RESTConfigFromKubeConfig 函数可以用来从作为一个字节数组的 kubeconfig 文件的内容中建立一个 `rest.Config` 结构体：

```go
func RESTConfigFromKubeConfig(
     configBytes []byte,
) (*rest.Config, error)
```

如果 kubeconfig 文件包含几个上下文（context），将使用当前的上下文，而其他的上下文将被忽略。例如，你可以先读取一个 kubeconfig 文件的内容，然后使用以下函数：

```go
import "k8s.io/client-go/tools/clientcmd"
configBytes, err := os.ReadFile(
     "/home/user/.kube/config",
)
if err != nil {
     return err
}
config, err := clientcmd.RESTConfigFromKubeConfig(
     configBytes,
)
if err != nil {
     return err
}
```

#### 从磁盘上的kubeconfig

BuildConfigFromFlags 函数可用于从 API server 的URL中建立 `rest.Config` 结构体，或基于给定路径的 kubeconfig 文件，或两者都是。

```go
func BuildConfigFromFlags(
     masterUrl,
     kubeconfigPath string,
) (*rest.Config, error)
```

下面的代码可以让你得到一个 `rest.Config` 结构体：

```go
import "k8s.io/client-go/tools/clientcmd"
config, err := clientcmd.BuildConfigFromFlags(
     "",
     "/home/user/.kube/config",
)
```

下面的代码从 kubeconfig 获取配置，并重写了 api server 的 URL：

```go
config, err := clientcmd.BuildConfigFromFlags(
     "https://192.168.1.10:6443",
     "/home/user/.kube/config",
)
```

#### 来自个性化的kubeconfig

前面的函数在内部使用一个 `api.Config` 结构体，代表 kubeconfig 文件中的数据（不要与包含 REST HTTP 连接参数的 `rest.Config` 结构体混淆）。

如果你需要操作这个中间数据，你可以使用 BuildConfigFromKubeconfigGetter 函数，接受一个 kubeconfigGetter 函数作为参数，它本身将返回一个 `api.Config` 结构体。

```go
BuildConfigFromKubeconfigGetter(
     masterUrl string,
     kubeconfigGetter KubeconfigGetter,
) (*rest.Config, error)
type KubeconfigGetter
     func() (*api.Config, error)
```

例如，以下代码将用 `clientcmd.Load` 或 `clientcmd.LoadFromFile` 函数从 `kubeconfigGetter` 函数加载 `kubeconfig` 文件：

```go
import (
     "k8s.io/client-go/tools/clientcmd"
     "k8s.io/client-go/tools/clientcmd/api"
)
config, err :=
clientcmd.BuildConfigFromKubeconfigGetter(
     "",
     func() (*api.Config, error) {
          apiConfig, err := clientcmd.LoadFromFile(
               "/home/user/.kube/config",
          )
          if err != nil {
               return nil, nil
          }
          // TODO: manipulate apiConfig
          return apiConfig, nil
     },
)
```

#### 来自多个kubeconfig文件

kubectl 工具默认使用 `$HOME/.kube/config` kubeconfig`文件，你可以使用 KUBECONFIG 环境变量指定另一个 kubeconfig 文件路径。

不仅如此，你还可以在这个环境变量中指定一个 kubeconfig 文件路径的列表，这些 kubeconfig 文件在被使用之前将被合并成一个而已。你可以用这个函数获得同样的行为： NewNonInteractiveDeferredLoadingClientConfig。

```go
func NewNonInteractiveDeferredLoadingClientConfig(
     loader ClientConfigLoader,
     overrides *ConfigOverrides,
) ClientConfig
```

clientcmd.ClientConfigLoadingRules 类型实现了 ClientConfigLoader 接口，你可以用下面的函数得到这个类型的值：

```go
func NewDefaultClientConfigLoadingRules()
 *ClientConfigLoadingRules
```

这个函数将获得 KUBECONFIG 环境变量的值，如果它存在的话，以获得要合并的 kubeconfig 文件的列表，或者将退回到使用位于 `$HOME/.kube/config` 的默认kubeconfig文件。

使用下面的代码来创建 `rest.Config` 结构体，你的程序将具有与 kubectl 相同的行为，如前所述：

```go
import (
     "k8s.io/client-go/tools/clientcmd"
)
config, err :=
clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
     clientcmd.NewDefaultClientConfigLoadingRules(),
     nil,
).ClientConfig()
```

#### 用CLI标志重写kubeconfig

已经表明，这个函数的第二个参数，NewNonInteractiveDeferredLoadingClientConfig，是一个 ConfigOverrides 结构。这个结构包含覆盖合并 kubeconfig 文件结果的一些字段的值。

你可以自己在这个结构体中设置特定的值，或者，如果你正在使用 `spf13/pflag` 库（即 `github.com/spf13/pflag`）创建一个CLI，你可以使用下面的代码为你的CLI自动声明默认标志，并将它们绑定到 ConfigOverrides 结构体：

```go
import (
     "github.com/spf13/pflag"
     "k8s.io/client-go/tools/clientcmd"
)
var (
     flags pflag.FlagSet
     overrides clientcmd.ConfigOverrides
     of = clientcmd.RecommendedConfigOverrideFlags("")
)
clientcmd.BindOverrideFlags(&overrides, &flags, of)
flags.Parse(os.Args[1:])
config, err :=
clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
     clientcmd.NewDefaultClientConfigLoadingRules(),
     &overrides,
).ClientConfig()
```

注意，你可以在调用函数 RecommendedConfigOverrideFlags 时为添加的标志声明一个前缀。

## 获取 ClientSet

Kubernetes 包提供了创建 `kubernetes.Clientset` 类型的 ClientSet 的函数。

- `func NewForConfig(c *rest.Config) (*Clientset, error) `- NewForConfig 函数返回 ClientSet，使用提供的 `rest.Config` 与上一节中看到的方法之一构建。

- `func NewForConfigOrDie(c *rest.Config) *Clientset` - 这个函数和前一个函数一样，但是在出错的情况下会 panic ，而不是返回错误。这个函数可以与硬编码的配置一起使用，你会想要断言其有效性。

- `NewForConfigAndClient(
     c *rest.Config、
     httpClient *http.Client、
     ) (*Clientset, error)`
     
     这个 NewForConfigAndClient 函数使用提供的 `rest.Config` 和提供的 `http.Client` 返回一个 Clientset。

之前的函数 NewForConfig 使用的是用函数 rest.HTTPClientFor 构建的默认 HTTP 客户端。如果你想在构建客户集之前个性化HTTP客户端，你可以使用这个函数来代替。

### 使用 ClientSet

kubernetes.Clientset 类型实现了 kubernetes.Interface 接口，定义如下：

```go
type Interface interface {
     Discovery() discovery.DiscoveryInterface
     [...]
     AppsV1()          appsv1.AppsV1Interface
     AppsV1beta1()     appsv1beta1.AppsV1beta1Interface
     AppsV1beta2()     appsv1beta2.AppsV1beta2Interface
     [...]
     CoreV1()           corev1.CoreV1Interface
     [...]
}
```

第一个方法 Discovery() 提供了对一个接口的访问，该接口提供了发现集群中可用的分组、版本和资源的方法，以及资源的首选版本。这个接口还提供对服务器版本和 OpenAPI v2 和v3定义的访问。这将在发现客户端部分详细研究。

除了 Discovery() 方法外，kubernetes.Interface 由一系列方法组成，Kubernetes API 定义的每个 Group/Version 都有一个。当你看到这个接口的定义时，就可以理解 ClientSet 是一组客户端，每个客户端都专门用于自己的分组/版本。

每个方法都会返回一个值，该值实现了该分组/版本的特定接口。例如，`kubernetes.Interface的CoreV1()` 方法返回一个值，实现 `corev1.CoreV1Interface` 接口，定义如下：

```go
type CoreV1Interface interface {
     RESTClient() rest.Interface
     ComponentStatusesGetter
     ConfigMapsGetter
     EndpointsGetter
     [...]
}
```

这个 CoreV1Interface 接口的第一个方法是 `RESTClient() rest.Interface`，它是一个用来获取特定 Group/Version 的 REST 客户端的方法。这个低级客户端将被 Group/Version 客户端内部使用，你可以使用这个 REST 客户端来构建这个 CoreV1Interface 接口的其他方法所不能原生提供的请求。

由 REST 客户端实现的接口 `rest.Interface` 定义如下：

```go
type Interface interface {
     GetRateLimiter()            flowcontrol.RateLimiter
     Verb(verb string)           *Request
     Post()                      *Request
     Put()                       *Request
     Patch(pt types.PatchType)   *Request
     Get()                       *Request
     Delete()                    *Request
     APIVersion()                schema.GroupVersion
}
```

正如你所看到的，这个接口提供了一系列的方法--Verb、Post、Put、Patch、Get 和 Delete--它们返回一个带有特定 HTTP Verb 的 Request 对象。在 "如何使用这些Request对象来完成操作 "一节中，将进一步研究这个问题。

CoreV1Interface 中的其他方法被用来获取分组/版本中每个资源的特定方法。例如，ConfigMapsGetter 嵌入式接口的定义如下：

```go
type ConfigMapsGetter interface {
     ConfigMaps(namespace string) ConfigMapInterface
}
```

然后，接口 ConfigMapInterface由方法 ConfigMaps 返回，定义如下：

```go
type ConfigMapInterface interface {
     Create(
          ctx context.Context,
          configMap *v1.ConfigMap,
          opts metav1.CreateOptions,
     ) (*v1.ConfigMap, error)
     Update(
          ctx context.Context,
          configMap *v1.ConfigMap,
          opts metav1.UpdateOptions,
     ) (*v1.ConfigMap, error)
     Delete(
          ctx context.Context,
          name string,
          opts metav1.DeleteOptions,
     ) error
     [...]
}
```

你可以看到，这个接口提供了一系列的方法，每个 Kubernetes API 动词都有一个。

每个与操作相关的方法都需要一个 Option 结构体作为参数，以操作的名称命名： CreateOptions, UpdateOptions, DeleteOptions，等等。这些结构体和相关的常量都定义在这个包中：`k8s.io/apimachinery/pkg/apis/meta/v1`。

最后，要对一个 Group-Version 的资源进行操作，你可以按照这个模式对 namespaced 资源进行连锁调用，其中 namespace 可以是空字符串，以表示一个集群范围的操作：

```go
clientset.
     GroupVersion().
     NamespacedResource(namespace).
     Operation(ctx, options)
```

那么，以下是不带命名空间的资源的模式：

```go
clientset.
    GroupVersion().
     NonNamespacedResource().
     Operation(ctx, options)
```

例如，使用下面的方法来列出命名空间 project1 中 core/v1 分组/版本的 Pods：

```go
podList, err := clientset.
     CoreV1().
     Pods("project1").
     List(ctx, metav1.ListOptions{})
```

要获得所有命名空间的 pod 列表，你需要指定一个空的命名空间名称：

```go
podList, err := clientset.
     CoreV1().
     Pods("").
     List(ctx, metav1.ListOptions{})
```

要获得节点的列表（这些节点是没有命名的资源），请使用这个：

```go
nodesList, err := clientset.
     CoreV1().
     Nodes().
     List(ctx, metav1.ListOptions{})
```

下面的章节详细描述了使用 Pod 资源的各种操作。在处理非命名空间的资源时，你可以通过删除命名空间参数来应用同样的例子。

## 检查请求

如果你想知道在调用 client-go 方法时，哪些 HTTP 请求被执行，你可以为你的程序启用日志记录。Client-go库使用klog库（https://github.com/kubernetes/klog），你可以用以下代码为你的命令启用日志标志：

```go
import (
     "flag"
     "k8s.io/klog/v2"
)
func main() {
     klog.InitFlags(nil)
     flag.Parse()
     [...]
}
```

现在，你可以用标志 `-v <level>` 来运行你的程序--例如，`-v 6` 来获得每个请求的URL调用。你可以在表2-1中找到更多关于定义的日志级别的细节。

## 创建资源

要在集群中创建一个新的资源，你首先需要使用专用的 Kind 结构体在内存中声明这个资源，然后为你要创建的资源使用创建方法。例如，使用下面的方法，在 project1 命名空间中创建一个名为 nginx-pod 的 Pod：

```go
wantedPod := corev1.Pod{
     Spec: corev1.PodSpec{
          Containers: []corev1.Container{
               {
                    Name:  "nginx",
                    Image: "nginx",
               },
          },
     },
}
wantedPod.SetName("nginx-pod")
createdPod, err := clientset.
     CoreV1().
     Pods("project1").
     Create(ctx, &wantedPod, v1.CreateOptions{})
```

在创建资源时，用于声明 CreateOptions 结构体的各种选项是：

- **DryRun** - 这表明API服务器端的哪些操作应该被执行。唯一可用的值是 metav1.DryRunAll，表示执行所有的操作，除了将资源持久化到存储。

  使用这个选项，你可以得到命令的结果，即在集群中创建的确切对象，而不是真正的创建，并检查在这个创建过程中是否会发生错误。

- **FieldManager** - 这表示这个操作的字段管理器的名称。这个信息将被用于未来的服务器端应用操作。

- **FieldValidation** - 这表明当结构中出现重复或未知字段时，服务器应该如何反应。以下是可能的值：

  - `metav1.FieldValidationIgnore` 忽略所有重复的或未知的字段
  - `metav1.FieldValidationWarn` 当出现重复或未知字段时发出警告。
  - `metav1.FieldValidationStrict` 当重复字段或未知字段出现时失败。

​	请注意，使用这个方法，你将无法定义重复或未知的字段，因为你是使用结构体来定义对象。

如果出现错误，你可以用 `k8s.io/apimachinery/pkg/api/errors` 包中定义的函数测试其类型。所有可能的错误都在 "错误和状态 "一节中定义，这里是针对 **Create** 操作的可能错误：

- **IsAlreadyExists** - 这个函数指示请求是否因为集群中已经存在同名的资源而失败：

  ```go
  if errors.IsAlreadyExists(err) {
       // ...
  }
  ```

- **IsNotFound** - 这个函数表示你在请求中指定的命名空间是否不存在。

- **IsInvalid** - 这个函数表示传入结构体的数据是否无效。

## 获取资源的信息

要获得集群中某一特定资源的信息，可以使用 Get 方法，从该资源中获取信息。例如，要获得 project1 命名空间中名为 nginx-pod 的 pod 的信息：

```go
pod, err := clientset.
     CoreV1().
     Pods("project1").
     Get(ctx, "nginx-pod", metav1.GetOptions{})
```


在获取资源的信息时，声明到 GetOptions 结构体中的各种选项是：

- **ResourceVersion** - 请求一个不早于指定版本的资源版本。

- 如果 ResourceVersion 是 "0"，表示返回该资源的任何版本。你通常会收到资源的最新版本，但这并不保证；由于分区或陈旧的缓存，在高可用性集群上可能会收到较旧的版本。

- 如果没有设置该选项，你将保证收到资源的最新版本。

获取操作特有的可能错误是：

- **IsNotFound** - 这个函数表示你在请求中指定的命名空间不存在，或者指定名称的资源不存在。

## 获取资源列表

要获得集群中的资源列表，你可以为你想要列出的资源使用 List 方法。例如，使用下面的方法来列出 project1 命名空间中的 pods：

```go
podList, err := clientset.
     CoreV1().
     Pods("project1").
     List(ctx, metav1.ListOptions{})
```

或者，要获得所有命名空间中的 pod 列表，使用：

```go
podList, err := clientset.
     CoreV1().
     Pods("").
     List(ctx, metav1.ListOptions{})
```

在列出资源时，需要向 ListOptions 结构体声明的各种选项如下：

- LabelSelector, FieldSelector - 这是用来按标签或按字段过滤列表的。这些选项在 "过滤列表结果 "部分有详细介绍。

- Watch, AllowWatchBookmarks - 这是用来运行 watch 操作。这些选项在 "观察资源 "部分有详细介绍。

- ResourceVersion, ResourceVersionMatch - 这表明你想获得哪个版本的资源列表。

  请注意，当收到 List 操作的响应时，List元素本身的 ResourceVersion 值，以及List中每个元素的 ResourceVersion 值都会被指出。选项中指出的资源版本指的是 List 的资源版本。

- 对于没有分页的列表操作（你可以参考 "分页结果 "和 "观察资源" 部分，了解这些选项在其他情况下的行为）：

  - 当 ResourceVersionMatch 没有被设置时，其行为与Get操作相同：
  - ResourceVersion 表示你应该返回一个不比指定版本早的列表。

  - 如果 ResourceVersion是 "0"，这表明有必要返回列表的任何版本。一般来说，你会收到它的最新版本，但这并不保证；在高可用性的集群上，由于分区或陈旧的缓存，收到旧版本的情况可能发生。

  - 如果不设置该选项，你就能保证收到列表的最新版本。

  - 当 ResourceVersionMatch 被设置为 `metav1.ResourceVersionMatchExact` 时，ResourceVersion 值表示你想获得的列表的确切版本。

  - 将 ResourceVersion 设置为 "0"，或者不定义它，是无效的。

  - 当ResourceVersionMatch设置为 `metav1.ResourceVersionMatchNotOlderThan` 时，ResourceVersion 表示你将获得一个不比指定版本老的列表。

  - 如果ResourceVersion是 "0"，这表示将返回列表的任何版本。你通常会收到列表的最新版本，但这并不保证；在高可用性集群中，由于分区或陈旧的缓存，收到旧版本的情况可能发生。

  - 不定义ResourceVersion是无效的。

- TimeoutSeconds - 这将请求的持续时间限制在指定的秒数内。

- Limit, Continue - 这用于对列表的结果进行分页。这些选项在第二章的 "分页结果 "部分有详细说明。

以下是 List 操作特有的可能错误：

- **IsResourceExpired** - 这个函数表示指定的 **ResourceVersion** 与 ResourceVersionMatch，设置为 `metav1.ResourceVersionMatchExact`，已经过期。

注意，如果你为 List 操作指定一个不存在的命名空间，你将不会收到 NotFound 错误。

## 筛选列表的结果

正如第2章 "过滤列表结果"一节所述，可以用标签选择器和字段选择器来过滤列表操作的结果。本节展示了如何使用API Machinery 库的字段和标签包来创建一个适用于 LabelSelector 和 FieldSelector 选项的字符串。

### 使用标签包设置LabelSelector

下面是使用API Machinery 库的 labels 包的必要导入信息。

```go
import (
     "k8s.io/apimachinery/pkg/labels"
)
```

该包提供了几种建立和验证 LabelsSelector 字符串的方法：使用 Requirements，解析 labelSelector 字符串，或使用一组键值对。

#### 使用 Requirements

你首先需要使用下面的代码创建一个 `label.Selector `对象：

```go
labelsSelector := labels.NewSelector()
```

然后，你可以使用 `labs.NewRequirement` 函数创建 `Requirement` 对象：

```go
func NewRequirement(
     key string,
     op selection.Operator,
     vals []string,
     opts ...field.PathOption,
) (*Requirement, error)
```

op 的可能值的常量在 selection 包中定义（即 `k8s.io/apimachinery/pkg/selection`）。vals 字符串数组中的值的数量取决于操作：

- selection.In; selection.NotIn - 附加到 key 的值必须等于(In)/必须不等于(NotIn)vals定义的值中的一个。

  vals必须不是空的。

- selection.Equals; selection.DoubleEquals; selection.NotEquals - 附加到key的值必须等于（Equals, DoubleEquals）或者不等于（NotEquals）vals中定义的值。

  vals必须包含一个单一的值。

- selection.Exists; selection.DoesNotExist - 键必须被定义（Exists）或必须不被定义（DoesNotExist）。

  vals必须是空的。

- selection.Gt; selection.Lt - 附加在键上的值必须大于（Gt）或小于（Lt）vals中定义的值。

  vals必须包含一个单一的值，代表一个整数。

例如，为了要求键 mykey 的值等于 value1，你可以声明 Requirement：

```go
req1, err := labels.NewRequirement(
     "mykey",
     selection.Equals,
     []string{"value1"},
)
```

在定义 Requirement 后，你可以使用选择器上的 Add 方法将需求添加到选择器中：

```go
labelsSelector = labelsSelector.Add(*req1, *req2)
```

最后，你可以用以下方法获得 LabelSelector 选项所要传递的字符串：

```go
s := labelsSelector.String()
```

#### 解析 LabelSelector 字符串

如果你已经有一个描述标签选择器的字符串，你可以用 Parse 函数检查其有效性。Parse 函数将验证该字符串并返回一个 LabelSelector 对象。您可以在这个 LabelSelector 对象上使用 String 方法来获得由 Parse 函数验证的字符串。

作为一个例子，下面的代码将解析、验证并返回标签选择器的典型形式，"mykey = value1, count < 5"：

```go
selector, err := labels.Parse(
     "mykey = value1, count < 5",
)
if err != nil {
     return err
}
s := selector.String()
// s = "mykey=value1,count<5"
```

#### 使用键值对的集合

当你只想使用等价(Equal)操作时，可以使用 ValidatedSelectorFromSet 这个函数，以满足一个或几个要求：

```go
func ValidatedSelectorFromSet(
     ls Set
) (Selector, error)
```

在这种情况下，Set 将定义你想检查的键值对的集合，以确保等价(Equal)。

作为一个例子，下面的代码将声明一个标签选择器，要求键 key1，等于value1，键 key2，等于value2：

```go
set := labels.Set{
     "key1": "value1",
     "key2": "value2",
}
selector, err = labels.ValidatedSelectorFromSet(set)
s = selector.String()
// s = "key1=value1,key2=value2"
```

### 使用 Fields 包设置 Fieldselector

下面是用于从 API Machinery 中导入 Fields 包的必要代码。

```go
import (
     "k8s.io/apimachinery/pkg/fields"
)
```

该包提供了几种建立和验证 FieldSelector 字符串的方法：组装术语选择器(term selector)，解析 fieldSelector 字符串，或使用一组键值对。

#### 组装术语选择器

你可以用函数 OneTermEqualSelector 和 OneTermNotEqualSelector 创建一个术语选择器，然后用函数 AndSelectors 组装选择器来建立一个完整的字段选择器。

```go
func OneTermEqualSelector(
     k, v string,
) Selector
func OneTermNotEqualSelector(
     k, v string,
) Selector
func AndSelectors(
     selectors ...Selector,
) Selector
```

例如，这段代码建立了一个字段选择器，在字段 `status.Phase`上有一个 Equal 条件，在字段 `spec.restartPolicy` 上有一个 NotEqual 条件：

```go
fselector = fields.AndSelectors(
     fields.OneTermEqualSelector(
          "status.Phase",
          "Running",
     ),
     fields.OneTermNotEqualSelector(
          "spec.restartPolicy",
          "Always",
     ),
)
fs = fselector.String()
```

#### 解析字段选择器字符串

如果你已经有一个描述字段选择器的字符串，你可以用 ParseSelector 或 ParseSelectorOrDie 函数检查其有效性。ParseSelector 函数将验证该字符串并返回一个 fields.Selector 对象。你可以在这个 fields.Selector 对象上使用 String 方法来获得由 ParseSelector 函数验证的字符串。

作为一个例子，这段代码将解析、验证并返回字段选择器的典型形式 "status.Phase = Running, spec.restartPolicy != Always"：

```go
selector, err := fields.ParseSelector(
     "status.Phase=Running, spec.restartPolicy!=Always",
)
if err != nil {
     return err
}
s := selector.String()
// s = "spec.restartPolicy!=Always,status.Phase=Running"
```

#### 使用键值对的集合

当你想对一个或几个单一的选择器只使用等价操作时，可以使用SelectorFromSet这个函数。

```go
func SelectorFromSet(ls Set) Selector
```

在这种情况下，Set将定义你要检查的键值对的集合，以确保平等。

作为一个例子，下面的代码将声明一个字段选择器，要求键 key1 等于 value1，键 key2，等于value2：

```go
set := fields.Set{
     "field1": "value1",
     "field2": "value2",
}
selector = fields.SelectorFromSet(set)
s = selector.String()
// s = "key1=value1,key2=value2"
```

## 删除资源

要从集群中删除资源，可以对你要删除的资源使用删除方法。例如，要从 project1 命名空间中删除一个名为 nginx-pod 的 Pod，可以使用：

```go
err = clientset.
     CoreV1().
     Pods("project1").
     Delete(ctx, "nginx-pod", metav1.DeleteOptions{})
```

请注意，不保证操作终止时资源被删除。删除操作不会有效地删除资源，但会标记资源被删除（通过设置字段 `.metadata.deletionTimestamp`），并且删除将以异步方式发生。

DryRun - 这表明API服务器端的哪些操作应该被执行。唯一可用的值是 `metav1.DryRunAll`，表示要执行所有的操作，除了（将资源持久化到存储的操作）。使用这个选项，你可以得到命令的结果，而不是真的删除资源，并检查在这个删除过程中是否会发生错误。

GracePeriodSeconds - 这个值只在删除 pod 时有用。它表示在删除 pod 之前的持续时间，单位是秒。

该值必须是一个指向非负整数的指针。值为零表示立即删除。如果这个值为 nil，将使用 pod 的默认宽限期，如 pod spec 中的TerminationGracePeriodSeconds 字段所示。

你可以使用 `metav1.NewDeleteOptions` 函数来创建一个定义了 GracePeriodSeconds的DeleteOptions 的结构体：

```go
err = clientset.
     CoreV1().
     Pods("project1").
     Delete(ctx,
          "nginx-pod",
          *metav1.NewDeleteOptions(5),
     )
```

Preconditions（前提条件） - 当你删除一个对象时，你可能想确保删除预期的对象。前提条件字段让你指出你期望删除的资源，可以通过以下方式：

- 指明UID，所以如果预期的资源被删除，而另一个资源被创建了相同的名字，那么删除将失败，产生一个冲突错误。你可以使用`metav1.NewPreconditionDeleteOptions` 函数来创建一个 DeleteOptions 结构体，并设置 Preconditions 的UID：

  ```go
  uid := createdPod.GetUID()
  err = clientset.
       CoreV1().
       Pods("project1").
       Delete(ctx,
            "nginx-pod",
            *metav1.NewPreconditionDeleteOptions(
                 string(uid),
            ),
       )
  if errors.IsConflict(err) {
     [...]
  }
  ```

- 指定 ResourceVersion，所以如果在此期间资源被更新，删除将失败，并出现 Conflict 错误。你可以使用 `metav1.NewRVDeletionPrecondition` 函数来创建一个 DeleteOptions 结构体，并设置前提条件的 ResourceVersion：

  ```go
  rv := createdPod.GetResourceVersion()
  err = clientset.
       CoreV1().
       Pods("project1").
       Delete(ctx,
            "nginx-pod",
            *metav1.NewRVDeletionPrecondition(
                 rv,
            ),
       )
  if errors.IsConflict(err) {
     [...]
  }
  ```

OrphanDependents - 这个字段已被废弃，转而使用 PropagationPolicy。

PropagationPolicy - 这表明是否以及如何进行垃圾收集。参见第三章的 "OwnerReferences" 部分。可接受的值是：

- `metav1.DeletePropagationOrphan` - 向Kubernetes API表示将你正在删除的资源所拥有的资源变成孤儿，这样它们就不会被垃圾收集器删除。

- `metav1.DeletePropagationBackground` - 指示Kubernetes API在所有者资源被标记为删除后立即返回删除操作，而不是等待拥有的资源被垃圾收集器删除。

- `metav1.DeletePropagationForeground `- 指示 Kubernetes API 在所有者和 BlockOwnerDeletion 设置为 true 的自有资源被删除后，从 Delete 操作中返回。Kubernetes API将不会等待其他拥有的资源被删除。

以下是删除操作特有的可能错误：

- IsNotFound - 这个函数表示你在请求中指定的资源或命名空间不存在。
- IsConflict - 这个函数表示请求失败，因为一个前提条件没有被遵守（UID或ResourceVersion）。

## 删除资源集合

要从集群中删除资源集合，你可以为你要删除的资源使用 DeleteCollection 方法。例如，要从 project1 命名空间中删除 Pod 的集合：

```go
err = clientset.
     CoreV1().
     Pods("project1").
     DeleteCollection(
          ctx,
          metav1.DeleteOptions{},
          metav1.ListOptions{},
     )
```

必须向该函数提供两组选项：

- DeleteOptions，表示对每个对象进行删除操作的选项，如 "删除资源" 部分所述。
- ListOptions，细化要删除的资源集合，如 "获取资源列表" 部分所述。

## 更新资源

要更新集群中的资源，你可以为你要更新的资源使用更新方法。例如，使用以下方法来更新 project1 命名空间中的 deployment：

```go
updatedDep, err := clientset.
     AppsV1().
     Deployments("project1").
     Update(
          ctx,
          myDep,
          metav1.UpdateOptions{},
     )
```

当更新资源时，要声明到 UpdateOptions 结构体中的各种选项，与 "创建资源"一节中描述的 CreateOptions 中的选项相同。

更新操作可能出现的特定错误是：

- IsInvalid - 这个函数表示传递到结构中的数据是无效的。
- IsConflict（冲突）--该函数表示纳入结构中的 ResourceVersion（这里是 myDep）比集群中的版本要早。更多信息请参见第2章的 "更新资源管理冲突" 部分。

## 使用 Strategic Merge Patch 来更新资源

在第二章 "使用Strategic Merge Patch（战略合并补丁）更新资源 "一节中，你已经看到了用战略合并补丁对资源进行修补的过程。总而言之，你需要：

- 使用 "Patch" 操作：
- 为 content-type 头指定特定的值

- 在正文中传递你想修改的唯一字段


使用 Client-go 库，你可以对你要修补的资源使用 Patch 方法。

```go
Patch(
     ctx context.Context,
     name string,
     pt types.PatchType,
     data []byte,
     opts metav1.PatchOptions,
     subresources ...string,
     ) (result *v1.Deployment, err error)
```

PatchType 表明你是想使用 StrategicMerge patch（`types.StrategicMergePatchType`）还是合并补丁（`types.MergePatchType`）。这些常数在`k8s.io/apimachinery/pkg/types` 包中定义。

data 字段包含你想应用到资源的补丁。你可以直接写这个补丁数据，就像在第二章中做的那样，或者你可以使用 controller-runtime 的以下功能来帮助你建立这个补丁。这个库将在第10章中进行更深入的探讨。

```go
import "sigs.k8s.io/controller-runtime/pkg/client"
func StrategicMergeFrom(
     obj Object,
     opts ...MergeFromOption,
) Patch
```

StrategicMergeFrom 函数的第一个参数接受一个 Object 类型，代表任何 Kubernetes 对象。你将通过这个参数传递你想要修补的对象，在任何改变之前。

然后，该函数接受一系列的选项。目前唯一接受的选项是 `client.MergeFromWithOptimisticLock{}` 值。这个值要求库将 ResourceVersion 添加到补丁数据中，因此服务器将能够检查你要更新的资源版本是否是最后一个。

在你使用 StrategicMergeFrom 函数创建了 Patch 对象后，你可以创建你想打补丁的对象的深度拷贝，然后修改它。然后，当你完成更新对象后，你可以用 Patch 对象的专用数据方法建立补丁的数据。

作为例子，要为 Deploymen 建立补丁数据，包含乐观锁的资源版本（ResourceVersion），你可以使用下面的代码（createdDep 是一个反映在集群中创建的Deployment 的结构体）：

```go
patch := client.StrategicMergeFrom(
     createdDep,
     pkgclient.MergeFromWithOptimisticLock{},
)
updatedDep := createdDep.DeepCopy()
updatedDep.Spec.Replicas = pointer.Int32(2)
patchData, err := patch.Data(updatedDep)
// patchData = []byte(`{
//   "metadata":{"resourceVersion":"4807923"},
//   "spec":{"replicas":2}
// }`)
patchedDep, err := clientset.
     AppsV1().Deployments("project1").Patch(
          ctx,
          "dep1",
          patch.Type(),
          patchData,
          metav1.PatchOptions{},
     )
```

注意 MergeFrom 和 MergeFromWithOptions 函数也是可用的，如果你喜欢执行一个合并补丁。

Patch 对象的 Type 方法可以用来检索补丁类型，而不是使用类型包中的常量。你可以在调用补丁操作时传递 PatchOptions。可能的选项有：

- DryRun - 这表明API服务器端的哪些操作应该被执行。唯一可用的值是 `metav1.DryRunAll`，表示执行所有操作，除了将资源持久化到存储。

- Force - 这个选项只能用于 Apply patch 请求，在处理 StrategicMergePatch 或 MergePatch 请求时必须取消设置。

- FieldManager - 这表示该操作的字段管理器的名称。这个信息将被用于未来的服务器端 Apply 操作。这个选项对于 StrategicMergePatch 或 MergePatch 请求是可选的。

- FieldValidation - 这表明当结构体中出现重复或未知字段时，服务器应该如何反应。以下是可能的值：
  - metav1.FieldValidationIgnore - 忽略所有重复的或未知的字段
  - metav1.FieldValidationWarn - 当出现重复或未知字段时发出警告
  - metav1.FieldValidationStrict - 当出现重复字段或未知字段时失败。

注意，Patch 操作接受 subresources 参数。这个参数可以用来修补应用补丁方法的资源的子资源。例如，要修补一个 Deployment 的 Status，你可以使用subresources 参数的值 "status"。

MergePatch 操作特有的可能的错误是：

- IsInvalid - 这个函数指示作为补丁传递的数据是否无效。
- IsConflict - 这个函数表示并入补丁的资源版本（如果你在构建补丁数据时使用优化锁）是否比集群中的版本更早。更多信息可在第二章 "更新资源管理冲突 "部分找到。

## 用补丁在服务器端应用资源

第二章的 "在服务器端应用资源" 部分描述了服务器端应用补丁是如何工作的。总而言之，我们需要：

- 使用 "补丁 "操作
- 为 content-type 头指定一个特定的值

- 在正文中传递你想修改的唯一字段

- 提供一个 fieldManager 名称


使用 Client-go 库，你可以对你要修补的资源使用 Patch 方法。注意，你也可以使用 Apply 方法；见下一节，"使用Apply在服务器端应用资源"。

```go
Patch(
     ctx context.Context,
     name string,
     pt types.PatchType,
     data []byte,
     opts metav1.PatchOptions,
     subresources ...string,
) (result *v1.Deployment, err error)
```

PatchType 表示补丁的类型，这里是 `type.ApplyPatchType`，定义于 `k8s.io/apimachinery/pkg/types` 包。

data 字段包含你想应用到资源的补丁。你可以使用 client.Apply 值来构建这个数据。这个值实现了 client.Patch 接口，提供了Type和Data方法。

注意，你需要在你想打补丁的资源结构体中设置 APIVersion 和 Kind 字段。还要注意，这个 Apply 操作也可以用来创建资源。

补丁操作接受 subresources 参数。这个参数可以用来修补应用Patch方法的资源的子资源。例如，要修补 Deployment 的 Status，你可以使用 subresources 参数的值 "status"。

```go
import "sigs.k8s.io/controller-runtime/pkg/client"
wantedDep := appsv1.Deployment{
     Spec: appsv1.DeploymentSpec{
          Replicas: pointer.Int32(1),
     [...]
}
wantedDep.SetName("dep1")
wantedDep.APIVersion, wantedDep.Kind =
     appsv1.SchemeGroupVersion.
          WithKind("Deployment").
          ToAPIVersionAndKind()
patch := client.Apply
patchData, err := patch.Data(&wantedDep)
patchedDep, err := clientset.
     AppsV1().Deployments("project1").Patch(
          ctx,
          "dep1",
          patch.Type(),
          patchData,
          metav1.PatchOptions{
          FieldManager: "my-program",
     },
     )
```

你可以在调用 Patch 操作时传递 PatchOptions。以下是可能的选项：

- DryRun - 这表明API服务器端的哪些操作应该被执行。唯一可用的值是 `metav1.DryRunAll`，表示执行所有操作，除了将资源持久化到存储。
- Force - 这个选项表示强制应用请求。这意味着这个请求的字段管理器将获得其他字段管理器所拥有的冲突字段。
- FieldManager - 这表示该操作的字段管理器的名称。这个信息将被用于未来的服务器端 Apply 操作。
- FieldValidation - 这表明当结构体中出现重复或未知字段时，服务器应该如何反应。以下是可能的值：
  - metav1.FieldValidationIgnore - 忽略所有重复的或未知的字段
  - metav1.FieldValidationWarn - 当出现重复或未知字段时发出警告
  - metav1.FieldValidationStrict - 当出现重复字段或未知字段时失败。

**ApplyPatch** 操作特有的可能的错误是：

- IsInvalid - 这个函数指示作为补丁传递的数据是否无效。
- IsConflict - 这个函数表示被补丁修改的一些字段是否有冲突，因为它们被另一个字段管理器拥有。为了解决这个冲突，你可以使用强制选项，这样这些字段就会被这个操作的字段管理器获得。

### Server-side Apply Using Apply Configurations

TBD

### Building an ApplyConfiguration from Scratch

TBD

### Building an ApplyConfiguration from an Existing Resource

## 监视资源

第二章的 "监视资源 "部分描述了 Kubernetes API 如何观察资源。使用 Client-go 库，你可以对你想观察的资源使用Watch方法。

```go
Watch(
     ctx context.Context,
     opts metav1.ListOptions,
) (watch.Interface, error)
```

这个 Watch 方法返回一个实现了 watch.Interface 接口的对象，并提供以下方法：

```go
import "k8s.io/apimachinery/pkg/watch"
type Interface interface {
     ResultChan() <-chan Event
     Stop()
}
```

ResultChan 方法返回一个 Go通道（只能读取），你将能够接收所有的事件。

Stop 方法将停止 Watch 操作并关闭使用 ResultChan 接收的通道。

使用通道接收的 watch.Event 对象的定义如下：

```go
type Event struct {
     Type EventType
     Object runtime.Object
}
```

Type 字段可以得到第2章表2-2中早先描述的值，你可以在 watch 包中找到这些不同值的常量：watch.Added, watch.Modified, watch.Deleted, watch.Bookmark, 和 watch.Error。

Object 字段实现了 runtime.Object 接口，它的具体类型可以根据 Type 的值而不同。

对于除 Error 以外的类型，Object 的具体类型将是你正在监视的资源的类型（例如，如果你正在监视 **Deployment**，则是 **Deployment** 类型）。

对于 Error 类型，具体类型通常是 `metav1.Status`，但它可以是任何其他类型，取决于你正在观察的资源。作为一个例子，这里有一段观察部署的代码：

```go
import "k8s.io/apimachinery/pkg/watch"
watcher, err := clientset.AppsV1().
     Deployments("project1").
     Watch(
          ctx,
          metav1.ListOptions{},
     )
if err != nil {
     return err
}
for ev := range watcher.ResultChan() {
     switch v := ev.Object.(type) {
     case *appsv1.Deployment:
          fmt.Printf("%s %s\n", ev.Type, v.GetName())
     case *metav1.Status:
          fmt.Printf("%s\n", v.Status)
          watcher.Stop()
     }
}
```

在观察资源时，需要在 ListOptions 结构体中声明的各种选项如下：

- LabelSelector, FieldSelector - 这是用来过滤按标签或按字段观察的元素。这些选项在 "过滤列表结果" 部分有详细说明。

- Watch, AllowWatchBookmarks - Watch 选项表示正在运行一个观察操作。这个选项是在执行 Watch 方法时自动设置的；你不需要明确地设置它。

- AllowWatchBookmarks 选项要求服务器定期返回 Bookmarks。书签的使用在第二章的 "允许书签有效地重启观察请求" 一节中有所描述。

- ResourceVersion, ResourceVersionMatch - 这表明你想在资源列表的哪个版本上开始观察操作。

  请注意，当收到 List 操作的响应时，会为列表元素本身指出一个ResourceVersion值，以及列表中每个元素的ResourceVersion值。选项中指出的ResourceVersion是指列表的ResourceVersion。

- ResourceVersionMatch 选项不用于观察操作。对于观察操作，请执行以下操作：

  - 当 ResourceVersion 没有设置时，API将从最近的资源列表开始观察操作。该通道首先接收 ADDED 事件以声明资源的初始状态，然后在集群上发生变化时接收其他事件。

  - 当 ResourceVersion 被设置为一个特定的版本时，API将从资源列表的指定版本开始观察操作。该通道将不接收声明资源初始状态的 ADDED 事件，而只接收该版本之后集群上发生变化时的事件（可以是指定版本和你运行Watch操作之间发生的事件）。

  - 一个用例是观察一个特定资源的删除情况。为此，你可以

    1. 列出资源，包括你想删除的那个，并保存收到的列表的ResourceVersion。

    2. 对资源执行删除操作（删除是异步的，当操作终止时，资源可能不会被删除）。

    3. 通过指定在步骤1中收到的ResourceVersion，启动一个Watch操作。即使删除发生在步骤2和步骤3之间，你也能保证收到DELETED事件。

- 当 ResourceVersion 被设置为 "0 "时，API将在任何资源列表中启动Watch操作。该通道首先接收ADDED事件，以声明资源的初始状态，然后在这个初始状态之后集群上发生变化时接收其他事件。

​		在使用这种语义时，你必须特别小心，因为Watch操作通常会从最新的版本开始；但是，从较早的版本开始也是可能的。

- TimeoutSeconds - 这将请求的持续时间限制在指定的秒数内。

- Limit, Continue - 这用于对列表操作的结果进行分页。这些选项不支持观察操作。

注意，如果你为 Watch 操作指定一个不存在的命名空间，你将不会收到 NotFound 错误。

还要注意的是，如果你指定了过期的 ResourceVersion，你在调用 Watch 方法时不会收到错误，但会得到ERROR事件，其中包含 metav1.Status 对象，表示一个Reason的值 `metav1.StatusReasonExpired`。

metav1.Status 是一个基础对象，用来构建使用客户集的调用所返回的错误。你将能够在 "错误和状态" 部分了解更多。

## 错误和状态

如第一章所示，Kubernetes API 定义了 Kinds 来与调用者交换数据。目前，你应该考虑 Kinds 与资源有关，要么 Kind 有资源的单数名称（如Pod），要么 Kind 为资源列表（如PodList）。当一个 API 操作既没有返回资源也没有返回资源列表时，它使用一个普通的Kind，`metav1.Status`，来表示操作的状态。

### metav1.Status结构体的定义

`metav1.Status` 结构体的定义如下：

```go
type Status struct {
     Status      string
     Message     string
     Reason      StatusReason
     Details     *StatusDetails
     Code        int32
}
```

- Status - 这表示操作的状态，是 `metav1.StatusSuccess` 或 `metav1.StatusFailure`。

- Message - 这是对操作状态的自由形式的人类可读描述。

- Code - 这表示为操作返回的 HTTP 状态代码。

- Reason（原因）--这表示操作处于失败状态的原因。原因与给定的HTTP状态代码有关。定义的原因有：

  - StatusReasonBadRequest (400) - 这个请求本身是无效的。这与 `StatusReasonInvalid `不同，后者表明 API 调用可能成功，但数据无效。回复StatusReasonBadRequest 的请求永远不可能成功，无论数据如何。

  - StatusReasonUnauthorized (401) - 授权凭证丢失、不完整或无效。

  - StatusReasonForbidden (403) - 授权证书是有效的，但对资源的操作对这些证书是禁止的。

  - StatusReasonNotFound (404) - 请求的资源或资源无法找到。

  - StatusReasonMethodNotAllowed (405) - 在资源中请求的操作是不允许的，因为它没有实现。一个回复StatusReasonMethodNotAllowed的请求永远不会成功，不管是什么数据。

  - StatusReasonNotAcceptable (406) - 客户端在 Accept 头中指出的接受类型都不可能。回复 StatusReasonNotAcceptable 的请求永远不会成功，无论数据如何。

  - StatusReasonAlreadyExists (409) - 正在创建的资源已经存在。

  - StatusReasonConflict (409) - 由于冲突，请求无法完成--例如，由于操作试图用旧的资源版本更新资源，或者由于删除操作中的前提条件没有被遵守。

  - StatusReasonGone (410) - 项目已不再可用。

  - StatusReasonExpired (410) - 内容已经过期，不再可用--例如，当用过期的资源版本执行List或Watch操作时。

  - StatusReasonRequestEntityTooLarge (413) - 请求实体太大。

  - StatusReasonUnsupportedMediaType (415) - 此资源不支持 Content-Type 标头中的内容类型。回复 StatusReasonUnsupportedMediaType 的请求永远不会成功，不管是什么数据。

  - StatusReasonInvalid (422) - 为创建或更新操作发送的数据是无效的。Causes字段列举了数据的无效字段。

  - StatusReasonTooManyRequests (429) - 客户端应该至少等待 **Details** 字段 RetryAfterSeconds 中指定的秒数，才能再次执行操作。

  - StatusReasonUnknown (500) - 服务器没有指出任何失败的原因。

  - StatusReasonServerTimeout (500) - 可以到达服务器并理解请求，但不能在合理时间内完成操作。客户端应该在 **Details** 字段 RetryAfterSeconds 中指定的秒数后重试该请求。

  - StatusReasonInternalError (500) - 发生了一个内部错误；它是意料之外的，调用的结果是未知的。

  - StatusReasonServiceUnavailable (503) - 请求是有效的，但是所请求的服务在这个时候不可用。一段时间后重试该请求可能会成功。

  - StatusReasonTimeout (504) - 在请求中指定的超时时间内不能完成操作。如果指定了 Details 字段的RetryAfterSeconds字段，客户端应该在再次执行该操作之前等待这个秒数。

- Details -- 这些可以包含更多关于原因的细节，取决于 Reason 字段。

Details 字段的 StatusDetails 类型定义如下：

```go
type StatusDetails struct {
     Name string
     Group string
     Kind string
     UID types.UID
     Causes []StatusCause
     RetryAfterSeconds int32
}
```

- 如果指定的话，Name、Group、Kind 和 UID 字段表明哪个资源受到了故障的影响。

- RetryAfterSeconds 字段，如果指定的话，表示客户端在再次执行操作之前应该等待多少秒。

- Causes 字段列举了失败的原因。当执行创建或更新操作导致 StatusReasonInvalid 原因的失败时，Causes 字段列举了无效的字段和每个字段的错误类型。

- Causes 字段的 StatusCause 类型定义如下：

  ```go
  type StatusCause struct {
       Type       CauseType
       Message    string
       Field      string
  }
  ```

### CllientSet 操作返回的错误

本章前面包含了对 Clientset 提供的各种操作的描述，这些操作一般会返回一个错误，你可以使用 errors 包中的函数来测试错误的原因--例如，用 IsAlreadyExists 这个函数。

这些错误的具体类型是 `errors.StatusError`，定义为：

```go
type StatusError struct {
     ErrStatus metav1.Status
}
```

可以看出，这个类型只包括本节前面已经探讨过的 `metav1.Status` 结构体。为这个 StatusError 类型提供了函数来访问底层的Status。

- `Is<ReasonValue>(err error) bool ` - 本节前面列举的每个 Reason 值都有一个，表示错误是否属于特定状态。
- `FromObject(obj runtime.Object) error ` - 当你在 Watch 操作中接收到 `metav1.Status` 时，你可以用这个函数建立一个 StatusError 对象。
- `(e *StatusError) Status() metav1.Status ` - 返回基础状态。
- `ReasonForError(err error) metav1.StatusReason` - 返回基础状态的原因。
- `HasStatusCause(err error, name metav1.CauseType) bool` - 这表明一个错误是否声明了一个特定的原因，并给出了CauseType。
- `StatusCause(err error, name metav1.CseType) (metav1.StatusCause, bool) `- 如果给定的CauseType存在，返回该原因，否则返回false。
- `SuggestsClientDelay(err error) (int, bool) `- 这表明错误是否在状态的RetryAfterSeconds字段中指示了一个值以及该值本身。

## RESTClient

在本章前面的 "使用clientset" 部分，你可以为 Kubernetes API 的每个分组/版本获得一个 REST 客户端。例如，下面的代码返回 `Core/v1` 分组的REST客户端：

```go
restClient := clientset.CoreV1().RESTClient()
```

restClient 对象实现了 rest.Interface 接口，定义如下：

```go
type Interface interface {
     GetRateLimiter()           flowcontrol.RateLimiter
     Verb(verb string)          *Request
     Post()                     *Request
     Put()                      *Request
     Patch(pt types.PatchType)  *Request
     Get()                      *Request
     Delete()                   *Request
     APIVersion()                schema.GroupVersion
}
```

在这个接口中，你可以看到通用方法 Verb 和辅助方法 Post、Put、Patch、Get 和 Delete，它们返回 Request 对象。

### 构建Request

Request 结构体只包含私有字段，它提供了对 Request 进行个性化处理的方法。如第1章所示，Kubernetes 资源或子资源的路径形式（根据操作和资源的不同，有些段可能不存在）如下：

```
/apis/<group>/<version>
    /namespaces/<namesapce_name>
        /<resource>
            /<resource_name>
                /<subresource>
```

以下方法可以用来建立这个路径。注意， **<group>** 和 **<version> **段是固定的，因为REST客户端是特定于一个分组和版本的。

- `Namespace(namespace string) *Request; `

  `NamespaceIfScoped(namespace string, scoped bool) *Request `- 这些表明要查询的资源的名称空间。NamespaceIfScoped 只有在请求被标记为作用域时才会添加命名空间部分。

- `Resource(resource string) *Request` - 这表示要查询的资源。

- `Name(resourceName string) *Request` - 这表示要查询的资源的名称。

- `SubResource(subresources ...string) *Request` - 这表示要查询的资源的子资源。

- `Prefix(segments ...string) *Request; Suffix(segments ...string) *Request `- 在请求路径的开头或结尾添加段。前缀段将被添加到 "命名空间 "段之前。后缀段将被添加到子资源段之后。对这些方法的新调用将在现有的基础上增加前缀和后缀。

- `AbsPath(segments ...string) *Request` - 用所提供的段重设前缀。

下面的方法用查询参数、正文和头文件完成请求：

TBD

### 执行请求

一旦建立了Request，我们就可以执行它。可以使用Request对象上的以下方法：

- `Do(ctx context.Context) Result ` - 这将执行请求并返回 Result 对象。我们将在下一节看到如何利用这个结果对象。
- `Watch(ctx context.Context) (watch.Interface, error)` - 在请求的位置上执行 Watch 操作，并返回实现 `watch.Interface` 接口的对象，用来接收事件。你可以参阅本章的 "观察资源"一节，了解如何使用返回的对象。
- `Stream(ctx context.Context) (io.ReadCloser, error)` - 这将执行请求，并通过 ReadCloser 将结果体流出来。
- `DoRaw(ctx context.Context) ([]byte, error)` - 这将执行请求并将结果作为字节数组返回。

### 对结果的利用

当你在 Request 上执行 Do() 方法时，该方法返回 Result 对象。

结果结构没有任何公共字段。以下方法可以用来获取结果的信息：

- `Into(obj runtime.Object) error` - 如果可能的话，这将解码并将结果体的内容存储到对象中。作为参数传递的对象的具体类型必须与正文中定义的类型相匹配。同时返回执行请求的错误。
- `Error() error` - 这将返回执行请求的错误。这个方法在执行一个没有返回正文内容的请求时很有用。
- `Get() (runtime.Object, error) ` - 这个方法将结果体的内容解码并作为一个对象返回。返回对象的具体类型将与正文中定义的类型相匹配。同时返回执行请求的错误。
- `Raw() ([]byte, error)` - 这将返回作为字节数组的body，以及执行请求的错误。
- `StatusCode(statusCode *int) Result` - 这将状态代码存储到传递的参数中，并返回结果，因此该方法可以被链起来。
- `WasCreated(wasCreated *bool) Result` - 这存储了一个值，表明请求创建的资源是否已经被成功创建，并返回结果，因此该方法可以被链起来。
- `Warnings() []net.WarningHeader` - 这将返回包含在Result中的Warnings列表。

### 以表格形式获取结果

TBD

## 发现客户端

Kubernetes API 提供了发现 API 所提供的资源的端点。kubectl 正在使用这些端点来显示 kubectl api-resources 命令的结果（图6-1）。

![](images/533908_1_En_6_Fig1_HTML.png)

客户端可以通过调用 clientset 上的 Discovery() 方法来获得（参见第6章 "获得clientset" 部分中如何获得 **clientset**），或者使用 **discovery** 包提供的函数。

```go
import "k8s.io/client-go/discovery"
```


所有这些函数，都希望有一个 `rest.Config`，作为一个参数。你可以在第6章的 "连接到集群" 部分看到如何获得这样一个 `rest.Config` 对象。

- NewDiscoveryClientForConfig 将返回一个使用所提供的 `rest.Config` 的 DiscoveryClient。

  ```go
  NewDiscoveryClientForConfig( 
  	c *rest.Config,
  ) (*DiscoveryClient, error)
  ```

- NewDiscoveryClientForConfigOrDie 与前一个函数类似，但在出错的情况下会 panic ，而不是返回错误。这个函数可以用在一个硬编码的配置上，我们要断言它的有效性。

  ```go
  NewDiscoveryClientForConfigOrDie(
  	c *rest.Config,
  ) *DiscoveryClient
  ```

- NewDiscoveryClientForConfigAndClient

  ```go
  NewDiscoveryClientForConfigAndClient(
  	c *rest.Config,
  	httpClient *http.Client,
  ) (*DiscoveryClient, error)
  ```

  之前的函数 NewDiscoveryClientForConfig 使用了一个用函数 `rest.HTTPClientFor` 构建的默认HTTP客户端。如果你想在构建 `DiscoveryClient` 之前个性化HTTP客户端，你可以使用这个函数来代替。

## RESTMapper

TBD

## 总结

在本章中，你已经看到了如何连接到集群以及如何获得 Clientset。它是一组客户端，每个 Group-Version 都有一个，你可以用它来执行对资源的操作（获取、列表、创建等）。

你还涵盖了 REST 客户端，在内部由 Clientset 使用，开发者可以用它来建立更具体的请求。最后，本章介绍了 Discovery 客户端，用于以动态方式发现Kubernetes API 提供的资源。

下一章介绍了如何测试用 Client-go 库编写的应用程序，使用它所提供的客户端的 fack 实现。

