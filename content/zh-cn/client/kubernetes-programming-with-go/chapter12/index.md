---
title: "[第12章]测试 Reconcile 循环"
linkTitle: "测试 Reconcile 循环"
weight: 120
date: 2021-02-01
description: >
  测试 Reconcile 循环
---

上一章描述了如何为 Operator 调和自定义资源实现一个简单但完整的 Reconcile 函数。

为了测试你在上一章编写的 Reconcile 函数，你将使用 ginkgo，它是Go的一个测试框架；以及 controller-runtime 库中的e nvtest 包，它提供了一个 Kubernetes 环境用于测试。

## envtest 包

Controller-runtime 库提供了 envtest 包。这个包通过启动简单的本地控制平面来提供 Kubernetes 环境。

默认情况下，该包使用位于 `/usr/local/kubebuilder/bin` 的 `etcd` 和 `kube-apiserver` 的本地二进制文件，你也可以提供自己的路径来找到这些二进制文件。你可以安装 `setup-envtest` 来获得不同版本的 Kubernetes 的这些二进制文件。

### 安装envtest二进制文件

setup-envtest 工具可以用来安装 envtest 使用的二进制文件。要安装该工具，你必须运行：

```go
$ go install sigs.k8s.io/controller-runtime/tools/setup-envtest@latest
```

然后，你可以使用以下命令为特定的 Kubernetes 版本安装二进制文件：

```go
$ setup-envtest use 1.23
Version: 1.23.5
OS/Arch: linux/amd64
Path: /path/to/kubebuilder-envtest/k8s/1.23.5-linux-amd64
```

该命令的输出将告诉你二进制文件被安装在哪个目录下。如果你想从默认目录 `/usr/local/kubebuilder/bin` 中使用这些二进制文件，你可以创建一个符号链接，从那里访问它们：

```go
$ sudo mkdir /usr/local/kubebuilder
$ sudo ln -s /path/to/kubebuilder-envtest/k8s/1.23.5-linux-amd64 /usr/local/kubebuilder/bin
```

或者，如果你喜欢使用 KUBEBUILDER_ASSETS 环境变量来定义包含二进制文件的目录，你可以执行：

```bash
$ source <(setup-envtest use -i -p env 1.23.5)
$ echo $KUBEBUILDER_ASSETS
/path/to/kubebuilder-envtest/k8s/1.23.5-linux-amd64
```

这将定义并导出 KUBEBUILDER_ASSETS 变量，其路径包含 Kubernetes 1.23.5 的二进制文件。

### 使用envtest

控制平面将只运行 API 服务器和 etcd，但没有控制器。这意味着，当你想测试的 operator 将创建 Kubernetes 资源时，没有控制器会做出反应。例如，如果 operator 创建了 Deployment ，不会有 pod 被创建，而且 Deployment 状态也不会被更新。

这在一开始可能会令人惊讶，但这将有助于你只测试你的 operator，而不是 Kubernetes 控制器。为了创建测试环境，你首先需要创建一个 `envtest.Environment` 结构体的实例。

使用环境结构的默认值将启动一个本地控制平面，使用 `/usr/local/kubebuilder/bin` 中的二进制文件或来自 KUBEBUILDER_ASSETS 环境变量中定义的目录。

如果你正在编写 operator 调和自定义资源，你将需要为这个自定义资源添加自定义资源定义（CRD）。为此，你可以使用 CRDDirectoryPaths 字段来传递包含  YAML 或 JSON 格式的 CRD 定义的目录列表。所有这些定义将在初始化环境时被应用到本地集群。

如果你想在 CRD 目录不存在时被改变，ErrorIfCRDPathMissing 字段很有用。作为一个例子，下面是如何创建一个 **Environment** 结构体，CRD YAML或JSON文件位于 `././crd` 目录下：

```go
import (
  "path/filepath"
  "sigs.k8s.io/controller-runtime/pkg/envtest"
)
testEnv = &envtest.Environment{
  CRDDirectoryPaths:     []string{
    filepath.Join("..", "..", "crd"),
  },
  ErrorIfCRDPathMissing: true,
}
```

要启动 environment ，你可以使用该 Environment 的 Start 方法：

```go
cfg, err := testEnv.Start()
```

这个方法返回一个 `rest.Config` 值，这是用来连接到由环境启动的本地集群的 Config 值。在测试结束时，可以使用 Stop 方法停止 Environment：

```go
err := testEnv.Stop()
```

一旦 Environment 启动，并且你有一个 Config (配置)，你就可以创建管理器和控制器，并启动管理器，如第10章所述。

### 定义 ginkgo 套件

