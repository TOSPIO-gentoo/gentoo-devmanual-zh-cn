# gentoo-ebuild-zh-cn

---

## 什么是ebuild

*ebuild*是一种由Gentoo包管理器使用的文本文件，它标识了某个特定的软件包，并告知包管理器如何处理该软件包。
ebuild使用类似bash的语法，由EAPI版本进行标准化。

## 什么是EAPI

*EAPI*是定义在ebuild和其他包管理器相关的文件中的一个版本标识，该标识告知包管理器使用哪种文件内容和语法。
它实际上是文件所遵循的*包管理器规范 (PMS)*

## 什么是包管理器规范 (PMS)

*包管理器规范 (PMS)*是一种标准化方案，定义了ebuild文件格式、ebuild仓库格式以及包管理器与这些ebuild的交互方式。

（所有已通过审核的EAPI清单请参见[https://wiki.gentoo.org/wiki/Package_Manager_Specification](https://wiki.gentoo.org/wiki/Package_Manager_Specification)）

## 什么是ebuild仓库

*ebuild仓库*，以前被称作overlay，是一个文件和文件夹的组织结构，用于给包管理器扩展现有的或添加更多的安装包。

## 什么是portage

*portage*是Gentoo官方的包管理器和发布系统。但其并非唯一，其他还有诸如*paludis*和*pkgcore*等。

---

## 如何编写ebuild

当你装好Gentoo的时候，你应该已经设置好了Gentoo的仓库，存放在`/usr/portage`下。
好，如果你想添加新的ebuild，你可不能直接创建一个`/usr/portage/hello-world.ebuild`，原因如下：

1. 它是一份远程Gentoo仓库的本地镜像：如果你在里面添加或修改了一些东西，你一`emerge --sync`，你的改动就全没了。
1. 没有在正确的文件夹下面：应该按照`./类别/包/ebuild文件`的层级来存。
1. ebuild文件名没有包含版本号：包的版本号必须在文件名中指定，所以我们要创建的ebuild的完整路径可能长这样：`/usr/local/portage/app-misc/hello-world/hello-world-1.0.ebuild`

下面我们来创建一个最小的ebuild文件。假定你已经有root权限：
```
root # mkdir -p /usr/local/portage/app-misc/hello-world
root # cd $_
```

如果你使用`app-editors/vim`的话，直接用下述命令打开，就可以得到一个基础的文件框架（由`app-vim/gentoo-syntax`提供）

```
root # vim ./hello-world-1.0.ebuild
```

```ebuild
# Copyright 1999-2017 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

EAPI=6

DESCRIPTION=""
HOMEPAGE=""
SRC_URI=""

LICENSE=""
SLOT="0"
KEYWORDS="~amd64 ~x86"
IUSE=""

DEPEND=""
RDEPEND="${DEPEND}"
```

如果你是用其他的编辑器，则可以执行如下命令：

```
root # cp /usr/portage/header.txt ./hello-world-1.0.ebuild
```

当然，现在这个ebuild还不能用，然而只需定义两个变量就可以算是一个完好的ebuild文件：

```ebuild
DESCRIPTION="A classical example to use when starting on something new"
SLOT="0"
```

好，执行以下命令来安装这个ebuild：

```
root # ebuild hello-world-1.0.ebuild manifest clean merge
```

这条命令将会创建`Manifest`文件（生成hash以防止包损坏）、清理临时文件夹并安装ebuild。

> 什么是Manifest文件？
> > 在portage树中，每个包都有一个Manifest文件，存放在和ebuild相同的文件夹下，其中包含了摘要信息 (目前包括RMD160, SHA1, SHA256, SHA512以及WHIRLPOOL)，
> > 以及文件夹、子文件夹下所有文件的大小，用于验证包的完整性。Manifest文件还可以进行数字签名。
> > 使用`ebuild foo.ebuild manifest`来生成Manifest文件。每次提交ebuild时都必须重新生成Manifest文件（使用repoman来自动生成）

至此，一个最简单的ebuild文件就算是创建完成了。
