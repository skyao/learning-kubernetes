---
date: 2018-12-08T09:00:00+08:00
title: Headless服务
menu:
  main:
    parent: "concept-service"
weight: 332
description : "Kubernetes的headles服务"
---

有时您不需要负载均衡和单个服务IP。在这种情况下，您可以通过指定cluster IP（`.spec.clusterIP`）为`"None"`来创建“headless”服务。

此选项允许开发人员通过允许他们自由地以自己的方式进行发现来减少与Kubernetes系统的耦合。应用程序仍然可以使用自注册模式，并且可以轻松地在此API上构建适用于其他发现系统的适配器。

对于这种`Services`，不分配cluster IP，kube-proxy不处理这些服务，并且平台没有为它们实现负载均衡或代理。如何自动配置DNS取决于服务是否已定义选择器。

### 有选择器

对于定义了选择器的headerless服务，端点控制器在API中创建Endpoints记录，并修改DNS配置以返回直接指向组成服务的Pod的A记录（地址）。

### 没有选择器

对于未定义选择器的headless服务，端点控制器不会创建`Endpoints`记录。但是，DNS系统会查找并配置：

- 对于类型为 [`ExternalName`](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) 的服务，使用CNAME记录
- 对于所有其他类型，所有与服务共享名称的Endpoint的A记录

