---
title: "Manager"
linkTitle: "Manager"
weight: 20
date: 2021-02-01
description: >
  Manager
---

controller-runtime 库提供的第一个重要抽象是 Manager / 管理器，它为在管理器内运行的所有控制器提供共享资源，包括：

- 读取和写入 Kubernetes 资源的 Kubernetes 客户端
- 用于从本地缓存中读取 Kubernetes 资源的缓存

- 用于注册所有 Kubernetes 本地和自定义资源的 scheme


要创建管理器，你需要使用提供的 **New** 函数，如下所示：

```go
import (
      "flag"
      "sigs.k8s.io/controller-runtime/pkg/client/config"
      "sigs.k8s.io/controller-runtime/pkg/manager"
)
flag.Parse()                        ❶
mgr, err := manager.New(
     config.GetConfigOrDie(),
      manager.Options{},
)
```

- ❶解析命令行标志，如 GetConfigOrDie 来处理 `--kubeconfig` 标志；见下文。

  第一个参数是 rest.Config 对象，见第 6 章 "连接到集群" 部分。请注意，在这个例子中，选择了 controller-runtime 库提供的 GetConfigOrDie() 实用函数，而不是使用 client-go 库的函数。

GetConfigOrDie() 函数将尝试获得一个配置来连接到集群：

- 通过获取 `--kubeconfig` 标志的值，如果定义了的话，并在这个路径上读取 kubeconfig 文件。为此，首先你需要执行 `flag.Parse()`
- 通过获取 KUBECONFIG 环境变量的值（如果定义了的话），并读取此路径下的 kubeconfig 文件

- 通过查看  in-cluster 配置（见第6章的 "集群内配置 "部分），如果定义了的话

- 通过读取 `$HOME/.kube/config` 文件

如果前面的情况都不可行，该函数将使程序退出，代码为1。第二个参数是一个用于选项的结构体。

一个重要的选项是 "**Scheme**"。默认情况下，如果你没有为这个选项指定任何值，将使用 Client-go 库提供的 **Scheme**。如果控制器只需要访问本地 Kubernetes 资源，这就足够了。然而，如果你想让控制器访问自定义资源，你将需要提供一个能够解决自定义资源的 **Scheme**。

例如，如果你想让控制器访问第九章中定义的自定义资源，你将需要在初始化时运行以下代码：

```go
import (
      "k8s.io/apimachinery/pkg/runtime"
      clientgoscheme "k8s.io/client-go/kubernetes/scheme"
      mygroupv1alpha1 "github.com/myid/myresource-crd/pkg/apis/mygroup.example.com/v1alpha1"
)
scheme := runtime.NewScheme()                  ❶
clientgoscheme.AddToScheme(scheme)             ❷
mygroupv1alpha1.AddToScheme(scheme)            ❸
mgr, err := manager.New(
      config.GetConfigOrDie(),
      manager.Options{
            Scheme: scheme,                    ❹
      },
)
```

❶ 创建一个新的空 scheme

❷ 使用 Client-go 库添加本地 Kubernetes 资源

❷ 将 mygroup/v1alpha1 的资源添加到 scheme 中，其中包含我们的自定义资源

❹ 在这个管理器中使用这个 scheme
