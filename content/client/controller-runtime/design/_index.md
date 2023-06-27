---
title: "controller-runtime设计"
linkTitle: "设计"
weight: 90
date: 2021-02-01
description: >
  
---

> https://github.com/kubernetes-sigs/controller-runtime/tree/main/designs



这些是对 Controller Runtime 进行修改的设计文件。它们的存在是为了帮助记录编写 Controller Runtime 的设计过程，但可能不是最新的（更多内容见下文）。

并非所有对 Controller Runtime 的修改都需要设计文档--只有重大的修改才需要。使用你的最佳判断。

当提交设计文件时，我们鼓励你写概念验证，而且完全可以在提交设计文件的同时提交概念验证PR，因为概念验证过程可以帮助消除困惑，并且可以帮助模板中的示例部分。

## 过时的设计

Controller Runtime 文档 GoDoc 应该被认为是 Controller Runtime 的经典的、最新的参考和架构文档。

然而，如果你看到一个过时的设计文档，请随时提交一个PR，将其标记为这样，并添加一个链接到问题的附录，记录事情的变化原因。比如说：
