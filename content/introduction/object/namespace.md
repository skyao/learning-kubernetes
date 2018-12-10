---
date: 2018-12-08T09:00:00+08:00
title: Object Namespace
menu:
  main:
    parent: "introduction-object"
weight: 152
description : "Kubernetes对象的namespace属性"
---

> 备注： 内容来自 [Namespaces](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)

Kubernetes支持由同一物理集群支持的多个虚拟集群。 这些虚拟集群称为命名空间(Namespace)。

### 什么时候使用多个Namespace？

Namespace旨在用于多个用户分布在多个团队或项目中的环境中。对于具有几个到几十个用户的集群，您根本不需要创建或考虑namespace。当您需要namespace提供的功能时，请开始使用他们。

Namespace提供名称范围。 资源名称在namespace中必须是唯一的，但跨namespace可以相同。

Namespace是一种在多个用户之间划分集群资源的方法(通过 [resource quota](https://kubernetes.io/docs/concepts/policy/resource-quotas/))。

在Kubernetes的未来版本中，默认情况下，同一namespace中的对象将具有相同的访问控制策略。

没有必要使用多个namespace来分隔略有不同的资源，例如同一软件的不同版本：使用标签（label）来区分同一namespace中的资源。

### 使用Namespace

[名称空间的管理指南文档](https://kubernetes.io/docs/admin/namespaces) 中描述了名称空间的创建和删除。

#### 查看namespace

您可以使用以下命令列出集群中当前的namespace：

```bash
$ kubectl get namespaces
NAME          STATUS    AGE
default       Active    1d
kube-system   Active    1d
kube-public   Active    1d
```

Kubernetes一开始就带有三个初始名称空间：

- `default` ：没有其他名称空间的对象的默认名称空间
- `kube-system`： Kubernetes系统创建的对象的命名空间
- `kube-public`： 此命名空间是自动创建的，并且可供所有用户（包括未经过身份验证的用户）读取。 此命名空间主要用于集群使用，以备某些资源在整个集群中可见且可公开读取。 此命名空间的公共方面只是一个约定，而不是一个要求。

#### 为请求设置namespace

要临时设置请求的命名空间，请使用`--namespace`标志。

例如：

```bash
$ kubectl --namespace=<insert-namespace-name-here> run nginx --image=nginx
$ kubectl --namespace=<insert-namespace-name-here> get pods
```

#### 设置命名空间首选项

您可以在该上下文中为所有后续 kubectl 命令永久保存命名空间。

```bash
$ kubectl config set-context $(kubectl config current-context) --namespace=<insert-namespace-name-here>
# 验证一下
$ kubectl config view | grep namespace:
```

备注：这个功能很实用，比如在用istio时，每次命令都要输入`--namespace=istio-system`，很累，设置之后就简单多了：

```bash
# 设置之前
$ kubectl get pods --namespace istio-system
# 设置
$ kubectl config set-context $(kubectl config current-context) --namespace=istio-system
Context "kubernetes-admin@kubernetes" modified.
# 设置之后
$ kubectl get pods
```

#### Namespace和DNS

创建service时，会创建相应的DNS条目。 此条目的格式为  `<service-name>.<namespace-name>.svc.cluster.local`，这意味着如果容器只使用 `<service-name>`，它将解析为名称空间本地的service。 这对于在多个名称空间（如Development，Staging和Production）中使用相同的配置非常有用。 如果要跨命名空间访问，则需要使用完全限定的域名（fully qualified domain name/FQDN）。

### 不是所有对象都在Namespace中

大多数Kubernetes资源（例如pod，service，replication controllers等）都在某些名称空间中。 但是，命名空间资源本身并不在命名空间中。 并且低级资源（例如 node 和 persistentVolumes）不在任何名称空间中。

要查看哪些Kubernetes资源在命名空间中，哪些不在：

```bash
$ kubectl api-resources --namespaced=true
$ kubectl api-resources --namespaced=false
```

后者的输出如下，作为参考：

```bash
$  kubectl api-resources --namespaced=false
NAME                              SHORTNAMES   APIGROUP                       NAMESPACED   KIND
componentstatuses                 cs                                          false        ComponentStatus
namespaces                        ns                                          false        Namespace
nodes                             no                                          false        Node
persistentvolumes                 pv                                          false        PersistentVolume
mutatingwebhookconfigurations                  admissionregistration.k8s.io   false        MutatingWebhookConfiguration
validatingwebhookconfigurations                admissionregistration.k8s.io   false        ValidatingWebhookConfiguration
customresourcedefinitions         crd,crds     apiextensions.k8s.io           false        CustomResourceDefinition
apiservices                                    apiregistration.k8s.io         false        APIService
meshpolicies                                   authentication.istio.io        false        MeshPolicy
tokenreviews                                   authentication.k8s.io          false        TokenReview
selfsubjectaccessreviews                       authorization.k8s.io           false        SelfSubjectAccessReview
selfsubjectrulesreviews                        authorization.k8s.io           false        SelfSubjectRulesReview
subjectaccessreviews                           authorization.k8s.io           false        SubjectAccessReview
certificatesigningrequests        csr          certificates.k8s.io            false        CertificateSigningRequest
podsecuritypolicies               psp          extensions                     false        PodSecurityPolicy
podsecuritypolicies               psp          policy                         false        PodSecurityPolicy
clusterrolebindings                            rbac.authorization.k8s.io      false        ClusterRoleBinding
clusterroles                                   rbac.authorization.k8s.io      false        ClusterRole
priorityclasses                   pc           scheduling.k8s.io              false        PriorityClass
storageclasses                    sc           storage.k8s.io                 false        StorageClass
volumeattachments                              storage.k8s.io                 false        VolumeAttachment
```

