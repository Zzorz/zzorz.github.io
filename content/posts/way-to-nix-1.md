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
2. derivation既可以描述软件的构建过程，又可以描述多种软件组合称为操作系统的过程，进一步说，如果把操作系统当成目标产物，通过derivation的嵌套，nix可以描述出整个操作系统的构建过程，且在保证可完全可复现。

# Nix as programing language
与其他构建系统采用的语法风格不同的是，Nix采用了函数式语法，且没有类型系统，这也是Nix看起来比较吓人的主要原因

有关函数式编程的基本知识，其实也不需要有多复杂，毕竟nix用途也不是什么通用的编程语言，就当作Makefile写或者Cmake来写就成。知道lambda表达式，柯里化之类的基础知识应该就成(大概)

对于nix的语法，详细内容可以参考[Nix Language](https://nixos.org/manual/nix/stable/language/index.html)

而derivation函数，也在语言层面上有更高级的封装，避免了手动写derivation依赖的麻烦，在实际nix构建derivation时，更多使用的是nixpkgs中提供的stdenv.mkDerivation函数，其中stdenv类似于debian或者ubuntu中的build-essential包，包含了一些构建软件所需要的常用依赖：
```
nix-repl> nixpkgs = import <nixpkgs> {}

nix-repl> nixpkgs.stdenv
«derivation /nix/store/0m92zn8yv2z6zw29y60iwmzwwr05y0yl-stdenv-linux.drv»

nix-repl> nixpkgs.stdenv.mkDerivation
«lambda @ /nix/store/d80iz3mpcdzwp8r2dw7j9d7dfw2wh5cm-nixpkgs/nixpkgs/pkgs/stdenv/generic/make-derivation.nix:559:3»
```

# Nix as a package manager
## 构建时依赖
构建时依赖相对简单，这个也是Nix作为构建系统时最基本的能力，所有构建过程当中存在的依赖关系都可以通过derivation的依赖关系来描述。nix-store -q --references 可以查询drv文件的构建时依赖，其实就是derivation依赖其他derivation列表

```
❯ nix-store -q --references /nix/store/0m92zn8yv2z6zw29y60iwmzwwr05y0yl-stdenv-linux.drv
/nix/store/ks6kir3vky8mb8zqpfhchwasn0rv1ix6-bootstrap-tools.drv
/nix/store/2cglag9gjpryls3lvx4k0l0xwy3sjw2w-ed-1.19.drv
/nix/store/13m1zpcsr1y5ns182rdak7y8iwf0qy9h-patch-2.7.6.drv
/nix/store/mlg49kdfzri70jjl8vnkk8kjfjpf9yam-zlib-1.3.drv
/nix/store/1m4wnjjawsppbv1j4xrbwqiv0snw87cv-binutils-2.40.drv
/nix/store/7nmmsk1fsm195fypb54pljywxg2jsk65-gnu-config-2023-07-31.drv
......

```
## 软件分发
从本质上来说，Nix是类似gentoo一样基于源代码的包管理，Nix的软件仓库其实就是结构组织良好的代码库，描述了所有软件构建过程，具体来说就是[nixpkgs](https://github.com/NixOS/nixpkgs)

引入一个软件源，实际上是引入了一套nix代码库，在尝试安装代码库中某一个软件时，只要执行对应代码，就可以将软件编译到本机固定目录下并配置好。

但nix也支持编译缓存（Binary Cache），由于在代码执行的结果是`.drv`文件，且理论上，所有使用nix包管理的用户，在使用同一个库中某软件代码后，执行出来的`.drv`文件目录是确定的，而且对于某个固定的`.drv`文件来说，构建出来的产物路径也是在编译前就确定的。

所以对于nix来说，基于二进制的包发行，理论上只需要建立一个维护着`*.drv`文件中`outPath`与下载地址之间的对应关系即可。在安装时就可以省去构建过程，直接用`outPath`去缓存服务器中下载实现编译好的二进制文件放在对应目录下即可。而实际上nix也是这样操作的，nix会将`*.drv`构建出来的目标路径进行打包压缩(NAR格式)，并在Binary Cache中提供下载，以stdenv为例：
``` bash
❯ nix derivation show /nix/store/0m92zn8yv2z6zw29y60iwmzwwr05y0yl-stdenv-linux.drv  | jq '.[].outputs.out.path'
"/nix/store/bbxdw4rgwwl3gnajri82yidr1nlsfskf-stdenv-linux"

❯ curl https://cache.nixos.org/bbxdw4rgwwl3gnajri82yidr1nlsfskf.narinfo
StorePath: /nix/store/bbxdw4rgwwl3gnajri82yidr1nlsfskf-stdenv-linux
URL: nar/187vlg41bks33pkhjh97s239sqkf3nk0hgp059p1ajn43qr72qss.nar.xz
Compression: xz
FileHash: sha256:187vlg41bks33pkhjh97s239sqkf3nk0hgp059p1ajn43qr72qss
FileSize: 14120
NarHash: sha256:1f5f4l91zqn961azgxn7h9pmdnrc1prkdl77kmb4kzl897whgdwk
NarSize: 51368
References: 5yzw0vhkyszf2d179m0qfkgxmp5wjjx4-move-docs.sh 6g3mnziija245czxdqvs4k4sc6lad0sw-update-autotools-gnu-config-scripts-hook 9dh2csn531by6b1vr9jv85v4k17xwkid-gnumake-4.4.1 cickvswrvann041nqxb0rxilc46svw1n-prune-libtool-files.sh cmn958i8qym0qvmvydl23fh3bm3fbhl7-gawk-5.2.2 f5qy259g9b4qh0hwz22z5j5bq3m53cpv-gnutar-1.35 fyaryjvghbkpfnsyw97hb3lyb37s1pd6-move-lib64.sh g5p3ky90xa05ggg5szyb0pbbl2vp7n03-gnused-4.9 h5pshzq92r2xcv6w1p10cmkar4nyv0xp-file-5.45 h9lc1dpi14z7is86ffhl3ld569138595-audit-tmpdir.sh jivxp510zxakaaic7qkrb7v1dd2rdbw9-multiple-outputs.sh kd4xwxjpjxi71jkm6ka0np72if9rm3y0-move-sbin.sh kmr52zpw7wazxywqvzgpdx0vnn9prd3v-gzip-1.13 lf0wpjrj8yx4gsmw2s3xfl58ixmqk8qa-bash-5.2-p15 m54bmrhj6fqz8nds5zcj97w9s9bckc9v-compress-man-pages.sh ngg1cv31c8c7bcm2n8ww4g06nq7s4zhm-set-source-date-epoch-to-latest.sh p2r51wfg9m3ga7pp7avslpfhfa7w5y83-gnugrep-3.11 pag6l61paj1dc9sv15l7bm5c17xn5kyk-move-systemd-user-units.sh pinwlz7294p37d2sbkdpjildzxii42vv-patch-2.7.6 qyzfglbrqb5ck0dgljplin2bvc4995w7-findutils-4.9.0 skrzk0g88jf9rg28labqsyxv7gg357q1-xz-5.4.4-bin vwkvhj69z4qqgmpa2lwm97kabf12p26r-coreutils-9.3 w1mar48lwkavwy64mvj567lwaqnm2l11-bzip2-1.0.8-bin wgrbkkaldkrlrni33ccvm3b6vbxzb656-make-symlinks-relative.sh wmknncrif06fqxa16hpdldhixk95nds0-strip.sh wzdsbnv2ba3nj91aql8jjdddfmkkdh7h-patch-shebangs.sh x6y2i213maj6ibcn0qzgg7graif5qcvi-diffutils-3.10 xyff06pkhki3qy1ls77w10s0v79c9il0-reproducible-builds.sh zlzz2z48s7ry0hkl55xiqp5a73b4mzrg-gcc-wrapper-12.3.0 znqwpxy9jlxcgi2ms2hga0ch87bbbr9g-patchelf-0.15.0
Deriver: 0m92zn8yv2z6zw29y60iwmzwwr05y0yl-stdenv-linux.drv
Sig: cache.nixos.org-1:skqQQqSKaEEMxDd6KvqVoRZ+OG+CoXQSonhB/Z1En0p/g3xBgesasW5Q0AmQyxYbi1Imw0JWObvQBKoCpj/2BQ==
```
## 运行时依赖
运行时依赖的对于包管理来说总是一件困难的事情，传统包管理可能需要编译时注释或表明自己的软件在运行时需要其他什么包。例如，一个简单的helloworld程序，在编译时可能依赖gcc，但在实际运行时gcc通常不是需要依赖的对象，只需要ld和libc.so.6即可使简单的helloworld程序运行。

而在运行时依赖的解决过程当中，有很多非常著名的问题，如依赖冲突，环形依赖等等。并且其他包管理在进行软件或者系统升级时，往往是整个系统最脆弱的时候。因为升级过程当中涉及到软件依赖的变化，以及部署方式和配置文件的改变。只要在升级过程当中遇到意外，文件不一致的问题几乎不可避免。

而nix选择的运行时依赖解决方法，可以说是巧妙的绕过了这些依赖地狱的所有生成条件。简单来说，就上文提到的stdenv-linux软件包来说，nix确定软件运行时依赖的办法可以简单概括为以下命令：
```
❯ curl -s https://cache.nixos.org/nar/187vlg41bks33pkhjh97s239sqkf3nk0hgp059p1ajn43qr72qss.nar.xz | xz -d | grep -aoP '/nix/store/[0-9a-z-\./]+'
/nix/store/lf0wpjrj8yx4gsmw2s3xfl58ixmqk8qa-bash-5.2-p15/bin/bash
/nix/store/vwkvhj69z4qqgmpa2lwm97kabf12p26r-coreutils-9.3
/nix/store/qyzfglbrqb5ck0dgljplin2bvc4995w7-findutils-4.9.0
/nix/store/x6y2i213maj6ibcn0qzgg7graif5qcvi-diffutils-3.10
/nix/store/g5p3ky90xa05ggg5szyb0pbbl2vp7n03-gnused-4.9
/nix/store/p2r51wfg9m3ga7pp7avslpfhfa7w5y83-gnugrep-3.11
/nix/store/cmn958i8qym0qvmvydl23fh3bm3fbhl7-gawk-5.2.2
/nix/store/f5qy259g9b4qh0hwz22z5j5bq3m53cpv-gnutar-1.35
/nix/store/kmr52zpw7wazxywqvzgpdx0vnn9prd3v-gzip-1.13
/nix/store/w1mar48lwkavwy64mvj567lwaqnm2l11-bzip2-1.0.8-bin
/nix/store/9dh2csn531by6b1vr9jv85v4k17xwkid-gnumake-4.4.1
/nix/store/lf0wpjrj8yx4gsmw2s3xfl58ixmqk8qa-bash-5.2-p15
......
```
nix使用了简单粗暴的方式，直接将软件的outPath目录打包为nar格式，然后在其中搜索以/nix/store/开头的路径，能搜到的目录一定为软件运行所需要的依赖，或者说，软件所依赖的包一定为这个列表的子集。这样一来，nix只需要找这个软件依赖对应的包，然后将其下载下来，再递归的寻找依赖的依赖，直到整颗依赖树被完整的下载下来。


也许会有人产生疑问，假如说编译的产物数据段被压缩了，没有办法直接搜索到哈希值该怎么办？实际解决起来也很简单，只需要在outPath里面再生成一个文本文件，把这些路径塞进去就好了😄


# Take It Further
Nix这种软件目录组织形式其实从根源上支持了系统中不同软件依赖同一个软件不同版本的问题。不需要软件作者通过类似libc.so.6之类的版本后缀来保证不兼容时的多版本共存。但在nix的实际适用当中，由于大家都用的是一套代码库(nixpkgs)所以实际上几乎不会出现多版本共存的问题。而且nixpkgs也像常规发行版一样，存在长期维护的大版本和持续滚动更新的unstable版本，对应这nixpkgs代码仓库的不同git分支。

但是假如说一个人既想用nixos-22.03里的部分软件，又想使用unstable中的部分软件，是不是直接引入两个不通分支的代码库就可以了？理论上是这样的，但这个问题其实在nix早期并没有实现这样的机制。直到`flake`的出现。

此外还有一个重要的主题就是nix是如何把这些非常规路径的软件路径组合在一起，让用户使用的。毕竟谁也不想打个bash都要打成`/nix/store/0rwyq0j954a7143p0wzd4rhycny8i967-bash-5.2-p15/bin/bash`，后续也可以学习和介绍一下nix这些环境变量和各种wrapper的组织方式。
