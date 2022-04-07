---
title: "Kubernetes安装后配置"
linkTitle: "安装后配置"
weight: 2150
date: 2021-05-10
description: >
  Kubernetes安装后配置
---



## 配置 kubectl 自动完成

### zsh配置

mac默认使用zsh，为了实现 kubectl 的自动完成功能，需要在 `~/.zshrc` 中增加以下内容:

```bash
# 注意这一行要加在文件的最前面
autoload -Uz compinit && compinit -i

......

# k8s auto complete
source <(kubectl completion zsh)
alias k=kubectl
complete -F __start_kubectl k
```

同时为了使用方便，为 kubectl 增加了 k 的别名，同样也为 k 增加了 自动完成功能。

### 使用oh-my-zsh

在使用 oh-my-zsh 之后，会更加的简单（强烈推荐使用 oh-my-zsh ），只要在 oh-my-zsh 的 plugins 列表中增加 kubectl 即可。

然后，在 `~/.zshrc` 中增加以下内容:

```bash
# k8s auto complete
alias k=kubectl
complete -F __start_kubectl k
```

`source ~/.zshrc ` 之后即可使用，此时用 k 这个别名来执行 kubectl 命令时也可以实现自动完成，非常的方便。

## 显示正在使用的kubectl上下文

https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/kubectx

这个插件增加了 kubectx_prompt_info()函数。它显示正在使用的 kubectl context 的名称（`kubectl config current-context`）。

你可以用它来定制提示，并知道你是否在prod集群上.

使用方式为修改 `~/.zshrc`:

1. 在 plugins 中增加 "kubectx"
2. 增加一行 `RPS1='$(kubectx_prompt_info)'`

`source ~/.zshrc ` 之后即可生效，会在命令行的最右侧显示出kubectl context 的名称，默认情况下 `kubectl config current-context` 的输出是 "kubernetes-admin@kubernetes"。

如果需要更友好的显示，则可以将名字映射为可读性更强的标记，如 dev, stage, prod：

```properties
kubectx_mapping[kubernetes-admin@kubernetes]="dev"
```

备注: 在多个k8s环境下切换时应该很有用，后续有需要再研究。



## 从其他机器上操作k8s集群

如果k8s安装在本机，则相应的 `kubectl` 等命令行工具在安装过程中都在本地准备就绪，而且 `kubeadm init` 命令在安装完毕之后会提示：

```bash
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf
```

安装上面的提示操作之后，就可以在本地通过  `kubectl` 命令行工具操作安装的k8s集群。

如果我们希望从其他机器上方便的操作k8s集群，而不是限制要先ssh登录到安装k8s控制平面的机器上，则可以简单的在这台机器上安装kubectl并配置好kubeconf文件。

步骤如下：

1. 安装 kubectl：和前面的不走类似

2. 配置 kubeconf

    ```bash
    mkdir -p $HOME/.kube
    # 复制集群的config文件到这台机器
    cp -i /path/to/cluster/config $HOME/.kube/config
    ```

如果有多个k8s集群需要操作，则可以在执行 `kubectl` 命令时通过 `--kubeconfig` 参数指定要使用的 kubeconf 文件:

```bash
kubectl --kubeconfig /home/sky/.kube/skyserver get nodes
```

每次都输入 "--kubeconfig /home/sky/.kube/skyserver" 会很累，可以通过设置临时的环境变量来在当前终端下选择kubeconf文件，如：

```bash
export KUBECONFIG=$HOME/.kube/skyserver
k get nodes

# 不需要用时，关闭终端或者unset
unset KUBECONFIG
```

如果需要同时操作多个集群，需要在多个集群之间反复切换，则应该使用context来灵活切换，参考:

- [配置对多集群的访问](https://kubernetes.io/zh/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable)

## 取消docker和k8s的更新

通过 apt 方式安装的 docker 和 k8s，会在 apt upgrade 时自动升级到最新版本，这未必安全，通常也没有必要。

可以考虑取消docker和k8s的的 apt 更新，`cd /etc/apt/sources.list.d`，将 docker 和 k8s 的ppa配置文件内容用 "#" 注释掉就可以了。需要时可以重新打开。

