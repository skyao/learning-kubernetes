---
title: "事件"
linkTitle: "事件"
weight: 70
date: 2021-02-01
description: >
  事件
---


Kubernetes API提供了一个 事件/Event 资源，事件实例被附加到任何类型的特定实例。事件由控制器发送，以通知用户与某个对象相关的一些事件发生。在执行`kubectl describe` 时，会显示这些事件。例如，你可以看到与一个 pod 执行相关的事件：

```bash
$ kubectl describe pods nginx
[...]
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  1s         default-scheduler  Successfully assigned...
  Normal  Pulling    0s         kubelet            Pulling image "nginx"
  Normal  Pulled     <invalid>  kubelet            Successfully pulled...
  Normal  Created    <invalid>  kubelet            Created container nginx
  Normal  Started    <invalid>  kubelet            Started container nginx
```

为了从 Reconcile 函数中发送此类事件，你需要访问由管理器提供的 EventRecorder 实例。在初始化过程中，你可以用管理器上的 GetEventRecorderFor 方法获得这个实例，然后在构建 Reconcile 结构体时传递这个值：

```go
type MyReconciler struct {
      client        client.Client
      EventRecorder record.EventRecorder
}
func main() {
      [...]
      eventRecorder := mgr.GetEventRecorderFor(
"MyResource",
     )
      err = builder.
            ControllerManagedBy(mgr).
            Named(controller.Name).
            For(&mygroupv1alpha1.MyResource{}).
            Owns(&appsv1.Deployment{}).
            Complete(&controller.MyReconciler{
                  EventRecorder: eventRecorder,
            })
      [...]
}
```

然后，从 Reconcile 函数中，你可以调用 Event、Eventf 和 AnnotatedEventf 方法。

```go
func (record.EventRecorder) Event(
     object runtime.Object,
     eventtype, reason, message string,
)
func (record.EventRecorder) Eventf(
     object runtime.Object,
     eventtype, reason, messageFmt string,
     args ...interface{},
)
func (record.EventRecorder) AnnotatedEventf(
     object runtime.Object,
     annotations map[string]string,
     eventtype, reason, messageFmt string,
     args ...interface{},
)
```

**object** 参数表示将事件附加到哪个对象。你将传递被调和的自定义资源。

**eventtype** 参数接受两个值：`corev1.EventTypeNormal` 和 `corev1.EventTypeWarning`。

**reason** 参数是一个 UpperCamelCase 格式的短值。**message** 参数是一个人类可读的文本。

**Event** 方法用于传递静态消息，Eventf 方法可以使用 Sprintf 创建一个消息，AnnotatedEventf 方法也可以给事件附加注释。

## 结论

在本章中，你已经使用 controller-runtime 库开始创建 operator。你已经看到了如何使用适当的 schema 创建 manager，如何创建控制器并声明 Reconcile 函数，并探索了库所提供的客户端的各种功能来访问集群中的资源。

下一章将探讨如何编写 Reconcile 函数。
