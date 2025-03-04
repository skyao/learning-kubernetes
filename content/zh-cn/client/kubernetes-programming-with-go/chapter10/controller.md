---
title: "Controller"
linkTitle: "Controller"
weight: 30
date: 2021-02-01
description: >
  Controller
---

第二个重要的抽象是控制器 / Controller。控制器负责实现特定 Kubernetes 资源的实例所给出的规范（Spec）。(在 operator 的情况下，自定义资源是由 operator 处理的）。

为此，控制器观察特定的资源（至少是相关的自定义资源，在本节中称为 "主要资源"），并接收这些资源的 watch 事件（即，创建、更新、删除）。当事件发生在资源上时，控制器用包含受事件影响的 "主要资源" 实例的名称和命名空间的请求填充一个队列。

请注意，入队的对象只是被 operator 监视的主要资源的实例。如果事件是由另一个资源的实例接收的，则通过跟踪所有者参考（ownerReference）找到主资源。例如，Deployment 控制器观察 Deployment 资源和 ReplicaSet 资源。所有 ReplicaSet 实例都包含一个指向 Deployment 实例的 ownerReference。

- 当 Deployment 被创建时，控制器会收到一个 Create 事件，刚刚创建的 Deployment 实例被入队

- 当 ReplicaSet 被修改时（例如，被一些用户修改），这个 ReplicaSet 会收到一个 Update 事件，控制器会使用 ReplicaSet 中包含的 ownerReference 找到被更新的 ReplicaSet 引用的 Deployment。然后，被引用的 Deployment 实例被入队。


控制器实现 Reconcile 方法，每当队列中出现一个 Request 时，它就会被调用。这个 Reconcile 方法接收 Request 作为参数，其中包含要调和（Reconcile）的主要资源的名称和命名空间。

请注意，触发请求的事件不是请求的一部分，因此，Reconcile 方法不能依赖这个信息。此外，如果事件发生在一个被拥有的（owned）资源上，只有主要资源被入队，Reconcile 方法不能依赖哪个被拥有的（owned）资源触发了该事件。

此外，由于多个事件可能在短时间内发生，并与同一主要资源有关，请求可以被批量处理，以限制入队的请求数量。

## 创建 Controller

要创建控制器，你需要使用提供的 New 函数：

```go
import (
      "sigs.k8s.io/controller-runtime/pkg/controller"
)
controller, err = controller.New(
     "my-operator", mgr,
     controller.Options{
            Reconciler: myReconciler,
})
```

Reconciler 选项是必需的，其值是一个必须实现 Reconciler 接口的对象，定义为：

```go
type Reconciler interface {
      Reconcile(context.Context, Request) (Result, error)
}
```

作为一种设施，提供了 `reconcile.Func` 类型，它实现了 Reconciler 接口，并且与 Reconcile 方法的签名相同的函数类型。

```go
type Func func(context.Context, Request) (Result, error)
func (r Func) Reconcile(ctx context.Context, o Request) (Result, error) { 
    return r(ctx, o) 
}
```

由于这个 `reconcile.Func` 类型，你可以用 Reconcile 签名构造一个函数，并把它分配给 Reconciler 选项。比如说：

```go
controller, err = controller.New(
     "my-operator", mgr,
     controller.Options{
            Reconciler: reconcile.Func(reconcileFunction),
})

func reconcileFunction(
     ctx context.Context,
     r reconcile.Request,
) (reconcile.Result, error) {
      [...]
      return reconcile.Result{}, nil
}
```

## 监控资源

在控制器创建后，你需要向容器指出哪些资源需要观察，以及这些资源是主要资源还是被拥有的（owned）资源。

控制器上的 Watch 方法被用来添加一个 Watch。该方法定义如下：

```go
Watch(
     src source.Source,
     eventhandler handler.EventHandler,
     predicates ...predicate.Predicate,
) error
```

第一个参数表示什么是要观察的事件的来源，其类型是 `source.Source` 接口。controller-runtime 库为 Source 接口提供了两种实现方式：

- Kind source 用于监视特定种类（kind）的 Kubernetes 对象的事件。Kind 结构体中的 Type 字段是必需的，其值是所需类型的对象。例如，如果我们想监视 Deployment，src 参数的值将是：

  ```go
  controller.Watch(
        &source.Kind{
              Type: &appsv1.Deployment{},
        },
        ...
  ```

- Channel source 是用来观察来自集群外的事件的。Channel 结构体的 Source 字段是必需的，它的值是一个发射 `event.GenericEvent` 类型对象的通道。

第二个参数是事件处理程序，其类型是 `handler.EventHandler` 接口。controller-runtime 库为 EventHandler 接口提供了两种实现方式：

- EnqueueRequestForObject 事件处理程序用于控制器处理的主要资源。在这种情况下，控制器将把连接到事件的对象放入队列中。例如，如果控制器是处理第9章中创建的自定义资源的 operator，你将写道：

  ```go
  controller.Watch(
        &source.Kind{
              Type: &mygroupv1alpha1.MyResource{},
        },
        &handler.EnqueueRequestForObject{},
  )
  ```

- EnqueueRequestForOwner 事件处理程序用于由主资源拥有( owned)的资源。EnqueueRequestForOwner 的一个字段是必需的： OwnerType。这个字段的值是主资源类型的对象；控制器将跟踪 ownerReferences，直到找到这种类型的对象，并将这个对象放入队列。

  例如，如果控制器处理 MyResource 主资源，并且正在创建 Pod 来实现 MyResource，它将希望使用这个事件处理程序来观察 Pod 资源，并指定一个 MyResource 对象作为 OwnerType。

- 如果字段 IsController 被设置为true，控制器将只考虑 `Controller: true` 的 ownerReferences。

  ```go
  controller.Watch(
        &source.Kind{
              Type: &corev1.Pod{},
        },
        &handler.EnqueueRequestForOwner{
              OwnerType: &mygroupv1alpha1.MyResource{},
              IsController: true,
        },
  )
  ```

第三个参数是一个可选的谓词列表，其类型为 `predicate.Predicate` 。controller-runtime 库为 Predicate 接口提供了几种实现：

- Funcs 是最通用的实现。Funcs 的结构体定义如下：

  ```go
  type Funcs struct {
        // Create returns true if the Create event
       // should be processed
        CreateFunc func(event.CreateEvent) bool
        // Delete returns true if the Delete event
       // should be processed
        DeleteFunc func(event.DeleteEvent) bool
        // Update returns true if the Update event
       // should be processed
        UpdateFunc func(event.UpdateEvent) bool
        // Generic returns true if the Generic event
       // should be processed
        GenericFunc func(event.GenericEvent) bool
  }
  ```

  你可以把这个结构体的一个实例传递给 Watch 方法，作为 Predicate。

  未定义的字段将表示匹配类型的事件应该被处理。

  对于非 nil 字段，匹配事件的函数将被调用（注意，当源是一个通道时，GenericFunc 将被调用；见前文），如果函数返回 true，事件将被处理。

- 使用 Predicate 的这种实现，你可以为每个事件类型定义一个特定的函数。

  ```go
  func NewPredicateFuncs(
  	filter func(object client.Object) bool,
  ) Funcs
  ```

  该函数接受一个 *filter* 函数，并返回一个 Funcs  结构体，该过滤器被应用于所有事件。使用这个函数，你可以定义一个适用于所有事件类型的单一过滤器。

- `ResourceVersionChangedPredicate` 结构体将定义一个只用于 UpdateEvent 的 filter。

  使用这个 predicate ，所有的Create、Delete 和 Generic 事件将被处理，不需要过滤，而 Update 事件将被过滤，以便只处理有 `metadata.resourceVersion` 变化的更新。

  每次保存资源的新版本时，`metadata.resourceVersion` 字段都会被更新，不管资源的变化是什么。

- `GenerationChangedPredicate` 结构体定义了一个只用于更新事件过滤器。

  使用这个谓词，所有的 Create、Delete 和 Generic 事件都将被处理，无需过滤，而 Update 事件将被过滤，因此只有具有 `metadata.Generation` 增量的更新才会被处理。

  每次发生资源的 Spec 部分的更新时，API 服务器都会按顺序递增 `metadata.Generation`。

  请注意，有些资源并不尊重这个假设。例如，当 Annotations 字段被更新时，Deployment 的 Generation 也会被递增。

  对于自定义资源，只有当状态子资源被启用时，生成才会被递增。

- `AnnotationChangedPredicate` 结构体定义了一个只用于更新事件过滤器。

  使用这个谓词，所有的创建、删除和通用事件都将被处理，而更新事件将被过滤，因此只有元数据.注释变化的更新才会被处理。

## 第一个例子

在第一个例子中，你将创建管理器和控制器。控制器将管理主要的自定义资源 MyResource，并观察这个资源以及 Pod 资源。

Reconcile 函数将只显示 MyResource 实例的命名空间和名称来进行调节。

```go
package main
import (
      "context"
      "fmt"
      corev1 "k8s.io/api/core/v1"
      "k8s.io/apimachinery/pkg/runtime"
      "sigs.k8s.io/controller-runtime/pkg/client/config"
      "sigs.k8s.io/controller-runtime/pkg/controller"
      "sigs.k8s.io/controller-runtime/pkg/handler"
      "sigs.k8s.io/controller-runtime/pkg/manager"
      "sigs.k8s.io/controller-runtime/pkg/reconcile"
      "sigs.k8s.io/controller-runtime/pkg/source"
      clientgoscheme "k8s.io/client-go/kubernetes/scheme"
      mygroupv1alpha1 "github.com/myid/myresource-crd/pkg/apis/mygroup.example.com/v1alpha1"
)
func main() {
      scheme := runtime.NewScheme()                     ❶
      clientgoscheme.AddToScheme(scheme)
      mygroupv1alpha1.AddToScheme(scheme)
      mgr, err := manager.New(                          ❷
            config.GetConfigOrDie(),
            manager.Options{
                  Scheme: scheme,
            },
      )
      panicIf(err)
      controller, err := controller.New(                 ❸
            "my-operator", mgr,
            controller.Options{
                  Reconciler: &MyReconciler{},
            })
      panicIf(err)
      err = controller.Watch(                             ❹
            &source.Kind{
                  Type: &mygroupv1alpha1.MyResource{},
            },
            &handler.EnqueueRequestForObject{},
      )
      panicIf(err)
      err = controller.Watch(                              ❺
            &source.Kind{
                  Type: &corev1.Pod{},
            },
            &handler.EnqueueRequestForOwner{
                  OwnerType:    &corev1.Pod{},
                  IsController: true,
            },
      )
      panicIf(err)
      err = mgr.Start(context.Background())                ❻
      panicIf(err)
}
type MyReconciler struct{}                                 ➐
func (o *MyReconciler) Reconcile(                          ➑
     ctx context.Context,
     r reconcile.Request,
) (reconcile.Result, error) {
      fmt.Printf("reconcile %v\n", r)
      return reconcile.Result{}, nil
}
// panicIf panic if err is not nil
// Please call from main only!
func panicIf(err error) {
      if err != nil {
            panic(err)
      }
}
```

➊ 用本地资源和自定义资源 MyResource 创建 scheme。

➋ 使用刚刚创建的 scheme 创建管理器

➌ 创建控制器，连接到管理器，并传递 Reconciler 实现。

➍ 开始观察作为主要资源的 MyResource 实例

➎ 开始观察 Pod 实例，作为自有（owned）资源

➏ 启动管理器。这个函数是长期运行的，只有在发生错误时才会返回

➐ 实现 Reconciler 接口的类型

➑ 实现 Reconcile 方法。这将显示要 reconcile 的实例的名称空间和名称

## 使用Controller Builder

Controller-runtime 库提出了一个控制器生成器 / Controller Builder，使控制器的创建更加简洁。

```go
import (
     "sigs.k8s.io/controller-runtime/pkg/builder"
)
func ControllerManagedBy(m manager.Manager) *Builder
```

ControllerManagedBy 函数被用来启动一个新的 ControllerBuilder。构建的控制器将被添加到 m manager 中。一个流畅的接口帮助配置构建：

- `For(object client.Object, opts ...ForOption) *Builder` - 该方法用于指示控制器处理的主要资源。它只能被调用一次，因为一个控制器只能有一个主要资源。这将在内部调用 Watch 函数与事件处理程序 EnqueueRequestForObject 。

  可以用 WithPredicates 函数为这个 watch 添加谓词（Predicates），其结果实现了 ForOption 接口。

- `Owns(object client.Object, opts ...OwnsOption) *Builder` - 这个方法用来表示控制器拥有（owned）的资源。这将在内部调用 Watch 函数与事件处理程序 EnqueueRequestForOwner。

  可以用 WithPredicates 函数为这个Watch添加谓词，该函数的结果实现了OwnsOption接口。

- `Watches(src source.Source, eventhandler handler.EventHandler, opts ...WatchesOption) *Builder` - 这个方法可以用来添加更多For或Owns方法没有涵盖的 watcher --例如，具有 **Channel** source 的观察者。

  可以用 WithPredicates 函数为该 watch 添加谓词，该函数的结果实现了 WatchesOption 接口。

- `WithEventFilter(p predicate.Predicate) *Builder ` - 这个方法可以用来添加所有用 For、Owns 和 Watch 方法创建的观察者共有的谓词。

- `WithOptions(options controller.Options) *Builder` - 此处设置将在内部传递给 `controller.New` 函数的选项。

- WithLogConstructor(- 这设置了 logConstructor 选项。

  ```go
  func(*reconcile.Request) logr.Logger,
  
  ) *Builder
  ```

- `Named(name string) *Builder` - 这设置了构造函数的名称。它应该只使用下划线和字母数字字符。默认情况下，它是主要资源的 kind 的小写版本。

- build() 构建并返回控制器。

  ```go
  Build( 
  	r reconcile.Reconciler,
  ) (controller.Controller, error)
  ```

- `Complete(r reconcile.Reconciler) error` --这就建立了控制器。你一般不需要直接访问控制器，所以你可以使用这个不返回控制器值的方法，而不是Build。

## 使用ControllerBuilder的第二个例子

在这个例子中，你将使用 ControllerBuilder 构建控制器，而不是使用 Controller.New 函数和控制器上的 Watch 方法。

```go
package main
import (
      "context"
      "fmt"
      corev1 "k8s.io/api/core/v1"
      "k8s.io/apimachinery/pkg/runtime"
      clientgoscheme "k8s.io/client-go/kubernetes/scheme"
      "sigs.k8s.io/controller-runtime/pkg/builder"
      "sigs.k8s.io/controller-runtime/pkg/client"
      "sigs.k8s.io/controller-runtime/pkg/client/config"
      "sigs.k8s.io/controller-runtime/pkg/manager"
      "sigs.k8s.io/controller-runtime/pkg/reconcile"
      mygroupv1alpha1 "github.com/feloy/myresource-crd/pkg/apis/mygroup.example.com/v1alpha1"
)
func main() {
      scheme := runtime.NewScheme()
      clientgoscheme.AddToScheme(scheme)
      mygroupv1alpha1.AddToScheme(scheme)
      mgr, err := manager.New(
            config.GetConfigOrDie(),
            manager.Options{
                  Scheme: scheme,
},
      )
      panicIf(err)
      err = builder.
            ControllerManagedBy(mgr).
            For(&mygroupv1alpha1.MyResource{}).
            Owns(&corev1.Pod{}).
            Complete(&MyReconciler{})
      panicIf(err)
      err = mgr.Start(context.Background())
      panicIf(err)
}
type MyReconciler struct {}
func (a *MyReconciler) Reconcile(
      ctx context.Context,
      req reconcile.Request,
) (reconcile.Result, error) {
      fmt.Printf("reconcile %v\n", req)
      return reconcile.Result{}, nil
}
func panicIf(err error) {
      if err != nil {
            panic(err)
      }
}
```





