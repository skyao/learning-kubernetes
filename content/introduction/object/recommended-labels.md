---
date: 2018-12-08T09:00:00+08:00
title: 推荐标签
menu:
  main:
    parent: "introduction-object"
weight: 256
description : "Kubernetes推荐标签"
---

> 备注： 内容来自 [Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)

您可以使用比 kubectl 和 dashboard 更多的工具来可视化和管理Kubernetes对象。 一组通用的标签允许工具以互操作的方式工作，以所有工具都能理解的通用方式描述对象。

除了支持工具之外，推荐标签还以可查询的方式描述应用程序。

元数据围绕应用程序的概念进行组织。 Kubernetes不是一个 Platform as a Service（PaaS），没有也不会强制推行正式的应用程序概念。 相反，应用程序是非正式的，用元数据描述。 应用程序包含的内容的定义是松散的。

> 注意：这些是推荐的标签。 它们使管理应用程序变得更容易，但对于任何核心工具都不是必需的。

共享标签和注释共享一个共同的前缀：`app.kubernetes.io`。 没有前缀的标签对用户是私有的。 共享前缀可确保共享标签不会干扰自定义用户标签。

### Label

为了充分利用这些标签，应将它们应用于每个资源对象。

| Key                            | 描述                                 | 示例               | 类型   |
| ------------------------------ | ------------------------------------ | ------------------ | ------ |
| `app.kubernetes.io/name`       | 应用名                               | `mysql`            | string |
| `app.kubernetes.io/instance`   | 标记应用示例的唯一名字               | `wordpress-abcxzy` | string |
| `app.kubernetes.io/version`    | （例如，语义化版本，修订版本哈希等） | `5.7.21`           | string |
| `app.kubernetes.io/component`  | 在架构中的组件                       | `database`         | string |
| `app.kubernetes.io/part-of`    | 这个应用所述的更高级别的应用名       | `wordpress`        | string |
| `app.kubernetes.io/managed-by` | 用于管理应用运营的工具               | `helm`             | string |

要说明这些标签的运行情况，请考虑以下StatefulSet对象：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
```

### 应用和应用实例

应用程序可以一次或多次安装到Kubernetes集群中，在某些情况下，可以安装在相同的名称空间中。 例如，wordpress可以不止一次安装，其中不同的网站是wordpress的不同安装。

应用程序的名称和实例名称分别记录。 例如，WordPress有值为 `wordpress`的`app.kubernetes.io/name` 标签，同时也有一个实例名称，表示为`app.kubernetes.io/instance`，值为 `wordpress-abcxzy` 。 这使应用程序的应用程序和实例可以识别。应用程序的每个实例都必须具有唯一的名称。

### 示例

为了说明使用这些标签的不同方式，以下示例具有不同的复杂性。

#### 简单无状态服务

考虑使用 Deployment 和 Service 对象部署的简单无状态服务的情况。 以下两个代码段表示如何以最简单的形式使用标签。

Deployment 用于监视运行应用程序本身的pod。

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

Service 用于暴露应用。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: myservice
    app.kubernetes.io/instance: myservice-abcxzy
...
```

#### 带有数据库的web应用

考虑一个稍微复杂的应用程序：用Helm安装的使用数据库（MySQL）的Web应用程序（WordPress）。 以下代码段说明了用于部署此应用程序的对象的开始。

以下 Deployment 的开始用于WordPress：

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

Service 用于暴露 WordPress: 

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: wordpress
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/version: "4.9.4"
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: server
    app.kubernetes.io/part-of: wordpress
...
```

MySQL作为 `StatefulSet` 公开，包含它和它所属的较大应用程序的元数据：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/version: "5.7.21"
...
```

Service 用于将MySQL作为WordPress的一部分公开：

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: wordpress-abcxzy
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/version: "5.7.21"
...
```

使用 MySQL StatefulSet和 Service，您会注意到有关MySQL和Wordpress的信息，包括更广泛的应用程序。





