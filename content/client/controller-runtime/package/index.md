---
title: "controller-runtime package概况"
linkTitle: "package概况"
weight: 20
date: 2021-02-01
description: >
  controller-runtime package概况
---



https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg



pkg 包提供了用于构建 controller 控制器的库。控制器实现了 Kubernetes API，是构建 operator、workload API、配置API、自动缩放器等的基础。

### client

客户端为读写Kubernetes对象提供了一个 读+写 的客户端。

### cache

缓存提供了一个读客户端，用于从本地缓存中读取对象。缓存可以注册 handler 来响应更新缓存的事件。

### manager

Manager是创建 controller 控制器的必要条件，并提供 controller 的共享依赖关系，如客户端、缓存、scheme 等。 controller 应该通过 manager 调用 Manager.Start 来启动。

### Controller

Controller 通过响应事件（对象创建、更新、删除）来实现 Kubernetes API，并确保对象的 Spec 中指定的状态与系统的状态相匹配。这被称为 reconcile（调和）。如果它们不匹配，Controller 将根据需要创建/更新/删除对象，使它们匹配。

Controller 被实现为处理 reconcile.Requests（为特定对象 reconcile 状态的请求）的工作队列。

与 http handler 不同的是，Controller 不直接处理事件，而是将 Requests 入列来最终 reconcile 对象。这意味着多个事件的处理可能会被批量进行，而且每次 reconcile 都必须读取系统的全部状态。

* Controller 需要提供一个 Reconciler 来执行从工作队列中拉出的工作。

* Controller 需要配置 Watches，以便将 reconcile.Requests 入列并对事件进行响应。

### Webhook

Admission Webhooks 是一种扩展 kubernetes APIs 的机制。Webhooks 可以配置目标事件类型（对象创建、更新、删除），API 服务器将在某些事件发生时向它们发送 AdmissionRequests。Webhooks 可以修改和（或）验证嵌入在 AdmissionReview 请求中的对象，并将响应发回给 API 服务器。

有2种类型的 Admission Webhooks ：修改（mutating）和验证（validating）的 Admission Webhooks 。修改的 webhook 是用来在 API 服务器接纳之前修改 core API 对象或 CRD 实例。验证 webhook 用于验证对象是否符合某些要求。

* Admission Webhooks 需要提供 Handler（s）来处理收到的 AdmissionReview 请求。

### Reconciler

Reconciler 是提供给 Controller 的一个函数，可以在任何时候用对象的名称和命名空间来调用。当被调用时，Reconciler 将确保系统的状态与 Reconciler 被调用时在对象中指定的内容相匹配。

例子： 为一个 ReplicaSet 对象调用 Reconciler。ReplicaSet 指定了5个副本，但系统中只存在 3 个 Pod。Reconciler 又创建了2个Pod，并将其 OwnerReference 设置为指向 ReplicaSet，并设置 controller=true。

* Reconciler 包含 Controller 的所有业务逻辑。

* Reconciler 通常只对单一对象类型工作。- 例如，它将只对 ReplicaSets 进行调节。对于单独的类型，使用单独的 Controller。如果你想从其他对象触发 reconcile，你可以提供一个映射（例如所有者引用/owner references），将触发 reconcile 的对象映射到被 reconcile 的对象。

* Reconciler 被提供给要 reconcile 的对象的名称/命名空间。

* Reconciler 不关心负责触发 Reconcile 的事件内容或事件类型。例如，ReplicaSet 是否被创建或更新并不重要，Reconciler 总是将系统中的 Pod 数量与它被调用时在对象中指定的数量进行比较。

### Source

resource.Source 是 Controller.Watch 的参数，提供了事件流。事件通常来自于 watch Kubernetes APIs（例如：Pod Create, Update, Delete）。

例如：source.Kind 使用 Kubernetes API Watch 端点，为 GroupVersionKind 提供创建、更新、删除事件。

* Source 通常通过 Watch API 为 Kubernetes 对象提供事件流（如对象创建、更新、删除）。

