---
title: "叛逃"
date: 2022-08-07T12:52:43+08:00
draft: false
tags: 
    - MacOS
---
# 所谓叛逃
仔细想一想，自从上大学开始，我用linux做为日常工作和学习的系统也有将近六年了。
这六年来，常见的Linux发行版基本都装了个遍，常见的桌面环境和窗口管理器也基本用了个遍。
不过随着时间流逝，我逐渐发现，桌面环境折腾无数遍，其实实际在用的图形界面软件只有三个：`emacs`、`chrome`、`alacritty`。

更抽象一点的话，其实就是一个编辑器，一个浏览器还有一个终端模拟器。其他软件使用频率都非常低，最近两年内，我逐渐发现自己打开文件管理器的频率也越来越低，以至于文件管理器对于我日常的电脑使用来说，也成为了一个没有必要的软件。

铺垫了这么多，其实我就是想说，爷叛逃了，到MacOS。

# 目的地
其实也没啥道理，最近想买一个笔记本，看了一圈。intel 11代摆大烂，12代功耗爆炸；AMD 6000系的笔记本，续航稍微好一些，但GPU硬件设计有问题。仔细想了一想，苹果的笔记本成了这两年唯一没重大问题，没开过什么倒车的设备了。

所以，Macbook Air m2 16+512，请：
![neofetch](2022-08-07_13.23.10.png)

# 基本配置
## 终端应用
```sh
brew install p7zip starship rustup-init tmux go coreutils grep startship bottom zoxide fzf ripgrep shellcheck starship hugo fd 7z pandoc neofetch
```
大概先装这么多吧，zsh配置后续再迁移过来
## GUI
> 写这篇文章的时候突然恍然大悟，原来我能在这台电脑上装qq和微信，突然间一股惆怅的情绪涌上心头

其实也没啥好装的，浏览器暂时看看safari表现怎么样了，之后如果不成的话再装chrome。

文本编辑器的话，还是当之无愧的emacs。虽然在macos上的这个版本没有native compile，但是感觉响应速度没有什么明显区别：
```sh
brew install --cask emacs
```
