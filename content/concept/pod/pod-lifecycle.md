---
date: 2018-12-08T09:00:00+08:00
title: Pod生命周期
menu:
  main:
    parent: "concept-pod"
weight: 312
description : "Kubernetes pod的生命周期"
---

> 备注： 内容来自 [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)

## Pod phase

Pod的status字段是 PodStatus 对象，它有一个phase字段。

Pod的phase是Pod在其生命周期中的简单而高级的摘要。phase不是对Container或Pod状态的全面观察汇总，也不是一个综合状态机。

Pod phase值的数量和含义受到严密保护。除了这里记录的内容之外，没有任何关于具有给定phase值的Pod的假设。

以下是可能的值phase：

值 | 描述 
:-----|:-----------
`Pending` | Pod已被Kubernetes系统接受，但尚未创建一个或多个Container镜像。这包括计划之前的时间以及通过网络下载镜像所花费的时间，这可能需要一段时间。补充：1. 可能处在写数据到etcd，调度，pull镜像，启动容器这四个阶段中的任何一个阶段。2. pending伴随的事件通常会有：ADDED, Modified 
`Running` |  Pod已绑定到节点，并且已创建所有Container。至少有一个Container仍在运行，或者正在启动或重新启动。
`Succeeded` | Pod中的所有容器都已成功终止，并且不会重新启动。补充：一般部署 job 时出现。 
`Failed` |  Pod中的所有容器都已终止，并且至少有一个Container已终止失败。也就是说，Container要么以非零状态退出，要么被系统终止。
`Unknown` |  由于某种原因，无法获得Pod的状态，这通常是由于与Pod的主机通信时出错。

## Pod条件

