---
date: 2018-12-08T09:00:00+08:00
title: Pod
weight: 310
description : "Kubernetes的Pod概念"
---

> 备注： 内容来自 [Pod Overview](https://kubernetes.io/docs/concepts/workloads/pods/pod-overview/)

此页面提供Pod的概述，pod 是 Kubernetes 对象模型中最小的可部署对象。

## 理解Pod

Pod是 Kubernetes 的基本构建块—— pod是要创建或部署的 Kubernetes 对象模型中最小最简单的单元。Pod表示集群上正在运行的进程。

Pod封装了一个应用容器（或者，在某些情况下，多个容器），存储资源，唯一的网络IP以及控制容器应该如何运行的选项。Pod表示部署单元：*Kubernetes中的单个应用实例*，可能包含单个容器或少量紧密耦合且共享资源的容器。

> [Docker](https://www.docker.com/)是Kubernetes Pod中最常见的容器运行时，但Pods也支持其他容器运行时。

Kubernetes集群中的Pod可以以两种主要方式使用：



* **运行单个容器的Pod**. “one-container-per-Pod”模型是最常见的Kubernetes用例; 在这种情况下，您可以将Pod视为单个容器的包装，而Kubernetes管理Pod而不是直接管理容器。
* **运行多个需要协同工作的容器的Pod**. Pod可能封装了一个由多个共址容器组成的应用，这些容器紧密耦合并需要共享资源。这些共处一地的容器可能形成一个统一的服务单元 - 一个容器从共享卷向公众提供文件，而一个单独的“sidecar”容器刷新或更新这些文件。Pod将这些容器和存储资源作为单个可管理实体包装在一起。

[Kubernetes博客](http://kubernetes.io/blog)对 pod 用例有一些额外的信息。有关更多信息，请参阅：

- [分布式系统工具包：复合容器的模式](https://kubernetes.io/blog/2015/06/the-distributed-system-toolkit-patterns)
- [容器设计模式](https://kubernetes.io/blog/2016/06/container-design-patterns)

每个Pod都用于运行给定应用程序的单个实例。如果要水平扩展应用程序（例如，运行多个实例），则应使用多个Pod，每个实例一个。在Kubernetes中，这通常被称为副本（replication）。副本 Pod 通常通过称为Controller的抽象来创建和管理。

### Pod如何管理多个容器

Pod旨在支持多个协作流程（作为容器），形成一个有凝聚力的服务单元。Pod中的容器将自动共存，并在集群中的同一物理或虚拟机上进行共同调度。容器可以共享资源和依赖关系，彼此通信，并协调它们何时以及如何终止。

请注意，将多个共址和共同管理的容器分组到一个Pod中是一个相对高级的用例。您应该仅在容器紧密耦合的特定实例中使用此模式。例如，您可能有一个容器充当共享卷中文件的Web服务器，以及一个单独的“sidecar”容器，用于从远程源更新这些文件，如下图所示：

![img](https://d33wubrfki0l68.cloudfront.net/aecab1f649bc640ebef1f05581bfcc91a48038c4/728d6/images/docs/pod.svg)

Pod为其组成容器提供两种共享资源：网络和存储。

#### 网络

每个Pod分配有一个唯一的IP地址。Pod中的每个容器都共享网络命名空间，包括IP地址和网络端口。Pod内的容器可以使用`localhost` 相互通信。当Pod中的容器与 Pod 外部的实体通信时，它们必须协调如何使用共享网络资源（例如端口）。

#### 存储

Pod可以指定一组共享存储*卷*。Pod中的所有容器都可以访问共享卷，允许这些容器共享数据。如果需要重新启动其中一个容器，则卷还允许Pod中的持久数据存活。

## 使用Pod

你很少直接在Kubernetes创建单独的Pod - 即使是单例Pod。这是因为Pod被设计为相对短暂的一次性实体。当Pod创建时（由您直接创建或由Controller间接创建），它将被安排在集群中的节点上运行。Pod保留在该节点上，直到进程终止，pod对象被删除，pod 因资源不足而被逐出，或者Node失败。

> **注意：**不要将重新启动Pod和重新启动Pod中的容器混淆。Pod本身不会运行，但是容器运行的环境会持续存在，直到删除为止。

Pod本身不能自我修复。如果将Pod调度到失败的节点，或者调度操作本身失败，则删除Pod; 同样，由于缺乏资源或节点维护，Pod将无法在逐出中存活。Kubernetes使用更高级别的抽象，称为Controller，来处理管理相对可处理的Pod实例的工作。因此，尽管可以直接使用Pod，但在Kubernetes中使用Controller管理pod更为常见。

