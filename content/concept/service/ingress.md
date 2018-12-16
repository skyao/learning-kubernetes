---
date: 2018-12-08T09:00:00+08:00
title: Ingress
menu:
  main:
    parent: "concept-service"
weight: 337
description : "Kubernetes的Ingress"
---

> 备注： 内容摘要自 [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

### Ingress 是什么？

在Kubernetes v1.1中添加的Ingress公开HTTP和HTTPS路由，可以从集群外部访问集群内的服务 。流量路由被Ingress资源上定义的规则控制。

```bash
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]
```

可以将Ingress配置为给服务提供外部可访问的URL，负载均衡流量，终止SSL以及提供基于名称的虚拟主机。一个Ingress Controller负责履行ingress，通常与负载均衡器一起使用，当然它也可以配置边缘路由器或额外的前端，以帮助处理流量。

Ingress不会暴露任意端口或协议。将除HTTP和HTTPS之外的服务公开到Internet通常使用 `Service.Type = NodePort` 或 `Service.Type = LoadBalancer` 类型的服务。

### Ingress Conntroller

为了使Ingress资源正常工作，群集必须运行Ingress控制器。这与其他类型的控制器不同，后者作为 `kube-controller-manager` 二进制文件的一部分运行，通常随集群自动启动。选择最适合你的集群的Ingress控制器实现。

- kubernetes 目前支持和维护 [GCE](https://git.k8s.io/ingress-gce/README.md) and [nginx](https://git.k8s.io/ingress-nginx/README.md) controllers.

其他控制器包括：

- Contour 是由Heptio提供和支持的基于Envoy的Ingress控制器。
- F5 Networks 为 Kubernetes 的 F5 BIG-IP 控制器提供支持和维护。
- 基于HAProxy的 Ingress 控制器 `jcmoraisjr/haproxy-ingress` 在博客文章 HAProxy Ingress Controller for Kubernetes中提到。HAProxy Technologies为 HAProxy Enterprise 和Ingress控制器 `jcmoraisjr/haproxy-ingress` 提供支持和维护。
- 基于 Istio 的Ingress控制器 控制入口流量。
- Kong为 Kong Ingress Controller for Kubernetes 提供社区或 商业支持和维护 
- NGINX，Inc。为Kubernetes的 NGINX Ingress 控制器提供支持和维护 。
- Traefik是一个功能齐全的入口控制器（Let's Encrypt，secrets，http2，websocket），它还附带了Containous的商业支持。

可以在集群中部署任意数量的Ingress控制器。创建Ingress时，应使用适当的注释对每个Ingress进行注释， ingress-class以指示如果集群中存在多个Ingress控制器，则应使用哪个。如果您未定义class，则云提供商可能会使用默认的Ingress。

### Ingress类型

- Single Service Ingress
- Simple fanout
- Name based virtual hosting

主要就是配置spec里面的几个关键字段 host/path/backend :

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: first.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
```

### 更新Ingress

直接修改：

```bash
kubectl describe ingress test
kubectl edit ingress test
```

### Nginx Ingress

- https://kubernetes.github.io/ingress-nginx/
- Installation Guide: https://kubernetes.github.io/ingress-nginx/deploy/

安装：

```bash
$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.extensions/nginx-ingress-controller created
```

前面的命令创建了namespace，configmap，serviceaccount，deploy，还要继续创建service，对于bare-metal方式安装的k8s：

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```

TBD: 配置dashboard的访问









