---
date: 2019-02-25T17:00:00+08:00
title: kubectl的自动补全
menu:
  main:
    parent: "tool-kubectl"
weight: 1111
description : "kubectl的自动补全"
---

### Bash

> 备注1：有段时间没有验证bash的内容了，后面再验证和更新
>
> 备注2:  官方文档 https://kubernetes.io/zh/docs/tasks/tools/install-kubectl/ 下有非常详细的内容，可以对照一下。

执行下列命令，可以在bash下启动kubectl的自动命令补全，使用时特别的方便，强烈推荐：

```bash
source <(kubectl completion bash)
```

对于macos，则要复杂一些，因为macos下默认安装的bash版本太低：

```bash
$ echo $BASH_VERSION
3.2.57(1)-release
```

需要升级到新版本，执行命令 `brew install bash`，会安装最新版本，如`5.0.2(1)-release`。此时需要修改macos的默认shell为新版本的bash，打开文件 `/etc/shells`，在最后加入一行 `/usr/local/bin/bash` ，这个文件最后看起来会是这个样子：

```bash
/bin/bash
/bin/csh
/bin/ksh
/bin/sh
/bin/tcsh
/bin/zsh
/usr/local/bin/bash
```

然后执行命令   `chsh -s /usr/local/bin/bash`  修改，关闭当前终端，重新打开一个新的终端，检查是否修改成功：

```bash
$ echo $BASH_VERSION
5.0.2(1)-release
```

然后继续安装 bash-completion ，参考网上资料，方式为：

```bash
$ brew install bash-completion
$ brew tap homebrew/completions
```

报错：

```bash
$ brew tap homebrew/completions
Error: homebrew/completions was deprecated. This tap is now empty as all its formulae were migrated.
```

google了一下，解决方案是：

```bash
brew tap homebrew/homebrew-core
```

但是即时这样，执行 `source <(kubectl completion bash)` 之后，在 kubectl 做补全时，会报错：

```bash
kubectl ap-bash: _get_comp_words_by_ref：未找到命令
```

继续google，找到的方式是说要先执行 `source /etc/bash_completion`，即：

```bash
source /etc/bash_completion
source <(kubectl completion bash)
```

但是:

```bash
$ source /etc/bash_completion
-bash: /etc/bash_completion: No such file or directory
```

最后 google 到解决方案，要安装bash_completion@2:

```bash
brew unlink bash-completion
brew install bash-completion@2
```

然后遵循提示,  在 `~/.bash_profile` 中加入如下内容后，执行 `source ~/.bash_profile ` 重新装载:

```bash
[[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"
```

之后再执行`source <(kubectl completion bash)` ，就可以愉快的使用 kubectl 命令的自动补全了。

> 备注：自动补全功能没法和alias 命令一起用，即 alias k=kubectl 之后，用 k 命令就没法自动补全了。

为了方便，在每次打开终端时都可以有自动补全功能，需要在 `~/.bash_profile` 文件中加入一句：

```bash
source <(kubectl completion bash)
```

参考资料：

- [Fixing Bash autocompletion on MacOS for kubectl and KOPS](https://medium.com/merapar/fixing-bash-autocompletion-on-macos-for-kubectl-and-kops-e87f019652e8): 比较细致，但是bash-completion的安装过期了

- [kubectl bash completion doesn't work in ubuntu docker container](https://stackoverflow.com/questions/50406142/kubectl-bash-completion-doesnt-work-in-ubuntu-docker-container): 在回帖中看到mac下要安装bash-completion@2，终于搞定



### Zsh

> 备注：zsh的设置要复杂的多，以下内容仅在 macos 下验证过

macos + zsh下进行 kubectl 的自动补全，主要存在以下几个问题：

1. 执行 `source <(kubectl completion zsh)` 时报错 `command not found: compdef`

   解决的方式是在 `~/.zshrc` 文件的最前面（主要不要加在中间或者后面）增加：

   ```zsh
   autoload -Uz compinit
   compinit
   ```

2. 执行上面的  compinit 命令出现警告

   ```zsh
   % exec zsh
   zsh compinit: insecure directories, run compaudit for list.
   Ignore insecure directories and continue [y] or abort compinit [n]? 
   
   % compaudit
   There are insecure directories:
   /usr/local/share/zsh/site-functions
   /usr/local/share/zsh
   ```

   解决方式，执行命令：

   ```zsh
   sudo chmod -R 755 /usr/local/share/zsh/site-functions
   sudo chmod -R 755 /usr/local/share/zsh/
   ```


总结需要做的内容：

1. 修改 `~/.zshrc`, 为 kubectl 和 别名 k 开启自动完成

   ```zsh
   autoload -Uz compinit
   compinit
   
   # 如果上面两句没搞定，试试换成下面的这两句
   #autoload -U +X bashcompinit && bashcompinit
   #autoload -U +X compinit && compinit
   
   # 其他内容。。。。。。。
   
   # k8s auto complete
   source <(kubectl completion zsh)
   alias k=kubectl
   complete -F __start_kubectl k
   ```

   

2. 执行命令

   ```zsh
   sudo chmod -R 755 /usr/local/share/zsh/site-functions
   sudo chmod -R 755 /usr/local/share/zsh/
   
   exec zsh
   ```

   

参考资料：

- [zsh: command not found: {compinstall,compinit,compdef}](https://unix.stackexchange.com/questions/339954/zsh-command-not-found-compinstall-compinit-compdef)

- [command not found : complete when using bash completion](https://github.com/eddiezane/lunchy/issues/57#issuecomment-448588918): 这里的讨论“zsh have a bash completion script compatibility mode”，在最后给出了一个可行的方案

- [zsh compinit: insecure directories](https://stackoverflow.com/questions/13762280/zsh-compinit-insecure-directories)