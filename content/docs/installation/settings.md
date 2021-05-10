---
title: "Kubernetes安装后配置"
linkTitle: "安装后配置"
weight: 299
date: 2021-05-10
description: >
  Kubernetes安装后配置
---



## 配置 kubectl 自动完成

### mac下zsh配置

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

在使用 oh-my-zsh 之后，会更加的简单（强烈推荐使用 oh-my-zsh ），只要在 oh-my-zsh 的 plugins 列表中增加 kubectl 即可。