Pod有一个PodStatus，它有一个[PodConditions](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.13/#podcondition-v1-core) 数组， Pod已经或没有通过它。PodCondition 数组的每个元素都有六个可能的字段：

- `lastProbeTime`字段提供上次探测Pod条件的时间戳。
- `lastTransitionTime`字段提供Pod最后从一个状态转换到另一个状态的时间戳。
- `message`字段是人类可读的消息，指示有关转换的详细信息。
- `reason`字段是该条件最后一次转换的唯一原因，单字，CamelCase。
- `status`字段是一个字符串，可能的值为“ `True`”，“ `False`”和“ `Unknown`”。
- `type`字段是一个包含以下可能值的字符串：
	- `PodScheduled`：Pod已被安排到一个节点;
	- `Ready`：Pod能够提供请求，应该添加到所有匹配服务的负载均衡池中;
	- `Initialized`：所有 [init容器](https://kubernetes.io/docs/concepts/workloads/pods/init-containers) 都已成功启动;
	- `Unschedulable`：调度程序现在无法调度Pod，例如由于缺少资源或其他限制;
	- `ContainersReady`：Pod中的所有容器都已准备就绪。

## 容器探针

探针（Probe）是通过 kubelet 周期性地执行 Container 的诊断。为了执行诊断，kubelet调用Container实现的 Handler。Handler有三种类型：

* ExecAction
  在Container内执行指定的命令。如果命令以状态代码0退出，则认为诊断成功。

* TCPSocketAction:
  对Container的IP地址执行指定端口的TCP检查。如果端口打开，则诊断被认为是成功的。

* HTTPGetAction:
  对指定端口和路径上的Container的IP地址执行HTTP Get请求。如果响应的状态代码大于或等于200且小于400，则认为诊断成功。

每个探针都有三个结果之一：

* Success: Container通过了诊断。
* Failure: 容器未通过诊断。
* Unknown: 诊断失败，因此不应采取任何措施。

kubelet可以选择在运行中的容器上执行两种探测并对其做出反应：

* `livenessProbe`:  指示Container是否正在运行。如果活动探测失败，则kubelet会杀死Container，并且Container将受其重启策略的约束。如果Container未提供活动探测，则默认状态为Success。

* `readinessProbe`: 指示Container是否已准备好为请求提供服务。如果准备探测失败，则端点控制器会从与Pod匹配的所有服务的端点中删除Pod的IP地址。初始延迟之前的默认准备状态是Failure。如果Container未提供就绪状态探测，则默认状态为Success。

### 什么时候应该livenessProbe或readinessProbe？

如果您的Container中的进程在遇到问题或变得不健康时能够自行崩溃，则您不一定需要livenessProbe; kubelet将根据Pod的内容自动执行正确的操作`restartPolicy`。

如果您希望在探测失败时杀死并重新启动Container，则指定livenessProbe，并指定`restartPolicy`为Always或OnFailure。

如果您只想在探测成功时开始向Pod发送流量，请指定readiness Probe。在这种情况下，readiness Probe可能与liveness probe 相同，但规范中存在readiness Probe意味着Pod将在不接收任何流量的情况下启动，并且仅在探测成功后才开始接收流量。

如果Container需要在启动期间处理大型数据，配置文件或迁移，请指定 readiness probe。

如果您希望Container能够自行维护，您可以指定一个readiness probe，该探针检查特定于就绪状态的端点，该端点与liveness probe不同。

请注意，如果您只想在删除Pod时排除请求，则不一定需要readiness probe; 删除时，无论是否存在readiness probe，Pod都会自动将其置于 unready 状态。Pod在等待Pod中的容器停止时仍处于 unready 状态。

## Pod readiness gate

> 状态： Kubernetes v1.12 公测

> 备注：新特性，没看明白，稍后继续。

## 重启策略

PodSpec的`restartPolicy`字段可能包含Always，OnFailure和Never。默认值为Always。 `restartPolicy`适用于Pod中的所有容器。`restartPolicy`仅指由同一节点上的 kubelet 重新启动Container。由kubelet重新启动的已退出容器将以指数退避延迟（10秒，20秒，40秒......）重新启动，上限为五分钟，并在成功执行十分钟后重置。正如[Pods文档中](https://kubernetes.io/docs/user-guide/pods/#durability-of-pods-or-lack-thereof)所讨论的 ，一旦绑定到节点，Pod将永远不会被反弹到另一个节点。


## Pod寿命

一般来说，pod不会消失，直到有人摧毁它们。这可能是人或controller。此规则的唯一例外是具有 Succeeded 的 `phase`或 Failed 超过一定持续时间（由 master 的 `terminated-pod-gc-threshold`确定）的Pod将过期并自动销毁。

有三种控制器可供选择：

- 使用[Job](https://kubernetes.io/docs/concepts/jobs/run-to-completion-finite-workloads/) for Pods预期终止，例如批量计算。作业仅适用于 `restartPolicy`等于OnFailure或Never的Pod。
- 对不希望终止的[Pod](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)（例如，Web服务器）使用[ReplicationController](https://kubernetes.io/docs/concepts/workloads/controllers/replicationcontroller/)， [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)或 [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)。ReplicationControllers仅适用于具有`restartPolicy`Always的Pod。
- 使用需要为每台计算机运行一个的[Pod的DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)，因为它们提供特定于计算机的系统服务。

所有三种类型的controllers都包含PodTemplate。建议创建适当的控制器并让它创建Pod，而不是直接创建Pod。这是因为Pods单独对机器故障没有弹性，但控制器有。

如果节点死亡或与群集的其余部分断开连接，Kubernetes会应用策略`phase`将丢失节点上的所有Pod设置为Failed。

## Examples

看原文吧。

## 其他内容

- [kubernetes之pod状态分析](https://blog.csdn.net/u013812710/article/details/72886491): 详细讲述了 pod 从创建到成功所触发的事件，以及pod相关状态的改变过程，挺细致的。
- [Kubernetes: Lifecycle of a Pod](https://dzone.com/articles/kubernetes-lifecycle-of-a-pod)
- [Kubernetes: A Pod’s Life](https://blog.openshift.com/kubernetes-pods-life/)
- [Auto-Healing Containers in Kubernetes](https://www.stratoscale.com/blog/kubernetes/auto-healing-containers-kubernetes/)






