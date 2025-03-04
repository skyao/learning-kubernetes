---
title: "将管理器资源注入 Reconciler 中"
linkTitle: "注入管理器"
weight: 40
date: 2021-02-01
description: >
  将管理器资源注入 Reconciler 中
---

管理器为控制器提供共享资源，包括读取和写入 Kubernetes 资源的客户端、从本地缓存中读取资源的缓存以及解析资源的方案。Reconcile 函数需要访问这些共享资源。有两种方法来共享它们：

## 创建 Reconciler 结构体时传递数值

当控制器被创建时，你正在传递一个 Reconcile 结构体的实例，实现 Reconciler 接口：

```go
type MyReconciler struct {}
err = builder.
            ControllerManagedBy(mgr).
            For(&mygroupv1alpha1.MyResource{}).
            Owns(&corev1.Pod{}).
            Complete(&MyReconciler{})
```

在这之前，管理器已经被创建，你可以使用管理器上的 Getters 来访问共享资源。

作为例子，下面是如何从新创建的管理器中获取客户端、缓存和 schema ：

```go
mgr, err := manager.New(
      config.GetConfigOrDie(),
      manager.Options{
            manager.Options{
                  Scheme: scheme,
          },
     },
)
// handle err
mgrClient := mgr.GetClient()
mgrCache := mgr.GetCache()
mgrScheme := mgr.GetScheme()
```

你可以在 Reconciler 结构体中添加字段来传递这些值：

```go
type MyReconciler struct {
     client client.Client
     cache cache.Cache
     scheme *runtime.Scheme
}
```

最后，你可以在创建控制器的过程中传递这些值：

```go
err = builder.
            ControllerManagedBy(mgr).
            For(&mygroupv1alpha1.MyResource{}).
            Owns(&corev1.Pod{}).
            Complete(&MyReconciler{
      client: mgr.GetClient(),
      cache: mgr.GetCache(),
      scheme: mgr.GetScheme(),
})
```

## 使用 Injector

controller-runtime 库提供了一个注入器（Injectors）系统，用于将共享资源注入 Reconcilers，以及其他结构体，如你自己实现的 Sources、EventHandlers 和 Predicates。

Reconciler 的实现需要实现 inject 包中的特定 Injector 接口：`inject.Client`、`inject.Cache`、`inject.Scheme`，等等。

这些方法将在初始化时被调用，当你调用 `controller.New` 或 `builder.Complete`。为此，需要为每个接口创建一个方法，例如：

```go
type MyReconciler struct {
     client client.Client
     cache cache.Cache
     scheme *runtime.Scheme
}
func (a *MyReconciler) InjectClient( c client.Client,) error {
      a.client = c
      return nil
}
func (a *MyReconciler) InjectCache( c cache.Cache, ) error {
      a.cache = c
      return nil
}
func (a *MyReconciler) InjectScheme( s *runtime.Scheme, ) error {
      a.scheme = s
      return nil
}
```





