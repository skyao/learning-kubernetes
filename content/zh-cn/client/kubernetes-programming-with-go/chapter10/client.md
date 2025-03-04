---
title: "使用客户端"
linkTitle: "使用客户端"
weight: 50
date: 2021-02-01
description: >
  使用客户端
---

客户端可以用来读取和写入集群上的资源，并更新资源的状态。

**Read** 方法在内部使用一个基于 Informers 和 Listers 的 Cache 系统，以限制对 API 服务器的读取访问。使用这个缓存，同一个管理器的所有控制器都有对资源的读取权限，同时限制对 API 服务器的请求。

> 必须注意:
>
> 由读操作返回的对象是指向缓存的值的指针。你决不能直接修改这些对象。相反，你必须在修改这些对象之前创建一个返回对象的深度拷贝。

客户端的方法是通用的：它们与任何 Kubernetes 资源一起工作，无论是原生资源还是自定义资源，如果它们被传递给管理器的 schema 所知道的话。

所有的方法都会返回错误，其类型与 Client-go Clientset 方法所返回的错误相同。你可以参考第6章的 "错误和状态" 部分以了解更多信息。

## 获取资源的信息

Get 方法用来获取资源的信息。

```go
Get(
     ctx context.Context,
     key ObjectKey,
     obj Object,
     opts ...GetOption,
) error
```

它需要一个 ObjectKey 值作为参数，表示资源的名称空间和名称，以及一个Object，表示要获得的资源的类型，并存储结果。Object 必须是一个指向类型资源的指针-例如，一个 Pod 或 MyResource 结构体。ObjectKey 类型是 `type.NamespacedName` 的别名，定义在 API Machinery 库中。

NamespacedName 也是嵌入在作为参数传递给 Reconcile 函数的 Request 中的对象的类型。你可以直接将 req.NamespacedName 作为 ObjectKey 传递，以获得要调和（reconcile）的资源。例如，使用下面的方法来获取要调和（reconcile）的资源：

```go
myresource := mygroupv1alpha1.MyResource{}
err := a.client.Get(
      ctx,
      req.NamespacedName,
      &myresource,
)
```

可以向 Get 请求传递一个特定的 `resourceVersion` 值，传递一个 `client.GetOptions` 结构体实例作为最后一个参数。

GetOptions 结构体实现了 GetOption 接口，包含一个具有 `metav1.GetOptions` 值的单一Raw字段。例如，指定一个值为 "0 " 的 resourceVersion 来获取资源的任何版本：

```go
err := a.client.Get(
      ctx,
      req.NamespacedName,
      &myresource,
      &client.GetOptions{
            Raw: &metav1.GetOptions{
                  ResourceVersion: "0",
            },
      },
)
```

## 列出资源

List 方法用于列出特定种类的资源：

```go
List(
     ctx context.Context,
     list ObjectList,
     opts ...ListOption,
) error
```

list 参数是一个 ObjectList 值，表示要列出并存储结果的资源的种类（kind）。默认情况下，list 是在所有命名空间中进行的。

List 方法接受实现 ListOption 接口的对象的零个或多个参数。这些类型由以下支持：

- **InNamespace**，string 的别名，用于返回特定命名空间的资源。

- **MatchingLabels**， `map[string]string` 的别名，用来表示标签的列表和它们的精确值，这些标签必须被定义为资源的返回。下面的例子建立了一个MatchingLabels 结构体来过滤带有标签 "app=myapp" 的资源。

  ```go
  matchLabel := client.MatchingLabels{
        "app": "myapp",
  }
  ```

- **HasLabels**，别名为 `[]string`，用来表示标签的列表，独立于它们的值，必须为一个资源的返回而定义。下面的例子建立了一个 HasLabels 结构体来过滤带有 "app" 和 "debug" 标签的资源。

  ```go
  hasLabels := client.HasLabels{"app", “debug”}
  ```

- **MatchingLabelsSelector**，嵌入了一个 `labels.Selector` 接口，用来传递更高级的标签选择器。关于如何建立一个选择器的更多信息，请参见第6章的 "过滤列表结果" 部分。下面的例子建立了一个 MatchingLabelsSelector 结构，它可以作为 List 的一个选项来过滤标签 mykey 不同于 **ignore** 的资源。

  ```go
  selector := labels.NewSelector()
  require, err := labels.NewRequirement(
      "mykey",
      selection.NotEquals,
      []string{"ignore"},
  )
  // assert err is nil
  selector = selector.Add(*require)
  labSelOption := client.MatchingLabelsSelector{
        Selector: selector,
  }
  ```

- MatchingFields 是 `fields.Set` 的别名，它本身是 `map[string]string` 的别名，用来指示要匹配的字段和它们的值。下面的例子建立了一个 MatchingFields 结构体，用来过滤字段 "status.phase" 为 "Running" 的资源：

  ```go
  matchFields := client.MatchingFields{
        "status.phase": "Running",
  }
  ```

