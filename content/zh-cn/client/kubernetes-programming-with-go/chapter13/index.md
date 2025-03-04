---
title: "[第13章]使用 kubebuilder 创建 Operator"
linkTitle: "使用 kubebuilder 创建 Operator"
weight: 130
date: 2021-02-01
description: >
  使用 kubebuilder 创建 Operator
---

在前面的章节中，你已经看到了如何使用自定义资源定义（CRD）来定义由 API 服务器提供的新资源，以及如何使用 controller-runtime 库来构建操作器。

Kubebuilder SDK 致力于帮助你创建新的资源和相关的 Operators。它提供的命令可以启动一个定义管理器的项目，并将资源和它们相关的控制器添加到项目中。

一旦生成了新的自定义资源和控制器的源代码，你将需要根据资源的业务领域来实现缺少的部分。然后，Kubebuilder SDK 提供工具来构建和部署自定义资源定义和管理器到集群上。

## 安装Kubebuilder

Kubebuilder 是以一个单一的二进制文件提供的。你可以从项目的发布页面下载二进制文件，并将其安装到你的PATH中。二进制文件被提供给 Linux 和 MacOS 系统。

### 创建项目

第一步是创建项目。该项目最初将包含：

- 定义管理器的Go源代码（目前没有任何控制器）
- Docker文件，用于构建包含管理器二进制文件的镜像，以部署到集群上

- Kubernetes 清单，以帮助将管理器部署到集群中。

- 一个定义命令的Makefile，以帮助你测试、构建和部署管理器。


要创建项目，首先创建一个空目录， cd 进入这个目录，然后执行以下 `kubebuilder init` 命令：

```bash
$ mkdir myresource-kb
$ cd myresource-kb
$ kubebuilder init
     --domain myid.dev                          ❶
     --repo github.com/myid/myresource          ❷
```

- ❶ 域名，作为 GVK 分组名称的后缀使用。在这个项目中定义的自定义资源可以属于不同的分组，但所有分组将属于同一个域。例如，`mygroup1.myid.dev `和 `mygroup2.myid.dev`

- ❷ 用于生成管理器 Go 代码的 Go 模块的名称


你可以通过运行以下命令检查 Makefile 中的可用命令：

```go
$ make help
```

你可以用以下命令为管理器构建二进制文件：

```bash
$ make build
```

然后，你可以在本地运行管理器：

```go
$ make run
```

或者

```go
$ ./bin/manager
```

在这一点上，启动一个源码控制项目（例如，一个git项目），并使用生成的文件创建第一个修订版是很有意思的。这样，你将能够检查接下来执行的 kubebuilder 命令所做的修改。例如，如果你使用git：

```bash
$ git init
$ git commit -am 'kubebuilder init --domain myid.dev --repo github.com/myid/myresource'
```

### 在项目中添加自定义资源

就目前而言，管理器并不管理任何控制器。即使你能构建和运行它，你也不能对它做任何事情。

下一个要执行的 kubebuilder 命令是 kubebuilder create api，以添加一个自定义资源和它相关的控制器到项目中。该命令会问你是否要创建资源和控制器。对每个问题回答Y。

```bash
$ kubebuilder create api
     --group mygroup                   ❶
     --version v1alpha1                ❷
     --kind MyResource                 ❸
Create Resource [y/n]
y
Create Controller [y/n]
y
```

- ❶ 自定义资源的分组。它将以域名为后缀，形成完整的 GVK 分组。

- ❷ 该资源的版本

- ❸ 资源的种类/kind


你可以通过运行下面的 git 命令看到该命令所做的修改：

```bash
$ git status
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
     modified:   PROJECT
     modified:   go.mod
     modified:   go.sum
     modified:   main.go
Untracked files:
  (use "git add <file>..." to include in what will be committed)
     api/
     config/crd/
     config/rbac/myresource_editor_role.yaml
     config/rbac/myresource_viewer_role.yaml
     config/samples/
     controllers/
no changes added to commit (use "git add" and/or "git commit -a")
```

**PROJECT** 文件包含项目的定义。它最初包含了作为标志提供给 init 命令的域名和 repo。现在它也包含了第一个资源的定义。main.go 文件和 controllers 目录为自定义资源定义了一个控制器。

`api/v1alpha1` 目录已经创建，包含了使用 Go 结构体的自定义资源的定义，以及由 deepcopy-gen（见第8章 "运行deepcopy-gen" 部分）在 controller-gen 工具的帮助下生成的代码。它还包含了 AddToScheme 函数的定义，对于将这种新的自定义资源添加到 Scheme 中非常有用。

`config/samples` 目录包含一个新文件，定义了 YAML 格式的自定义资源的实例。`config/rbac` 目录包含两个新文件，定义了两个新的 ClusterRole 资源，一个用于查看，一个用于编辑 MyResource 实例。`config/crd` 目录包含用于构建 CRD 的 kustomize 文件。

### 添加RBAC注解

在本地运行 operator 时，operator 会使用您的 kubeconfig 文件，以及该 kubeconfig 的特定授权。如果您使用集群管理员账户进行连接，操作员就会拥有集群上的所有授权，并能进行任何操作。

然而，当 operator 被部署在集群上时，它是用一个特定的 Kubernetes Service Account 运行的，并被赋予有限的授权。这些授权是由 kubebuilder 构建和部署的 ClusterRole 中定义的。

为了帮助 Kubebuilder 构建这个 ClusterRole，在 Reconcile 函数的生成注释中出现了注解（为了清晰起见，加入了换行符）：

```go
//+kubebuilder:rbac:
     groups=mygroup.myid.dev,
     resources=myresources,
     verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:
     groups=mygroup.myid.dev,
     resources=myresources/status,
     verbs=get;update;patch
//+kubebuilder:rbac:
     groups=mygroup.myid.dev,
     resources=myresources/finalizers,
     verbs=update
```

这些规则将给予对 MyResource 资源的完全访问权，但没有对其他资源的访问权。

Reconcile 函数需要在观察 deployment 资源时拥有对它的读写权限；为此，你需要添加这个新的注解（已添加换行符）：

```go
//+kubebuilder:rbac:
     groups=apps,
     resources=deployments,
     verbs=get;list;watch;create;update;patch;delete
```

## 在群集上部署 operator

为了能够将 operator 部署到集群中，您需要构建容器镜像，并将其部署到容器镜像登记处（如DockerHub、Quay.io或其他许多地方）。

第一步是在您喜欢的容器镜像登记处创建一个新的存储库，包含 operator 容器的镜像。假设您在 `quay.io/myid` 中创建了一个名为 myresource 的存储库，镜像的全名将是 `quay.io/myid/myresource`。

在每次构建时，你都需要为镜像使用不同的标签，这样才能正确地更新容器。要在本地构建镜像，你需要运行以下命令（注意标签 v1alpha1-1）：

```go
$ make docker-build
     IMG=quay.io/myid/myresource:v1alpha1-1
```

要将其部署到注册表，必须执行以下命令：

```go
$ make docker-push
     IMG=quay.io/myid/myresource:v1alpha1-1
```

最后，将 Operator 部署到集群中：

```go
$ make deploy
    IMG=quay.io/myid/myresource:v1alpha1-1
```

这将创建一个新的命名空间，myresource-kb-system，其中包含一个新的 deployment，将执行 Operator。您可以用以下命令检查 Operator 的日志：