* 在几乎所有情况下，用户都应该只使用所提供的 Source 实现，而不是实现自己的。

### EventHandler

handler.EventHandler 是 Controller.Watch 的参数，它在响应事件时入队 reconcile.Requests。

例如：一个来自 Source 的 Pod 创建事件被提供给 eventhandler.EnqueueHandler，它入队包含 Pod 的名称/命名空间的reconcile.Request。

* EventHandlers 通过入队 reconcile.Requests 来为一个或多个对象处理事件。

* EventHandlers 可以将一个对象的事件映射到同一类型的另一个对象的 reconcile.Request 上。

* EventHandlers 可以将一个对象的事件映射到不同类型的另一个对象的 reconcile.Request 。例如，将 Pod 事件映射到拥有它的 ReplicaSet 的 reconcile.Request 上。

* EventHandlers 可以将一个对象的事件映射到相同或不同类型的对象的多个 reconcile.Request 。例如，将一个 Node 事件映射到响应集群大小事件的对象上。

* 在几乎所有情况下，用户都应该只使用所提供的 EventHandler 实现，而不是实现自己的。

### Predicate

predicate.Predicate 是 Controller.Watch 的一个可选的参数，用于过滤事件。这使得常见的过滤器可以被重复使用和组合。

* Predicate 接收一个事件，并返回bool（ true为enqueue ）。

* Predicate 是可选的参数

* 用户应该使用所提供的 Predicate 实现，但可以实现额外的 Predicate，例如：generation 改变，标签选择器改变等。

### PodController 图例

Source 提供事件：

* `&source.KindSource{&v1.Pod{}} -> (Pod foo/bar Create Event)`

EventHandlers 入队请求：

* `&handler.EnqueueRequestForObject{} -> (reconcile.Request{types.NamespaceName{Name: "foo", Namespace: "bar"}}) `

Reconciler 被调用，携带参数Request：

* `Reconciler(reconcile.Request{types.NamespaceName{Name: "foo", Namespace: "bar"}})`



### 用法

下面的例子显示了创建一个新的 Controller 程序，该程序响应 Pod 或 ReplicaSet 事件，对 ReplicaSet 对象进行 Reconcile。Reconciler 函数只是给 ReplicaSet 添加一个标签。

参见 `examples/builtins/main.go` 中的使用实例。

Controller 实例：

- 1. Watch ReplicaSet 和 Pods Source

  * 1.1 `ReplicaSet -> handler.EnqueueRequestForObject` -- 用 ReplicaSet 的命名空间和名称来入队请求。

  * 1.2 `Pod (由 ReplicaSet 创建) -> handler.EnqueueRequestForOwnerHandler` --以拥有的 ReplicaSet 命名空间和名称来入队Request。

- 2. 响应事件，对 ReplicaSet 进行 Reconcile

  * 2.1 创建 ReplicaSet 对象 -> 读取 ReplicaSet，尝试读取 Pods -> 如果空缺就创建 Pods。

  * 2.2 创建 Pods 锁触发的 Reconciler -> 读取 ReplicaSet 和 Pods，什么都不做。

  * 2.3 从其他角色中刪除Pods而触发Reconciler -> 读取 ReplicaSet 和 Pods，创建替代Pods。

### 监视和事件处理

Controller 可以观察多种类型的对象（如Pods、ReplicaSets和 deployment），但他们只 reconcile 一个类型。当一种类型的对象必须更新以响应另一种类型的对象的变化时，`EnqueueRequestsFromMapFunc` 可用于将事件从一种类型映射到另一种类型。例如，通过重新 reconcile 某些API的所有实例来响应集群大小调整事件（添加/删除节点）。

Deployment Controller 可能使用 `EnqueueRequestForObject` 和 `EnqueueRequestForOwner` 来：

* 观察 Deployment 事件--入队 Deployment 的命名空间和名称。

* 观察 ReplicaSet 事件-- 入队创建 ReplicaSet 的 Deployment 的命名空间和名称（例如，所有者/Owner）。