- MatchingFieldsSelector，嵌入了一个 `fields.Selector`，用来传递更高级的字段选择器。关于如何构建 `fields.Selector` 的更多信息，请参见第6章的 "过滤列表结果" 部分。下面的例子建立了一个 MatchingFieldsSelector 结构体来过滤字段 "status.phase" 与 "Running" 不同的资源：

  ```go
  fieldSel := fields.OneTermNotEqualSelector(
        "status.phase",
        "Running",
  )
  fieldSelector := client.MatchingFieldsSelector{
        Selector: fieldSel,
  }
  ```

- Limit（别名 int64）和Continue（别名 string）用于对结果进行分页。这些选项在第二章的 "分页结果" 部分有详细介绍。

## 创建资源

Create 方法用来在集群中创建一个新的资源。

```go
Create(
     ctx context.Context,
     obj Object,
     opts ...CreateOption,
) error
```

作为参数传递的 obj 定义了要创建的对象的种类（kind），以及它的定义。下面的例子将在集群中创建一个 Pod：

```go
podToCreate := corev1.Pod{ [...] }
podToCreate.SetName("nginx")
podToCreate.SetNamespace("default")
err = a.client.Create(ctx, &podToCreate)
```

以下选项可以作为 CreateOption 传递，以使 Create 请求参数化。

- DryRunAll 值表示所有的操作都应该被执行，除了那些将资源持久化到存储的操作。

- FieldOwner，别名为字符串，表示创建操作的字段管理器的名称。这个信息对于服务器端应用操作的正常工作很有用。

## 删除资源

删除方法是用来从集群中删除资源。

```go
Delete(
     ctx context.Context,
     obj Object, k
     opts ...DeleteOption,
) error
```

作为参数传递的 obj 定义了要删除的对象的种类（kind），以及它的命名空间（如果资源有命名空间）和它的名字。下面的例子可以用来删除 Pod。

```go
podToDelete := corev1.Pod{}
podToDelete.SetName("nginx")
podToDelete.SetNamespace("prj2")
err = a.client.Delete(ctx, &podToDelete)
```

以下选项可以作为 DeleteOption 被传递，以便对 Delete 请求进行参数化。

- DryRunAll - 这个值表示所有的操作都应该被执行，除了那些将资源持久化到存储的操作。
- **GracePeriodSeconds**, alias to int64 - 这个值只在删除 pod 时有用。它表示在删除 pod 之前的持续时间（秒）。更多细节见第6章的 "删除资源" 部分。
- **Preconditions** / 前提条件，别名 `metav1.Preconditions` - 这表明你期望删除的资源。更多细节请参见第6章 "删除资源" 部分。
- **PropagationPolicy**，别名 `metav1.DeletionPropagation` - 这表明是否以及如何进行垃圾回收。更多细节见第6章 "删除资源" 部分。

## 删除资源集合

DeleteAllOf 方法是用来从集群中删除所有给定类型的资源。

```go
DeleteAllOf(
     ctx context.Context,
     obj Object,
     opts ...DeleteAllOfOption,
) error
```

作为参数传递的 obj 定义了要删除的对象的种类。

DeleteAllOf 操作的 **opts** 可选项是 List 操作（见 "列出资源" 部分）和 Delete 操作（见 "删除资源" 部分）的选项的组合。

作为例子，下面是如何删除给定命名空间的所有 deployment：

```go
err = a.client.DeleteAllOf(
     ctx,
     &appsv1.Deployment{},
     client.InNamespace(aNamespace))
```

## 更新资源

TBD

## 修补资源

### Strategic Merge Patch

### Merge Patch

TBD

## 更新资源的状态

在和 Operator 工作时，您要在自定义资源的 status 部分修改数值，以表明它的当前状态。

```go
Update(
     ctx context.Context,
     obj Object,
     opts ...UpdateOption,
) error
```

请注意，您不需要为状态进行特定的创建、获取或删除操作，因为您将在创建自定义资源本身时将状态创建为自定义资源的一部分。另外，资源的 Get 操作将返回资源的 Status 部分，删除资源将删除其 Status 部分。

然而，API 服务器强迫你对 Status 和资源的其他部分使用不同的 Update 方法，以保护你不会在同一时间错误地修改这两部分。

为了使状态的更新发挥作用，自定义资源定义必须在自定义资源的子资源列表中声明 **status** 字段。参见第8章的 "资源版本的定义" 部分，了解如何声明这个状态子资源。

这个方法的工作原理和资源本身的 Update 方法一样，但是要调用这个方法，你需要在 client.Status() 返回的对象上调用这个方法：

```go
err = client.Status().Update(ctx, obj)
```

## 修补资源的状态

与更新方法一样，修补资源的状态需要一个专门的方法来处理 `client.Status()` 的结果。

```go
Patch(
    ctx context.Context,
    obj Object,
    patch Patch,
    opts ...PatchOption,
) error
```

对 Status 的 Patch 方法与对资源本身的 Patch 方法工作相同，只是它只对资源的 Status 部分进行修补。

```go
err = client.Status().Patch(ctx, obj, patch)
```

