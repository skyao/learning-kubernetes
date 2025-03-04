---
title: "Sidecar Container概述"
linkTitle: "概述"
weight: 1
date: 2021-02-01
description: >
  Kubernetes Sidecar Container概述
---



**From Kubernetes 1.18 containers can be marked as sidecars**



Unfortunately, that features has been removed from 1.18, then removed from 1.19 and currently has no specific date for landing.

reference: [kubernetes/enhancements#753](https://github.com/kubernetes/enhancements/issues/753)



## 资料

### 官方正式资料

- [k8s pod概念](https://kubernetes.io/docs/concepts/workloads/pods/)

- [Sidecar Containers（kubernetes/enhancements#753）](https://github.com/kubernetes/enhancements/issues/753): 最权威的资料了，准备细读
- [Support startup dependencies between containers on the same Pod](https://github.com/kubernetes/kubernetes/issues/65502)

### 社区介绍资料

- [Sidecar Containers improvement in Kubernetes 1.18](https://medium.com/@chan_seeker/sidecar-containers-improvement-in-kubernetes-1-18-b5eb66ee2b83): 重点阅读
- [Kubernetes — Learn Sidecar Container Pattern](https://medium.com/bb-tutorials-and-thoughts/kubernetes-learn-sidecar-container-pattern-6d8c21f873d)
- [Sidecar container lifecycle changes in Kubernetes 1.18](https://banzaicloud.com/blog/k8s-sidecars/)
- [Tutorial: Apply the Sidecar Pattern to Deploy Redis in Kubernetes](https://thenewstack.io/tutorial-apply-the-sidecar-pattern-to-deploy-redis-in-kubernetes/)
- [Sidecar Containers](https://imroc.io/learning-kubernetes/kep/sidecar-containers.html)：by 陈鹏，特别鸣谢

## 相关项目的处理

### Istio

#### 信息1

https://github.com/kubernetes/enhancements/issues/753#issuecomment-684176649

We use a custom daemon image like a `supervisor` to wrap the user's program. The daemon will also listen to a particular port to convey the health status of users' programs (exited or not).

> 我们使用一个类似`supervisor`的自定义守护进程镜像来包装用户的程序。守护进程也会监听特定的端口来传达用户程序的健康状态（是否退出）。

Here is the workaround:

- Using the daemon image as `initContainers` to copy the binary to a shared volume.
- Our `CD` will hijack users' command, let the daemon start first. Then, the daemon runs the users' program until Envoy is ready.
- Also, we add `preStop`, a script that keeps checking the daemon's health status, for Envoy.

> 下面是变通的方法：
> 
> - 以 "initContainers" 的方式用守护进程的镜像来复制二进制文件到共享卷。
> - 我们的 `CD` 会劫持用户的命令，让守护进程先启动，然后，守护进程运行用户的程序，直到 Envoy 准备好。
> - 同时，我们还为Envoy添加 `preStop`，一个不断检查守护进程健康状态的脚本。

As a result, the users' process will start if Envoy is ready, and Envoy will stop after the process of users is exited.

> 结果，如果Envoy准备好了，用户的程序就会启动，而Envoy会在用户的程序退出后停止。

It's a complicated workaround, but it works fine in our production environment.

> 这是一个复杂的变通方法，但在我们的生产环境中运行良好。

#### 信息2

还找到一个答复： https://github.com/kubernetes/enhancements/issues/753#issuecomment-687184232

[Allow users to delay application start until proxy is ready](https://github.com/istio/istio/pull/24737)

for startup issues, the istio community came up with a quite clever workaround which basically injects envoy as the first container in the container list and adds a postStart hook that checks and wait for envoy to be ready. This is blocking and the other containers are not started making sure envoy is there and ready before starting the app container.

> 对于启动问题，istio社区想出了一个相当聪明的变通方法，基本上是将envoy作为容器列表中的第一个容器注入，并添加一个postStart钩子，检查并等待envoy准备好。这是阻塞的，而其他容器不会启动，这样确保envoy启动并且准备好之后，然后再启动应用程序容器。

We had to port this to the version we're running but is quite straightforward and are happy with the results so far.

> 我们已经将其移植到我们正在运行的版本中，很直接，目前对结果很满意。

For shutdown we are also 'solving' with preStop hook but adding an arbitrary sleep which we hope the application would have gracefully shutdown before continue with SIGTERM.

> 对于关机，我们也用 preStop 钩子来 "解决"，但增加了一个任意的 sleep，我们希望应用程序在继续 SIGTERM 之前能优雅地关机。

相关issue： [Enable holdApplicationUntilProxyStarts at pod level](https://github.com/istio/istio/issues/26923)

### Knative

### dapr

- [Clarify lifecycle of Dapr process and app process ](https://github.com/dapr/dapr/issues/2007): dapr项目中在等待 sidecar container的结果。在此之前，dapr做了一个简单的调整，将daprd这个sidecar的启动顺序放在最前面（详见 https://github.com/dapr/dapr/pull/2341）