---
title: "Service"
linkTitle: "Service"
weight: 10
date: 2021-02-01
description: >
  Kubernetes Service
---

> https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/

Service 是将运行在一组 Pods 上的应用程序公开为网络服务的抽象方法。

## 动机

- Pod 是非永久性资源
- 每个 Pod 都有自己的 IP 地址
- 在 Deployment 中，在同一时刻运行的 Pod 集合可能与稍后运行该应用程序的 Pod 集合不同。

## Service 资源

Kubernetes Service 定义了这样一种抽象：逻辑上的一组 Pod 和一种可以访问它们的策略

## 定义 Service

定义 service 需要最基本的两个元素：

1. service selector / 服务选择算符
2. service port

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app.kubernetes.io/name: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 9376
```

> 小知识：Pod 中的端口定义是有名字的，可以在 Service 的 `targetPort` 属性中引用这些名称。这样可以只修改pod定义，service 这边就自动改过来了。

### 没有选择算符的 Service

服务没有选择算符时，不会自动创建相应的 EndpointSlice（和旧版 Endpoint）对象。 可以通过手动添加 EndpointSlice 对象，将服务手动映射到运行该服务的网络地址和端口



## 发布服务（服务类型）

Kubernetes `ServiceTypes` 允许指定你所需要的 Service 类型。

`Type` 的取值以及行为如下：

- `ClusterIP`：通过集群的内部 IP 暴露服务，选择该值时服务只能够在集群内部访问。 这也是你没有为服务显式指定 `type` 时使用的默认值。
- [`NodePort`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#type-nodeport)：通过每个节点上的 IP 和静态端口（`NodePort`）暴露服务。 为了让节点端口可用，Kubernetes 设置了集群 IP 地址，这等同于你请求 `type: ClusterIP` 的服务。
- [`LoadBalancer`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#loadbalancer)：使用云提供商的负载均衡器向外部暴露服务。 外部负载均衡器可以将流量路由到自动创建的 `NodePort` 服务和 `ClusterIP` 服务上。
- [`ExternalName`](https://kubernetes.io/zh-cn/docs/concepts/services-networking/service/#externalname)：通过返回 `CNAME` 记录和对应值，可以将服务映射到 `externalName` 字段的内容（例如，`foo.bar.example.com`）。 无需创建任何类型代理。

特别注意： `type` 字段被设计为嵌套功能 - 每个级别都添加到前一个级别。

也可以使用 [Ingress](https://kubernetes.io/zh-cn/docs/concepts/services-networking/ingress/) 来暴露自己的服务。

### NodePort 类型

如果你将 `type` 字段设置为 `NodePort`，则 Kubernetes 控制平面将在 `--service-node-port-range` 标志指定的范围内分配端口（默认值：30000-32767）。

#### 选择你自己的端口

如果需要特定的端口号，你可以在 `nodePort` 字段中指定一个值。

- 需要自己注意可能发生的端口冲突
- 必须使用有效的端口号，该端口号在配置用于 NodePort 的范围内。