注意：reconcile.Requests 在入队的时候会被去重。同一个 ReplicaSet 的许多 Pod 事件可能只触发1个 reconcile 调用，因为每个事件都会导致 handler 试图为 ReplicaSet 入队相同的 reconcile.Request。

### Controller的编写技巧

Reconciler 运行时的复杂性：

* 最好是编写 Controllers 来执行N次O(1) reconcile（例如对N个不同的对象），而不是执行1次O(N) reconcile（例如对一个管理着N个其他对象的单一对象）。

* 例子： 如果你需要更新所有服务以响应一个节点的添加-- reconcile 服务但观察节点（转变为服务对象的名称/命名空间），而不是 reconcile 节点并更新服务。

事件复用：

* 对同一名称/命名空间的 reconcile 请求在入队时被批量和去重处理。这使 Controller 能够优雅地处理单个对象的大量事件。将多个事件源复用到一个对象类型上，将对不同对象类型的事件进行批量请求。

* 例如： ReplicaSet 的 Pod 事件被转换为 ReplicaSet 名称/命名空间，因此 ReplicaSet 将只对来自多个 Pod 的多个事件进行1次Reconciled。



### 目录

|                                                              |                                                              |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [builder](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/builder) | Package builder 封装了其他 controller-runtime 库，并公开了构建普通 controller 的简单模式。 |
| [cache](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/cache) | Package cache provides object caches that act as caching client.Reader instances and help drive Kubernetes-object-based event handlers. 包缓存提供对象缓存，作为缓存客户端.阅读器实例，帮助驱动基于Kubernetes对象的事件处理程序。 |
| [certwatcher](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/certwatcher) | Package certwatcher is a helper for reloading Certificates from disk to be used with tls servers. |
| [client](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/client) | Package client contains functionality for interacting with Kubernetes API servers. |
| [cluster](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/cluster) |                                                              |
| [config](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/config) | Package config contains functionality for interacting with configuration for controller-runtime components. |
| [controller](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/controller) | Package controller provides types and functions for building Controllers. |
| [conversion](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/conversion) | Package conversion provides interface definitions that an API Type needs to implement for it to be supported by the generic conversion webhook handler defined under pkg/webhook/conversion. |
| [envtest](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/envtest) | Package envtest provides libraries for integration testing by starting a local control plane |
| [event](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/event) | Package event contains the definitions for the Event types produced by source.Sources and transformed into reconcile.Requests by handler.EventHandler. |
| [finalizer](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/finalizer) |                                                              |
| [handler](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/handler) | Package handler defines EventHandlers that enqueue reconcile.Requests in response to Create, Update, Deletion Events observed from Watching Kubernetes APIs. |
| [healthz](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/healthz) | Package healthz contains helpers from supporting liveness and readiness endpoints. |
| [leaderelection](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/leaderelection) | Package leaderelection contains a constructor for a leader election resource lock. |
| [log](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/log) | Package log contains utilities for fetching a new logger when one is not already available. |
| [manager](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/manager) | Package manager is required to create Controllers and provides shared dependencies such as clients, caches, schemes, etc. |
| [metrics](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/metrics) | Package metrics contains controller related metrics utilities |
| [predicate](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/predicate) | Package predicate defines Predicates used by Controllers to filter Events before they are provided to EventHandlers. |
| [ratelimiter](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/ratelimiter) | Package ratelimiter defines rate limiters used by Controllers to limit how frequently requests may be queued. |
| [reconcile](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/reconcile) | Package reconcile defines the Reconciler interface to implement Kubernetes APIs. |
| [recorder](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/recorder) | Package recorder defines interfaces for working with Kubernetes event recorders. |
| [scheme](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/scheme) | Package scheme contains utilities for gradually building Schemes, which contain information associating Go types with Kubernetes groups, versions, and kinds. |
| [source](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/source) | Package source provides event streams to hook up to Controllers with Controller.Watch. |
| [webhook](https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.15.0/pkg/webhook) | Package webhook provides methods to build and bootstrap a webhook server. |
