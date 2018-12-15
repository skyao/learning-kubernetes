---
date: 2018-12-08T09:00:00+08:00
title: 服务类型
menu:
  main:
    parent: "concept-service"
weight: 333
description : "Kubernetes service的服务类型"
---

对于应用程序的某些部分（例如前端），您可能希望将服务公开到外部（集群外）IP地址。

Kubernetes `ServiceTypes`允许您指定所需的服务类型。默认是`ClusterIP`。

| 服务类型     | 描述                                                         |
| ------------ | ------------------------------------------------------------ |
| ClusterIP    | 在集群内部IP上公开服务。选择此值使服务只能从群集中访问。这是默认值`ServiceType`。 |
| NodePort     | 在每个节点IP的静态端口（`NodePort`）上公开服务。该`NodePort`服务路由的`ClusterIP`服务将自动创建。您可以通过请求`<NodeIP>:<NodePort> ` 来从集群外部连接到该NodePort服务。 |
| LoadBalancer | 使用云提供商的负载均衡器在外部公开服务。外部负载均衡器路由的`NodePort`和`ClusterIP`服务将自动创建。 |
| ExternalName | 通过返回带有其值的CNAME记录，将服务映射到`externalName`字段的内容 （例如`foo.bar.example.com`）。不设置任何类型的代理。需要1.7或更高版本`kube-dns`。 |

### NodePort类型

如果将type字段设置为NodePort，Kubernetes master将从 `--service-node-port-range` flag 指定的范围（默认值：30000-32767）中分配一个端口，并且每个Node将代理该端口（每个节点上的相同端口号）到您的Service。该端口将在Service的 `.spec.ports[*].nodePort` 字段中出现。

如果要指定代理端口的特定IP，可以将 kube-proxy 中的 `--nodeport-addresses` 标志设置为特定的IP块（自Kubernetes v1.10起支持）。以逗号分隔的IP块列表（例如`10.0.0.0/8,1.2.3.4/32`）用于过滤此节点的本地地址。例如，如果您使用 `--nodeport-addresses=127.0.0.0/8` flag 启动 kube-proxy ，则 kube-proxy 将仅为NodePort服务选择环回接口。该`--nodeport-addresses`默认为空（[]），这意味着选择所有可用的接口，符合当前NodePort行为。

如果您需要特定的端口号，可以在nodePort 字段中指定一个值，系统将为您分配该端口，否则API事务将失败（即您需要自己处理可能的端口冲突）。您指定的值必须在节点端口的配置范围内。

这使开发人员可以自由地设置自己的负载均衡器，在不完全支持的环境中配置Kubernetes，甚至只直接暴露一个或多个节点的IP。

请注意，此服务将同时在 `<NodeIP>:spec.ports[*].nodePort`  和 `.spec.clusterIP:spec.ports[*].port` 上可见。（如果设置了kube-proxy中的 `--nodeport-addresses` 标志， 则是过滤后的NodeIP。）

### LoadBalancer类型

在支持外部负载均衡器的云提供商上，将 type 字段设置为LoadBalancer将为您的负载均衡器配置负载均衡器 Service。负载均衡器的实际创建异步发生，以及有关提供商均衡器信息将在公布Service的 `.status.loadBalancer` 领域。

### ExternalName类型

ExternalName类型的服务将服务映射到DNS名称（使用`spec.externalName` 参数指定），而不是典型的选择器 像如my-service或cassandra。例如，此服务定义会将 `prod` 命名空间中的 my-service 服务映射到 `my.database.example.com`：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
  namespace: prod
spec:
  type: ExternalName
  externalName: my.database.example.com
```

查找主机 `my-service.prod.svc.cluster.local` 时，集群DNS服务将返回值为 `my.database.example.com` 的 CNAME 记录。访问 my-service工作的方式与其他服务相同，但重要的区别在于重定向发生在DNS级别，而不是通过代理或转发。如果您以后决定将数据库移动到集群中，则可以启动其pod，添加适当的选择器或端点，以及更改服务type。

### 外部IP

如果有外部IP路由到一个或多个集群节点，则可以在这些节点的 externalIP 上公开Kubernetes服务。在服务端口上使用外部IP（作为 destination IP）进入集群的流量将路由到其中一个服务端点。externalIP 不由Kubernetes管理，而是集群管理员的责任。

在 `ServiceSpec`，externalIP 可以在任意一个ServiceType中指定。在下面的示例中，my-service客户端可以通过“ 80.11.12.10:80”“（externalIP:port）访问：

```yaml
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.10
```

官方文档中有个教程：[Exposing an External IP Address to Access an Application in a Cluster](https://kubernetes.io/docs/tutorials/stateless-application/expose-external-ip-address/)

### 其他方式

还有其他集中暴露服务的方式：

- hostNetwork
- hostPort
- Ingress

### 参考文档

- [Accessing Kubernetes Pods from Outside of the Cluster](http://alesnosek.com/blog/2017/02/14/accessing-kubernetes-pods-from-outside-of-the-cluster/): 有中文翻译版本 [从外部访问Kubernetes中的Pod](https://jimmysong.io/posts/accessing-kubernetes-pods-from-outside-of-the-cluster/)