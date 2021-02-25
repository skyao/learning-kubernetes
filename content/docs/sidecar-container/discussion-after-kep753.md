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

A few other notes based on Red Hat's experience with service mesh:

Red Hat does not support injection of privileged sidecar containers and will always require CNI approach. In this flow, the CNI runs, multus runs, iptables are setup, and then init containers start. The iptables rules are setup, but no proxy is running, so init containers lose connectivity. Users are unhappy that init containers are not participating in the mesh. Users should not have to sacrifice usage of an init container (or any aspect of the pod lifecycle) to fulfill auditor requirements. The API should be flexible enough to support graceful introduction in the right level of a intra pod life-cycle graph transparent to the user.

> 根据红帽在服务网格方面的经验，还有一些其他说明：
>
> 红帽不支持注入特权sidecar容器，总是需要CNI方式。在这个流程中，CNI运行，multus运行，设置iptables，然后 init 容器启动。iptables规则设置好了，但是没有代理运行，所以 init容器 失去了连接。用户对init容器不参与网格感到不满。用户不应该为了满足审计师的要求而牺牲init容器的使用（或pod生命周期的任何方面）。API应该足够灵活，以支持在正确的层次上优雅地引入对用户透明的 pod 生命周期图。

**Proposed next steps:**

- Get a dedicated set of working meetings to ensure that across the mesh and kubernetes community, we can meet a users auditing requirement without limiting usage or adoption of init containers and/or sidecar containers themselves by pod authors.
- I will send a doodle.

> 拟议的下一步措施：
>
> 召开一组专门的工作会议，以确保在整个mesh和kubernetes社区，我们可以满足用户审计要求，而不限制pod作者使用或采用init容器和/或sidecar容器本身。
>
> 我会发一个涂鸦。

### 其他人的意见

**mrunalp**：

Agree! We might as well tackle this general problem vs. doing it step by step with baggage added along the way.

> 同意! 我们不妨解决这个普遍性的问题，而不是按部就班地做，在做的过程中增加包袱。

**sjenning** ：

