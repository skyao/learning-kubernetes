---
title: "[第7章]测试使用Client-go的应用程序"
linkTitle: "测试使用Client-go的应用程序"
weight: 70
date: 2021-02-01
description: >
  client-go类库
---



Client-go 库提供了一些可以与 Kubernetes API 一起行动的客户端。

- `kubernetes.Clientset ` 提供了一组客户端，每个分组/版本的API都有一个，用于执行对资源的Kubernetes操作（创建、更新、删除等）。
- rest.RESTClient 提供了一个客户端，用于对资源执行REST操作（获取、发布、删除等）。

- discovery.DiscoveryClient 提供了一个客户端来发现由API提供的资源。


所有这些客户端都实现了由 Client-go 库定义的接口：kubernetes.Interface、rest.Interface 和 discovery.DiscoveryInterface。

此外，Client-go库提供了这些接口的假实现，以帮助你为你的功能编写单元测试。这些假的实现被定义在 fack 包中，每个包都位于真实实现的目录内：kubernetes/fake、rest/fake 和 discovery/fake。

testing 目录包含 fack 客户端使用的常用工具--例如，对象跟踪器或跟踪调用的系统。你将在本章中了解到这些工具。

为了能够使用这些客户端测试你的函数，函数需要定义一个参数来传递客户端的实现，而且参数的类型必须是接口，而不是具体类型。比如说：

```go
func CreatePod(
     ctx context.Context,
     clientset kubernetes.Interface,
     name string,
     namespace string,
     image string,
) (pod *corev1.Pod, error)
```

这样，你将在函数之外创建客户端，并简单地在函数内使用任何实现。

对于真正的代码，你将创建客户端，如第6章所定义。在测试中，你将使用 fack 包中的辅助函数来替代客户端。

## fack clientset

下面的函数用于创建 fack clientset ：

```go
import "k8s.io/client-go/kubernetes/fake"
func NewSimpleClientset(
     objects ...runtime.Object,
) *Clientset
```

fack clientset 是由一个对象跟踪器支持的，它处理创建、更新和删除操作，没有任何验证或突变，并且它在响应获取和列表操作时返回对象。

你可以把 Kubernetes 对象的列表作为参数传递，这些对象将被添加到 clientset 的对象跟踪器中。例如，使用下面的方法来创建一个 fack clientset，并调用前面定义的 CreatePod 函数：

```go
import "k8s.io/client-go/kubernetes/fake"
clientset := fake.NewSimpleClientset()
pod, err := CreatePod(
     context.Background(),
     clientset,
     aName,
     aNs,
     anImage,
)
```

在测试过程中，当你调用被测试的函数（在本例中是 CreatePod）后，你有几种方法来验证该函数是否完成了你所期望的。让我们考虑一下 CreatePod 函数的这个实现：

```go
func CreatePod(
     ctx context.Context,
     clientset kubernetes.Interface,
     name string,
     namespace string,
     image string,
) (pod *corev1.Pod, err error) {
     podToCreate := corev1.Pod{
          Spec: corev1.PodSpec{
               Containers: []corev1.Container{
                    {
                         Name:  "runtime",
                         Image: image,
                    },
               },
          },
     }
     podToCreate.SetName(name)
     return clientset.CoreV1().
     Pods(namespace).
     Create(
          ctx,
          &podToCreate,
          metav1.CreateOptions{},
     )
}
```

### 检查函数的结果

当用 fack clientset  调用 `CreatePod` 函数时，实际的 Kubernetes API 将不会被调用，资源不会在 etcd 数据库中生成，也不会对资源进行验证和变异。相反，资源将被原封不动地存储在内存存储中，只进行最小的转换。

在这个例子中，传递给 Create 函数的 podToCreate 和 Create 函数返回的 pod 之间的唯一转换是命名空间，它通过调用传递到 Pods(namespace)，并被添加到返回的 Pod 中。

为了测试 CreatePod 函数返回的值是否是你所期望的，你可以编写以下测试：

```go
func TestCreatePod(t *testing.T) {
     var (
          name      = "a-name"
          namespace = "a-namespace"
          image     = "an-image"
          wantPod = &corev1.Pod{
               ObjectMeta: v1.ObjectMeta{
                    Name:      "a-name",
                    Namespace: "a-namespace",
               },
               Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                         {
                              Name:  "runtime",
                              Image: "an-image",
                         },
                    },
               },
          }
     )
     clientset := fake.NewSimpleClientset()
     gotPod, err := CreatePod(
          context.Background(),
          clientset,
          name,
          namespace,
          image,
     )
     if err != nil {
          t.Errorf("err = %v, want nil", err)
     }
     if !reflect.DeepEqual(gotPod, wantPod) {
          t.Errorf("CreatePod() = %v, want %v",
               gotPod,
               wantPod,
          )
     }
}
```

这个测试将断言--给定一个名字、一个命名空间和一个图像--函数的结果将是wantPod，此时Pod中没有发生任何验证或变异。

不可能用这个测试来了解真正的客户端会发生什么，因为结果会有所不同--也就是说，真正的客户端和底层API会对对象进行突变，增加默认值，等等。

## 对行动的反应

假的客户集在没有任何验证或变异的情况下按原样存储资源。在测试中，你可能想模拟各种控制器对资源所做的改变。为此，假客户集提供了添加反应器的方法。Reactors是在特定资源上进行特定操作时执行的函数。

除了Watch和Proxy，所有操作的Reactors函数的类型定义如下：



TBD

