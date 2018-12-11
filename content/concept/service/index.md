---
date: 2018-12-08T09:00:00+08:00
title: Service
weight: 330
description : "Kubernetes的service概念"
---

> 备注： 内容来自 [Services](https://kubernetes.io/docs/concepts/services-networking/service/)

Kubernetes [`Pods`](https://kubernetes.io/docs/concepts/workloads/pods/pod/) are mortal. They are born and when they die, they are not resurrected. [`ReplicaSets`](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/) in particular create and destroy `Pods` dynamically (e.g. when scaling out or in). While each `Pod` gets its own IP address, even those IP addresses cannot be relied upon to be stable over time. This leads to a problem: if some set of `Pods` (let’s call them backends) provides functionality to other `Pods` (let’s call them frontends) inside the Kubernetes cluster, how do those frontends find out and keep track of which backends are in that set?

Enter `Services`.

Kubernetes Pods是致命的。他们出生，死后，他们没有复活。 ReplicaSet特别是动态地创建和销毁Pod（例如，当向外扩展时或在其中）。虽然每个Pod都有自己的IP地址，但即使是那些IP地址也不能依赖它们随时间变得稳定。这导致了一个问题：如果某些Pod（让我们称之为后端）为Kubernetes集群内的其他Pod（让我们称之为前端）提供功能，那些前端如何找出并跟踪该集合中的哪些后端？

输入服务。

A Kubernetes `Service` is an abstraction which defines a logical set of `Pods` and a policy by which to access them - sometimes called a micro-service. The set of `Pods` targeted by a `Service` is (usually) determined by a [`Label Selector`](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors) (see below for why you might want a `Service` without a selector).

As an example, consider an image-processing backend which is running with 3 replicas. Those replicas are fungible - frontends do not care which backend they use. While the actual `Pods` that compose the backend set may change, the frontend clients should not need to be aware of that or keep track of the list of backends themselves. The `Service` abstraction enables this decoupling.

For Kubernetes-native applications, Kubernetes offers a simple `Endpoints` API that is updated whenever the set of `Pods` in a `Service` changes. For non-native applications, Kubernetes offers a virtual-IP-based bridge to Services which redirects to the backend `Pods`.



