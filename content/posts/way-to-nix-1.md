---
title: "Way to Nix 1"
date: 2023-12-01T11:56:15+08:00
draft: false
---

最近学习了一些Nix相关的东西，把家里大多数机器都切到了nixos上，感觉很不错。但是由于nix文档比较差，写自己配置的时候，很多时候需要看代码，或者抄别人的配置一点点凑，过程当中遇到了不少问题。于是记录一下相关的概念和技术以免忘记。

前阵子看过NickCao的前些年的一个视频，非常有条例的介绍了一些nix的基本概念，有兴趣可以去看一下：[【金枪鱼之夜：Nix - 从构建系统到配置管理-哔哩哔哩】](https://b23.tv/vxaiTuS)

# Nix as a build systems
构建系统一般就是指从源码生成产物的一套工具生态集合，就拿Makefile举例，几乎所有的构建过程都可以描述成如下形式的嵌套结构：
``` Makefile
构建目标: 构建原料
    构建过程
```
围绕着gnu make，有一套完整的，有着悠久历史的构建系统，如GNU Autoconf，Cmake等，许多现代语言也包含了自己的构建系统，如rust的cargo，go的build，js的npm等等。也会有许多项目选择多种构建系统嵌套，如Cmake生成makefile调用Cargo来进行项目构建。

## Derivation
也许nix里最重要的概念就是[Derivation](https://nixos.org/manual/nix/stable/language/derivations.html)了，它与nix中的很多概念相互关联，并作为纽带。可以说理解了Derivation，至少就理解了Nix的半壁江山。

首先， derivition是nix中的一个内置函数,代码实现位于(https://github.com/NixOS/nix/blob/188c803ddb5e63b243ddb84eba9b70e45475b7ea/src/libexpr/primops/derivation.nix#L2)，实际调用的是内部C++函数[prim_derivationStrict](https://github.com/NixOS/nix/blob/188c803ddb5e63b243ddb84eba9b70e45475b7ea/src/libexpr/primops.cc#L1041):
``` repl
nix-repl> derivation
«lambda @ /builtin/derivation.nix:5:1»
```

与其他构建系统类似，derivation包含了构建原料，构建目标以及构建过程，调用这个函数必须有`name`、`system`以及`builder`三个必选参数，其他可选参数可参考[Nix Reference Manual](https://nixos.org/manual/nix/stable/language/derivations.html#optional):
```
nix-repl> derivation {name = "target_name";system="x86_64-linux"; builder="builder_binary";}
«derivation /nix/store/6z9jj5khn7j3xi2fv8fibpzj6gnq4iz4-target_name.drv»
```
值得注意的是，derivation函数存在一个副作用，即在/nix/store/目录下生成一个以`.drv`为后缀的文件，并且文件名中包含了一个类似哈希的字符串，保证路径的唯一性。derivation有如下属性:


- 同样参数的derivation函数调用后生成的文件，路径一定是一样的（*在哈希算法的保证下，可以假定不同参数调用derivation产生的drv文件路径一定是不同的*）
- derivation中包含的所有参数必须为字面量，或预先可以确定哈希值的固定内容（**Fixed-Output Derivations**）
- derivation可以互相依赖，一个derivation可以依赖另外一个derivation
- derivation可以被执行，类似Makefile一样可以被构建
- derivation会被在隔离的环境执行，其中没有类似`/bin/bash`之类可以预先假定存在的路径，能且只能通过derivation依赖的方式在构建过程当中引入其他软件
- derivation执行后会产生两个关键参数，out.out以及out.outPath，一个存储了drv的路径，一个存储了构建产物的路径

```
nix-repl> d = derivation {name = "target_name";system="x86_64-linux"; builder="builder_binary";}

nix-repl> d.out.out
«derivation /nix/store/6z9jj5khn7j3xi2fv8fibpzj6gnq4iz4-target_name.drv»

nix-repl> d.out.outPath
"/nix/store/pd5l9rzb613v5lv4c6q2m0c81zd9w3l6-target_name"

nix-repl> d.out.out == d
true

nix-repl> d.out == d
true
```

以上属性组合的结果，使得nix构建系统存在如下特性：
1. 任何软件的derivation在/nix/store中的路径包含了唯一哈希值，这个路径实际上代表了产物是用`何种输入源`，以`何种构建方式`，配置`何种构建参数`所构建出来的结果，且由于derivation之间互相依赖，整颗依赖树的任何一个环节发生了变化，重新构建后，derivation路径相应也会发生变化


# Nix as programing language
# Nix as a package manager
# Nix到底解决了什么问题




