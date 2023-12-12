---
title: "Nix and Content Address"
date: 2023-12-08T16:34:44+08:00
draft: false
---

# Nix目前的构建和发布模型
## `derivation outputs` and `output paths`
Nix软件构建以Derivation为核心，Derivation有以下核心属性：
1. Derivation表示了软件的构建过程；
2. Derivation可以依赖其他Derivation；
3. Derivation执行后具有副作用，会在`/nix/store`下生成路径为`xxx-name.drv`的文件，其中xxx为该Derivation的唯一标识符，在任何电脑中都一致；
4. `xxx-name.drv`可以被进一步构建成产物，并且产物的路径存储在`xxx-name.drv`中，在构建前就可以知道，此处我们称为`outPath`；
5. `outPath`路径中也存在唯一标识，同一个Derrvation产生的`.drv`文件中的`outPath`一定相同；
6. 由于构建前就知道产物路径，引用某个derivation时，只要检查其中存储的`outPath`存不存在，就可以判定需不需要重复编译；
7. 不同机器可以通过共享`outPath`来共享构建缓存；

>> 这里存在两个目录，一个是`.drv`的目录，一个是`outPath`的目录，两者的路径中都有唯一标识符，且路径都根据Derivation的参数计算得来，`outPath`存储在`.drv`中。

## 构建过程
nix包管理的软件构建过程用伪码表示大致如下:
``` python
derivation = getDerivationFrom(nixpkgs, package_name)
drv = derivatoin()      # 此处存在副作用，derivation函数调用后会生成.drv文件
if checkOutPathExist(drv.outPath):
    pass                # 如果产物目录已经存在，就不用重复构建了
else:
    drv.build()         # drv.build()会将软件构建后放到drv.outPath中
```

binary cache引入后:
``` python
derivation = getDerivationFrom(nixpkgs, package_name)
drv = derivatoin()
if checkOutPathExist(drv.outPath):
    pass
else:
    if fetchOutPathFromBinaryCache(drv.outPath):
        pass        # 尝试从BinaryCache服务器中下载对应的包，放到outPath中
    else:
        drv.build()
```

# Input-Address Model
总体来说，上面我们描述的目前Nix的做法是将软件构建软件的过程（包括输入）抽象为一个哈希值，这个唯一哈希值可以代表软件的某个特定状态，然后用这个哈希值来索引构建后的软件。

在构建过程当中，任何输入的改变必然会导致输出的路径发生变化：
![input address model](input_address.svg)
![input address model](input_address_dependencies_change.svg)

## 过度重复构建
你可能已经意识到了一些不对，在正常的软件构建过程当中，输入的改变不一定会导致输出不同，比如说代码路径下面增加了一些文档，或者说构建流程发生了改变。

然而在Nix目前的构建模型中，输入的任何变化都会导致输出路径的变化，但现实当中这样往往会导致一些不好的副作用。

例如go语言的源码中使用perl来做代码单元测试，原则上来说，使用什么版本的perl做单元测试，或者是否做单元测试，都不会改变构建产物。我们可以认为，源码中不是所有改动都会影响产物。但在nix中，如果perl包发生了改变，而go又依赖perl做单元测试，那么go的产物路径就会发生变化，进而需要重新编译。再进一步，所有依赖go的包也需要重新编译，导致了很多没有必要的重复编译。

# Content-Address Model
解决上述问题的方式思路也很简单，就是使用content address的方式来确定构建缓存的实际目录。

所谓content address，与input address相对。在input address的模型中，用于索引value的key与value内容无关；
但在content address的模型中，用于索引value的key是根据value的内容计算得来。content address最简单的例子就是使用文件的哈希值来索引文件，给定文件的哈希值，返回文件内容；
而input address典型例子就是给定文件的文件名，返回文件的内容；


content address最经典的应用场景莫过于git，git本质上就是一个[content addressable filesystem](https://linianhui.github.io/git/content-addressable-filesystem/)


早在nix最开始的[260多页的论文](https://edolstra.github.io/pubs/phd-thesis.pdf#page=143)中，作者就提到了content address的方案，并且例举了若干优点
包括信任模型的简化、重复构建的避免等等。

更进一步的，基于content address的模型，nix未来甚至可以引入例如IPFS之类的存储后端，从而做到分布式缓存，这也是nix项目下一步的主要方向。目前nix初步实现了Content Address Derivation，但还在早起阶段，hydra并没有进行默认集成。另外git hashing的支持也在路上，目前也有了初步实现。详细内容可以看下面的参考文章；

# References
## Nix Content Address Implementation
[TOWARDS A CONTENT-ADDRESSED MODEL FOR NIX](https://www.tweag.io/blog/2020-09-10-nix-cas/)

[SELF-REFERENCES IN A CONTENT-ADDRESSED NIX](https://www.tweag.io/blog/2020-11-18-nix-cas-self-references/)

[DERIVATION OUTPUTS IN A CONTENT-ADDRESSED WORLD](https://www.tweag.io/blog/2021-02-17-derivation-outputs-and-output-paths/)

[IMPLEMENTING A CONTENT-ADDRESSED NIX](https://www.tweag.io/blog/2021-12-02-nix-cas-4/)

## Nix And IPFS
[ipfs-nix-guide](https://github.com/obsidiansystems/ipfs-nix-guide)
## RFCS
[0062-content-addressed-paths.md](https://github.com/NixOS/rfcs/blob/master/rfcs/0062-content-addressed-paths.md)

[0133-git-hashing.md](https://github.com/NixOS/rfcs/blob/master/rfcs/0133-git-hashing.md)

