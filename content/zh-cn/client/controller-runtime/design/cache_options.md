---
title: "缓存选项"
linkTitle: "缓存选项"
weight: 90
date: 2021-02-01
description: >
  描述对未来缓存选项的设想
---

> https://github.com/kubernetes-sigs/controller-runtime/blob/main/designs/cache_options.md


这份文件描述了我们对未来缓存选项的设想。

## 目标

- 使每个人对我们想要支持的缓存的设置及其配置保持一致
- 确保我们既支持复杂的缓存设置，又提供一个直观的配置用户体验。

## 非目标

- 描述缓存本身的设计和实现。我们的假设是，最细的级别是 "每个对象的多命名空间和不同的选择器"/"per-object multiple namespaces with distinct selectors"，这可以通过一个 "元缓存"/"meta cache" 来实现，这个 "元缓存" 委托给每个对象，并通过扩展当前的多命名空间缓存。
- 概述这些设置何时实施的时间表。只要有人站出来做实际的工作，实施就会逐渐发生。

## 提案

```go
const (
   AllNamespaces = corev1.NamespaceAll
)

type Config struct {
  // LabelSelector 指定标签选择器。nil值允许默认。
  LabelSelector labels.Selector

  // FieldSelector 指定字段选择器。nil值允许默认。
  FieldSelector fields.Selector

  // Transform 指定转换函数。nil值允许默认。
  Transform     toolscache.TransformFunc

  // UnsafeDisableDeepCopy 指定针对缓存的List和Get请求是否不需要DeepCopy。nil值允许默认。
  UnsafeDisableDeepCopy *bool
}


type ByObject struct {
  // Namespaces 将命名空间名称映射到缓存设置。如果设置，只有该映射中的命名空间将被缓存。
  // 
  // 映射值中的设置如果因为整体值为零或具体设置为零而未设置，将被默认。为防止这种情况，请为特定的设置使用一个空值。
  // 
  // 可以通过使用 AllNamespaces 常量作为映射键，为某些命名空间设置特定的Config，但缓存所有命名空间。这将包括所有没有特定设置的名字空间。
  // 
  // nil的映射允许将其默认为缓存的DefaultNamespaces设置。
  //
  // 空的映射可以防止这种情况。
  //
  // 对于集群范围内的对象必须取消设置。
  Namespaces map[string]*Config

  // Config will be used for cluster-scoped objects and to default
  // Config in the Namespaces field.
  //
  // It gets defaulted from the cache'sDefaultLabelSelector, DefaultFieldSelector,
  // DefaultUnsafeDisableDeepCopy and DefaultTransform.
  Config *Config
}

type Options struct {
  // ByObject specifies per-object cache settings. If unset for a given
  // object, this will fall through to Default* settings.
  ByObject map[client.Object]*ByObject

  // DefaultNamespaces maps namespace names to cache settings. If set, it
  // will be used for all objects that have a nil Namespaces setting.
  //
  // It is possible to have a specific Config for just some namespaces
  // but cache all namespaces by using the `AllNamespaces` const as the map
  // key. This wil then include all namespaces that do not have a more
  // specific setting.
  //
  // The options in the Config that are nil will be defaulted from
  // the respective Default* settings.
  DefaultNamespaces map[string]*Config

  // DefaultLabelSelector is the label selector that will be used as
  // the default field label selector for everything that doesn't
  // have one configured.
  DefaultLabelSelector labels.Selector

  // DefaultFieldSelector is the field selector that will be used as
  // the default field selector for everything that doesn't have
  // one configured.
  DefaultFieldSelector fields.Selector

  // DefaultUnsafeDisableDeepCopy is the default for UnsafeDisableDeepCopy
  // for everything that doesn't specify this.
  DefaultUnsafeDisableDeepCopy *bool

  // DefaultTransform will be used as transform for all object types
  // unless they have a more specific transform set in ByObject.
  DefaultTransform toolscache.TransformFunc

  // HTTPClient is the http client to use for the REST client
  HTTPClient *http.Client

  // Scheme is the scheme to use for mapping objects to GroupVersionKinds
  Scheme *runtime.Scheme

  // Mapper is the RESTMapper to use for mapping GroupVersionKinds to Resources
  Mapper meta.RESTMapper

  // SyncPeriod determines the minimum frequency at which watched resources are
  // reconciled. A lower period will correct entropy more quickly, but reduce
  // responsiveness to change if there are many watched resources. Change this
  // value only if you know what you are doing. Defaults to 10 hours if unset.
  // there will a 10 percent jitter between the SyncPeriod of all controllers
  // so that all controllers will not send list requests simultaneously.
  //
  // This applies to all controllers.
  //
  // A period sync happens for two reasons:
  // 1. To insure against a bug in the controller that causes an object to not
  // be requeued, when it otherwise should be requeued.
  // 2. To insure against an unknown bug in controller-runtime, or its dependencies,
  // that causes an object to not be requeued, when it otherwise should be
  // requeued, or to be removed from the queue, when it otherwise should not
  // be removed.
  //
  // If you want
  // 1. to insure against missed watch events, or
  // 2. to poll services that cannot be watched,
  // then we recommend that, instead of changing the default period, the
  // controller requeue, with a constant duration `t`, whenever the controller
  // is "done" with an object, and would otherwise not requeue it, i.e., we
  // recommend the `Reconcile` function return `reconcile.Result{RequeueAfter: t}`,
  // instead of `reconcile.Result{}`.
  SyncPeriod *time.Duration

}
```