为了启动 Reconcile 函数的测试，你可以使用 `go test` 功能来启动 ginkgo specs：

```go
import (
      "testing"
     . "github.com/onsi/ginkgo/v2"
     . "github.com/onsi/gomega"
)
func TestMyReconciler_Reconcile(t *testing.T) {
     RegisterFailHandler(Fail)
     RunSpecs(t,
          "Controller Suite",
     )
}
```

然后，你可以声明 BeforeSuite 和 AfterSuite 函数，它们用于启动/停止环境和管理器。

下面是这些函数的例子，它将创建一个加载了 MyResource 的 CRD 的环境。在 BeforeSuite 函数的末尾，Manager 在 Go 例程中被启动，因此测试可以在主 Go 例程中执行。

注意，你正在创建一个可取消的上下文，在启动管理器时使用，所以你可以通过取消 AfterSuite 函数的上下文来停止管理器。

```go
import (
     "context"
     "path/filepath"
     "testing"
     . "github.com/onsi/ginkgo/v2"
     . "github.com/onsi/gomega"
     appsv1 "k8s.io/api/apps/v1"
     "k8s.io/apimachinery/pkg/runtime"
     clientgoscheme "k8s.io/client-go/kubernetes/scheme"
     "sigs.k8s.io/controller-runtime/pkg/builder"
     "sigs.k8s.io/controller-runtime/pkg/client"
     "sigs.k8s.io/controller-runtime/pkg/envtest"
     "sigs.k8s.io/controller-runtime/pkg/log"
     "sigs.k8s.io/controller-runtime/pkg/log/zap"
     "sigs.k8s.io/controller-runtime/pkg/manager"
     mygroupv1alpha1 "github.com/feloy/myresource-crd/pkg/apis/mygroup.example.com/v1alpha1"
)
var (
     testEnv   *envtest.Environment                        ❶
     ctx       context.Context
     cancel    context.CancelFunc
     k8sClient client.Client                               ❷
)
var _ = BeforeSuite(func() {
     log.SetLogger(zap.New(
          zap.WriteTo(GinkgoWriter),
          zap.UseDevMode(true),
     ))
     ctx, cancel = context.WithCancel(                     ❸
          context.Background(),
     )
     testEnv = &envtest.Environment{                       ❹
          CRDDirectoryPaths:     []string{
               filepath.Join("..", "..", "crd"),
          },
          ErrorIfCRDPathMissing: true,
     }
     var err error
     // cfg is defined in this file globally.
     cfg, err := testEnv.Start()                           ❺
     Expect(err).NotTo(HaveOccurred())
     Expect(cfg).NotTo(BeNil())
     scheme := runtime.NewScheme()                         ❻
     err = clientgoscheme.AddToScheme(scheme)
     Expect(err).NotTo(HaveOccurred())
     err = mygroupv1alpha1.AddToScheme(scheme)
     Expect(err).NotTo(HaveOccurred())
     mgr, err := manager.New(cfg, manager.Options{         ❼
          Scheme: scheme,
     })
     Expect(err).ToNot(HaveOccurred())
     k8sClient = mgr.GetClient()                           ❽
     err = builder.                                        ❾
          ControllerManagedBy(mgr).
          Named(Name).
          For(&mygroupv1alpha1.MyResource{}).
          Owns(&appsv1.Deployment{}).
          Complete(&MyReconciler{})
     go func() {
          defer GinkgoRecover()
          err = mgr.Start(ctx)                             ❿
          Expect(err).ToNot(
               HaveOccurred(),
               "failed to run manager",
)
     }()
})
var _ = AfterSuite(func() {
     cancel()                                              ⓫
     err := testEnv.Stop()                                 ⓬
     Expect(err).NotTo(HaveOccurred())
})
```

TBD

## 结语

本章结束了对可用于编写 Kubernetes Operators 的各种概念和库的介绍。

第8章介绍了自定义资源，允许通过在服务资源列表中添加新资源来扩展 Kubernetes API。第9章介绍了使用Go语言处理自定义资源的各种方法，可以通过为该资源生成 clientset，也可以使用 DynamicClient。

第10章介绍了 controller-runtime 库，对于实现管理自定义资源生命周期的 Operator 很有用。第11章着重于编写 Operator 的业务逻辑，第12章用来测试这个逻辑。

下一章介绍了 kubebuilder SDK，这是一个使用前几章中介绍的工具的框架。这个框架通过生成新的自定义资源定义和相关 Operator 的代码，以及提供构建和将这些自定义资源定义和 Operator 部署到集群的工具，促进了 Operator 的开发。