I agree [@derekwaynecarr](https://github.com/derekwaynecarr)

I think that in order to satisfy fully the use cases mentioned, we are gravitating toward systemd level semantics where there is just an ordered graph of ~~services~~ containers in the pod spec.

You could basically collapse init containers into the normal containers map and add two fields to `Container`; `oneshot bool` that expresses if the container terminates and dependent containers should wait for it to terminate (handles init containers w/ ordering), and `requires map[string]` a list of container names upon which the current container depends.

This is flexible enough to accommodate a `oneshot: true` container (init container) depending on a `oneshot: false` container (a proxy container on which the init container depends).

Admittedly this would be quite the undertaking and there is API compatibility to consider.

> 我同意 @derekwaynecarr
>
> 我认为，为了充分满足上述用例，我们正在倾向于systemd级别的语义，在pod规范中，需要有一个有序的~~服务~~容器图。
>
> 你基本上可以把init容器折叠到普通容器图中，并在Container中添加两个字段；` oneshot bool`，表示容器是否终止，依赖的容器是否应该等待它终止（处理init容器 w/排序），和 `requires map[string]` 一个当前容器依赖的容器名称列表。
>
> 这足够灵活，可以容纳一个  `oneshot: true`  容器（init 容器）依赖于一个 `oneshot: false` 容器（init 容器依赖的代理容器）。
>
> 诚然，这将是一个相当大的工程，而且还要考虑API的兼容性。

**thockin：**

I have also been thinking about this. There are a number of open issues, feature-requests, etc that all circle around the topic of pod and container lifecycle. I've been a vocal opponent of complex API here, but it's clear that what we have is inadequate.

When we consider init-container x sidecar-container, it is clear we will inevitably eventually need an init-sidecar.

> 我也一直在思考这个问题。有一些开放的问题、功能需求等，都是围绕着pod和容器生命周期这个话题展开的。我在这里一直是复杂API的强烈反对者，但很明显，我们所拥有的是不够的。
>
> 当我们考虑 init-container x sidecar-container 时，很明显我们最终将不可避免地需要一个init-sidecar。

Some (non-exhaustive) of the other related topics:

- Node shutdown -> Pod shutdown (in progress?)
- Voluntary pod restart ("Something bad happened, please burn me down to the ground and start over")
- Voluntary pod failure ("I know something you don't, and I can't run here - please terminate me and do not retry")
- "Critical" or "Keystone" containers ("when this container exits, the rest should be stopped")
- Startup/shutdown phases with well-defined semantics (e.g. "phase 0 has no network")
- Mixed restart policies in a pod (e.g. helper container which runs and terminates)
- Clearer interaction between pod, network, and device plugins

> 其他的一些（非详尽的）相关主题：
> 
> - 节点关闭 -> Pod关闭（正在进行中？
> - 自愿重启pod（"发生了不好的事情，请把我摧毁，然后重新开始"）。
> - 自愿pod失败("我知道一些你不知道的事情，我无法在这里运行--请终止我，不要重试")
> - "关键 "或 "基石 "容器（"当这个容器退出时，其他容器应停止"）。
> - 具有明确语义的启动/关闭阶段（如 "phase 0 没有网络"）。
> - 在一个pod中混合重启策略（例如，帮助容器，它会运行并终止）。
> - 更清晰的 pod、网络和设备插件之间的交互。

** thockin：**

This is a big enough topic that we almost certainly need to explore multiple avenues before we can have confidence in any one.

> 这是一个足够大的话题，我们几乎肯定需要探索多种途径，才能对任何一种途径有信心。

**kfox1111**: 

the dependency idea also would allow for doing an init container, then a sidecar network plugin, then more init containers, etc, which has some nice features.

Also the readyness checks and oneshot could all play together with the dependencies so the next steps aren't started before ready.

So, as a user experience, I think that api might be very nice.

Implementation wise there are probably lots of edge cases to carefully consider there.

> 依赖的想法还可以做一个init容器，然后做一个sidecar网络插件，然后做更多的init容器等等，这有一些不错的功能。
>
> 另外 readyness 检查和 oneshot 都可以和依赖一起考虑，这样就不会在准备好之前就开始下一步。
>
> 所以，作为用户体验来说，我觉得这个api可能是非常不错的。
>
> 从实现上来说，可能有很多边缘情况需要仔细考虑。

**SergeyKanzhelev:**

this is great idea to set up a working group to move it forward in bigger scope. One topic I suggest we cover early on in the discussions is whether we need to address the existing pain point of injecting sidecars in jobs in 1.20. This KEP intentionally limited the scope to just this - formalizing what people are already trying to do today with workarounds.

From Google side we also would love the bigger scope of a problem be addressed, but hope to address some immediate pain points early if possible. Either in current scope or [slightly bigger](https://github.com/kubernetes/enhancements/pull/1913#discussion_r463923308).

> 这是一个很好的想法，成立一个工作组，在更大范围内推进它。我建议我们在讨论中尽早涉及的一个话题是，我们是否需要在1.20中解决现有的Job中注入 sidecar 的痛点。这个KEP有意将范围限制在这一点上--将人们今天已经在尝试的工作方法正式化。
>
> 从Google方面来说，我们也希望更大范围的问题能够得到解决，但如果可能的话，希望能够尽早解决一些直接的痛点。要么在目前的范围内，要么稍微大一点。

**derekwaynecarr：**

I would speculate that the dominant consumer of the job scenario is a job that required participation in a mesh to complete its task, and since I don't see much point in solving for the mesh use case (which I view as the primary motivator for defining side car semantics) for only one workload type, I would rather ensure a pattern that solves the problem in light of our common experience across mesh and k8s communities.

>  我推测工作场景的主要消费者是需要参与网格来完成任务的Job，由于我认为只为一种工作负载类型解决mesh用例（我认为这是定义 sidecar 语义的主要动机）没有太大意义，所以我宁愿根据我们在 mesh 和k8s社区中的共同经验，确保一个能解决问题的模式。