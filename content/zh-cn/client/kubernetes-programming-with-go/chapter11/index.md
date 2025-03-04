---
title: "[第11章]编写 Reconcile 循环"
linkTitle: "编写 Reconcile 循环"
weight: 110
date: 2021-02-01
description: >
  编写 Reconcile 循环
---

在上一章中，我们已经看到了如何使用 controller-runtime 库来以引导一个新的项目来编写 Operator。在本章中，我们将重点讨论 Reconcile 函数的实现，它是实现 Operator 的重要部分。

Reconcile 函数包含 Operator 的所有业务逻辑。该函数将在一个单一的资源种类上工作 -- 或者说 Operator 调和这个资源 -- 并且可以在其他类型的对象触发事件时得到通知，通过使用所有者引用（*Owner References*）将这些其他类型映射到调和的对象上。

**Reconcile** 函数的作用是确保系统的状态与要 Reconcile 的资源中指定的内容相匹配。为此，它将创建 "低级" 资源来实现要调和的资源。这些资源反过来将由其他控制器或 Operator 进行调和。当这些资源的调和完成后，它们的状态将被更新以反映它们的新状态。此外，Operator 将能够检测到这些变化，并相应地调整被调和资源的状态。

举个例子，您在前一章开始实施的 Operator 调和了自定义资源 MyResource。Operator 将为实现 MyResource 实例创建 Deployment 实例，为此， Operator 也要观察 Deployment 资源。

当用于 Deployment 实例的事件被触发时，为其创建 Deployment 的 MyResource 实例将被调和。例如，在 Deployment 的 Pod 被创建后，Deployment 的状态将被更新，以表明所有副本都在运行。在这一点上，Operator 可以修改调和资源的状态，以表明它已准备就绪。

Reconcile 循环被称为观察资源的过程，当资源被创建、修改或删除时，调用 Reconcile 函数，用低级别的资源实现调和的资源，观察这些资源的状态，并相应地更新调和资源的状态。

## 编写 Reconcile 函数

Reconcile 函数从队列中接收一个要调和的资源。第一个要做的操作是获得关于这个资源的信息来进行调和。

事实上，只收到了资源的命名空间和名称（它的种类/kind 是由它的设计知道的，是由 operator 调和的种类），但你并没有收到资源的完整定义。

**Get** 操作被用来获取资源的定义。

### 检查资源是否存在

该资源可能由于各种原因被入队：它被创建、修改或删除（或者另一个拥有（owned）的资源被创建、修改或删除）。在前两种情况下（创建或修改），Get 操作将成功，Reconcile 函数将知道这时资源的定义。在删除的情况下，获取操作将以 Notfound 错误失败，因为该资源现在已经被删除了。

Operator 的良好做法是，在调和资源时为其创建的资源添加 OwnerReferences。主要目的是当这些创建的资源被修改时，能够调和其所有者，而添加这些 OwnerReferences 的结果是，当所有者资源被删除时，这些拥有（owned）的资源将被 Kubernetes 垃圾收集器删除。

出于这个原因，当一个被调和的对象被删除时，集群中没有什么可做的，因为相关的创建资源的删除将由集群处理。

### 实现调和的资源

如果在集群中找到了要调和的资源，operator’s 的下一步就是创建 "低级" 资源来实现这个要调和的资源。

因为 Kubernetes 是一个声明式平台，创建这些资源的好方法是声明低级资源应该是什么，与集群中存在或不存在什么无关，并依靠 Kubernetes 控制器来接管和调和这些低级资源。

由于这个原因，不可能盲目地使用 **Create** 方法，因为我们不确定资源是否存在，如果资源已经存在，操作就会失败。

你可以检查资源是否存在，如果不存在则创建，如果存在则修改。正如前几章所示，服务器端应用方法非常适合这种情况：在运行 Apply 操作时，如果资源不存在，就会被创建；如果资源存在，就会被修补，在资源被其他参与者修改的情况下解决冲突。

使用服务器端的 apply 方法，Operator 不需要检查资源是否存在，或是否被另一个参与者修改过。Operator 只需要从 Operator 的角度，用资源的定义运行服务器端 Apply 操作。

在应用低级别的资源后，应该考虑两种可能性。

- 情况1：如果低层资源已经存在，并且没有被 Apply 修改，那么将不会为这些资源触发 MODIFIED 事件，并且 Reconcile 函数将不会被再次调用（至少对于这个 Apply 操作）。
- 情况2：如果低层资源被创建或修改，这将触发这些资源的 CREATED 或 MODIFIED 事件，并且 Reconcile 函数将因为这些事件而被再次调用。这个函数的新执行将再次应用低层资源，如果在此期间没有更新这些资源，Operator 将落入情况1。


新的底层资源最终将由其各自的 Operator 或控制器处理。反过来，他们将调和这些资源，并更新它们的状态，宣布它们的当前状态。

一旦这些低层资源的状态被更新，这些资源的 MODIFIED 事件将被触发，Reconcile 函数将被再次调用。再一次，Operator 将应用这些资源，案例1和2必须被考虑。

Operator 在某些时候需要读取它所创建的低级资源的状态，以便计算出调和后的资源的状态。在简单的情况下，这可以在执行低级资源的服务器端应用后完成。

### 简单的实现例子

为了说明这一点，这里有一个完整的 Reconcile 函数，该函数适用于 Operator，该 Operator 用 MyResource 实例中提供的镜像和内存信息创建 Deployment。

