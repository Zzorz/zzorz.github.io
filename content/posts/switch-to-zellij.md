---
title: "Switch to Zellij"
date: 2022-08-06T10:38:43+08:00
draft: true
tags:
    - utils
    - zellij
---

# What?
Zellij是一个使用rust编写的终端复用工具，类似tmux或screen，github大约有7k的star。毕竟，有谁不喜欢rust写的东西呢。

此外，这个软件看上去，结构貌似是经过精心设计的，其中比较值得一说的，应该是他选择了wasm作为插件系统的核心，这样就可以使用任何一个能够编译到wasm的语言来编写插件。

最近讨论较多的vim9插件语言性能提升的问题，现代的软件只要引入一个wasm中间层，从最开始就不会有性能问题的顾虑。这可能就是软件工业的进步吧，感觉llvm思路带来的影响可能还会继续影响其他领域。毕竟，有什么问题是加一个中间层解决不了的事情呢。

作为一个emacs用户，也不禁遐想，要是以emacs为范本，有一个不是用eslip驱动，而是wasm驱动的文本编辑器，是不是现在emacs的目前的困境从新软件设计开始就会不存在了呢。

那么接下来，就尝试切换到zellij上吧

# 安装和基本使用
不废话：
```BASH
cargo-binstall zellij
```

他的快捷键风格类似于emacs，但作为终端应用有些反人类。比如说ctrl+p ctrl+n本来应该是↑和 ↓ ，结果在应用里面被劫持成了PANE和RESIZE的prefix：

![zellij](2022-08-06_12-00.png)

# 重新进行快捷键映射

