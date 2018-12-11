---
date: 2018-12-08T09:00:00+08:00
title: Controller
weight: 320
description : "Kubernetes的Controller概念"
---

备注： 内容来自 [Pod Overview](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)

### Pod 和 Controller

Controller可以为您创建和管理多个Pod，处理副本和部署，并在集群范围内提供自我修复功能。例如，如果节点发生故障，Controller可能会通过在不同节点上安排相同的替换品来自动替换Pod。

通常，Controller使用您提供的Pod模板来创建它负责的Pod。

## Pod 模板

Pod 模板是 pod 规范，包含在其他对象中，例如 [Replication Controllers](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)，[Jobs](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/)和 [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)。控制器使用Pod模板生成实际的pod。下面的示例是Pod的简单清单，其中包含一个打印消息的容器。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']
```

pod模板不是指定当前所有副本的所需状态，而是像cookie切割器那样。切割cookie后，cookie与切割器无关。没有“量子纠缠”。对模板的后续更改甚至切换到新模板对已创建的pod没有直接影响。类似地，由副本控制器创建的pod随后可以直接更新。这与pod有意对比，pod确实指定了属于pod的所有容器的当前所需状态。这种方法从根本上简化了系统语义并增加了原语的灵活性。





