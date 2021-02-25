---
title: "推翻KEP753的讨论"
linkTitle: "推翻KEP753的讨论"
weight: 3
date: 2021-02-23
description: >
  推翻 Kubernetes KEP753 Sidecar Container的讨论
---

https://github.com/kubernetes/enhancements/pull/1980

这是一个关于 sidecar 的讨论汇总，最后得出的结论是推翻 kep753.

### 起于derekwaynecarr的发言

I want to capture my latest thoughts on sidecar concepts, and get a path forward.

Here is my latest thinking:

> 我想归纳我对 sidecar 概念的最新思考，并得到一条前进的道路。
> 
> 这是我的最新思考。

I think it’s important to ask if the introduction of sidecar containers will actually address an end-user requirement or just shift a problem and further constrain adoption of sidecars themselves by pod authors. To help frame this exercise, I will look at the proposed use of sidecar containers in the service mesh community.

> 我认为重要的是要问一下 sidecar容器的引入是否会真正解决最终用户的需求，或者只是转移一个问题，并进一步限制pod作者对sidecars本身的采用。为了帮助构架这项工作，我将看看服务网格社区中拟议的 sidecar 容器的使用情况。

**User story**

I want to enable mTLS for all traffic in my mesh because my auditor demands it.

> 我想在我的Mesh中启用mTLS，因为我的会计要求这样做。

The proposed solution is the introduction of sidecar containers that change the pod lifecycle:

> 提出的解决方案是引入sidecar container，改变 pod 的生命周期:

1. Init containers start/stop
2. Sidecar containers start
3. Primary containers start/stop
4. Sidecar containers stop

The issue with the proposed solution meeting the user story is as follows:

> 建议的解决方案可以满足用户故事的问题如下：

- Init containers are not subject to service mesh because the proxy is not running. This is because init containers run to completion before starting the next container. Many users do network interaction that should be subject to the mesh in their init container.

   > Init container 不受服务网格的影响，因为代理没有运行。这是因为init container 在启动下一个容器之前会运行到完成状态。很多用户在 init container 中做网络交互，应该受制于网格。

- Sidecar containers (once introduced) will be used by users for use cases unrelated to the mesh, but subject to the mesh. The proposal makes no semantic guarantees on ordering among sidecars. Similar to init containers, this means sidecars are not guaranteed to participate in the mesh.

   > Sidecar 容器(一旦引入)将被用户用于与网格无关但受网格制约的用例。该提案没有对sidecars之间的顺序进行语义保证。与 init 容器类似，这意味着 sidecar 不能保证参与 mesh。

The real requirement is that the proxy container MUST stop last even among sidecars if those sidecars require network.

> **真正的需求是，如果这些sidecar需要网络，代理容器也必须最后停止，即使代理容器也是 sidecar**。

Similar to the behavior observed with init containers (users externalize run-once setup from their main application container), the introduction of sidecar containers will result in more elements of the application getting externalized into sidecars, but those elements will still desire to be part of the mesh when they require a network. Hence, we are just shifting, and not solving the problem.

> 与观察到的init容器的行为类似（用户从他们的主应用容器中外部化一次性设置），引入sidecar容器将导致更多的应用元素被外部化到sidecar中，但是当这些元素需要网络时，它们仍然会渴望成为网格结构的一部分。因此，**我们只是在转移，而不是解决问题**。

Given the above gaps, I feel we are not actually solving a primary requirement that would drive improved adoption of a service mesh (ensure all traffic is mTLS from my pod) to meet auditing.

> 鉴于上述差距，我觉得我们并没有实际上解决主要需求，这个需求将推动服务网格的改进采用（确保所有来自我的pod的流量都是mTLS），以满足审计。

Alternative proposal:

- Support an ordered graph among containers in the pod (it’s inevitable), possibly with N rings of runlevels?
- Identify which containers in that graph must run to completion before initiating termination (Job use case).
- Move init containers into the graph (collapse the concept)
- Have some way to express if a network is required by the container to act as a hint for the mesh community on where to inject a proxy in the graph.

> 替代建议:
>
> - 支持在pod中的容器之间建立一个有序图(这是不可避免的)，可能有N个运行级别的环？
> - 识别该图中的哪些容器必须在启动终止之前运行至完成状态（Job用例）。
> - 将 init 容器移入图中（折叠概念）。
> - 有某种方式来标记容器是否需要网络，用来作为网格社区的提示，在图中某处注入代理。