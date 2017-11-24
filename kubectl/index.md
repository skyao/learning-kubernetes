# kubectl

kubectl是官方提供的客户端工具，可直接以命令行的方式同集群交互。

## 资料

- 官方文档
	- [Overview of kubectl](https://kubernetes.io/docs/user-guide/kubectl-overview/)
	- [kubectl](https://kubernetes.io/docs/user-guide/kubectl/)
	- [v1.8 Commands](https://kubernetes.io/docs/user-guide/kubectl/v1.8/)
	- [v1.7 Commands](https://kubernetes.io/docs/user-guide/kubectl/v1.7/)
	- [Kubectl Cheatsheet](https://kubernetes.io/docs/user-guide/kubectl-cheatsheet/)
- [Kubectl Cheatsheet 中文版本](https://www.tuicool.com/articles/qm2A3qJ)

## 用法

### 自动补全

执行下列命令，可以在bash下启动kubectl的自动命令补全，使用时特别的方便，强烈推荐：

```bash
source <(kubectl completion bash)
```

方便起见，可以直接修改`/etc/profile`，加入上面的命令。