```go
func (a *MyReconciler) Reconcile(
  ctx context.Context,
  req reconcile.Request,
) (reconcile.Result, error) {
  log := log.FromContext(ctx)
  log.Info("getting myresource instance")
  myresource := mygroupv1alpha1.MyResource{}
  err := a.client.Get(                                     ❶
    ctx,
    req.NamespacedName,
    &myresource,
    &client.GetOptions{},
  )
  if err != nil {
    if errors.IsNotFound(err) {                            ❷
      log.Info("resource is not found")
      return reconcile.Result{}, nil
    }
    return reconcile.Result{}, err
  }
  ownerReference := metav1.NewControllerRef(               ❸
    &myresource,
    mygroupv1alpha1.SchemeGroupVersion.
          WithKind("MyResource"),
  )
  err = a.applyDeployment(                                 ❹
     ctx,
     &myresource,
     ownerReference,
  )
  if err != nil {
    return reconcile.Result{}, err
  }
  status, err := a.computeStatus(ctx, &myresource)          ❺
  if err != nil {
    return reconcile.Result{}, err
  }
  myresource.Status = *status
  log.Info("updating status", "state", status.State)
  err = a.client.Status().Update(ctx, &myresource)           ❻
  if err != nil {
    return reconcile.Result{}, err
  }
  return reconcile.Result{}, nil
}
```

- ❶ 获取要调和的资源的定义

- ❷ 如果资源不存在，立即返回

- ❸ 建立指向要调和的资源的 ownerReference

- ❹ 使用服务器端应用于 "低层次" deployment

- ❺ 根据 "低级别" deployment 计算资源的状态

- ❻ 更新资源的状态以进行调和

下面是一个为 operator 创建的 deployment 实现服务器端 Apply 操作的例子：

```go
func (a *MyReconciler) applyDeployment(
  ctx context.Context,
  myres *mygroupv1alpha1.MyResource,
  ownerref *metav1.OwnerReference,
) error {
  deploy := createDeployment(myres, ownerref)
  err := a.client.Patch(                                   ❼
    ctx,
    deploy,
    client.Apply,
    client.FieldOwner(Name),
    client.ForceOwnership,
  )
  return err
}

func createDeployment(
  myres *mygroupv1alpha1.MyResource,
  ownerref *metav1.OwnerReference,
) *appsv1.Deployment {
  deploy := &appsv1.Deployment{
    ObjectMeta: metav1.ObjectMeta{
      Labels: map[string]string{
        "myresource": myres.GetName(),
      },
    },
    Spec: appsv1.DeploymentSpec{
      Selector: &metav1.LabelSelector{
        MatchLabels: map[string]string{
          "myresource": myres.GetName(),
        },
      },
      Template: corev1.PodTemplateSpec{
        ObjectMeta: metav1.ObjectMeta{
          Labels: map[string]string{
            "myresource": myres.GetName(),
          },
        },
        Spec: corev1.PodSpec{
          Containers: []corev1.Container{
            {
              Name:  "main",
              Image: myres.Spec.Image,                    ❽
              Resources: corev1.ResourceRequirements{
                Requests: corev1.ResourceList{
                  corev1.ResourceMemory:      myres.Spec.Memory,          ❾
                },
              },
            },
          },
        },
      },
    },
  }
  deploy.SetName(myres.GetName() + "-deployment")
  deploy.SetNamespace(myres.GetNamespace())
  deploy.SetGroupVersionKind(
    appsv1.SchemeGroupVersion.WithKind("Deployment"),
  )
  deploy.SetOwnerReferences([]metav1.OwnerReference{     ❿
    *ownerref,
  })
  return deploy
}
```

❼ 使用补丁方法执行服务器端的应用操作

❾ 使用资源中定义的镜像来进行调和

❾ 使用资源中定义的内存来进行调和

❿ 设置 OwnerReference 指向要调和的资源

然后，下面是一个如何实现状态的计算和更新的例子：

```go
const (
  _buildingState = "Building"
  _readyState    = "Ready"
)
func (a *MyReconciler) computeStatus(
  ctx context.Context,
  myres *mygroupv1alpha1.MyResource,
) (*mygroupv1alpha1.MyResourceStatus, error) {
  logger := log.FromContext(ctx)
  result := mygroupv1alpha1.MyResourceStatus{
    State: _buildingState,
  }
  deployList := appsv1.DeploymentList{}
  err := a.client.List(                                   ⓫
    ctx,
    &deployList,
    client.InNamespace(myres.GetNamespace()),
    client.MatchingLabels{
      "myresource": myres.GetName(),
    },
  )
  if err != nil {
    return nil, err
  }
  if len(deployList.Items) == 0 {
    logger.Info("no deployment found")
    return &result, nil
  }
  if len(deployList.Items) > 1 {
    logger.Info(
            "too many deployments found", "count",
      len(deployList.Items),
)
    return nil, fmt.Errorf(
               "%d deployment found, expected 1",
      len(deployList.Items),
)
  }
  status := deployList.Items[0].Status                     ⓬
  logger.Info(
          "got deployment status",
          "status", status,
)
  if status.ReadyReplicas == 1 {
    result.State = _readyState                              ⓭
  }
  return &result, nil
}
```

- ⓫ 获取为该资源创建的 deployment 以进行调和

- ⓬ 获取找到的唯一 deployment 的状态

- ⓭ 当 replicas 为1时，为调和的资源设置就绪状态


## 总结

本章展示了如何为一个简单的 operator 实现 Reconcile 功能，该 operator 使用来自自定义资源的信息创建 deployment。

真正的 operator 通常会比这个简单的例子更复杂，他们会创建几个具有更复杂生命周期的资源，但它展示了开始编写 operator 时需要知道的要点： Reconcile 循环的声明性，所有者引用的使用，服务器端应用的使用，以及状态的更新方式。

下一章将展示如何测试 Reconcile 循环。
