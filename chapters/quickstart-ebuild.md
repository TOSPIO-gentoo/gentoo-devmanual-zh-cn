ebuild快速指南
---

本文非常精要地介绍了ebuild的编写方法，并不涉及许多细节和开发者可能遇到的各类问题，而是通过几个简单的示例帮助开发者熟悉ebuild的工作流程并快速上手。


## 第一个ebuild文件

首先我们为*Exuberant Ctags*工具编写一个ebuild。Exuberant Ctags是一个源代码索引工具。下面是`dev-util/ctags/ctags-5.5.4.ebuild`的简化版本：

```ebuild
# Copyright 1999-2017 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

EAPI=6

DESCRIPTION="Exuberant ctags generates tags files for quick source navigation"
HOMEPAGE="http://ctags.sourceforge.net"
SRC_URI="mirror://sourceforge/ctags/${P}.tar.gz"

LICENSE="GPL-2"
SLOT="0"
KEYWORDS="~mips ~sparc ~x86"

src_configure() {
    econf --with-posix-regex
}

src_install() {
    emake DESTDIR="${D}" install

    dodoc FAQ NEWS README
}
```

### 基本格式

如你所见，ebuild其实就是一个在特定环境下执行的bash脚本。
顶部是文件头块，在所有ebuild中都有。
ebuild文件使用tab缩进，每个tab占位四个空格。参见[ebuild文件格式](chapters/ebuild-writing/file-format.md)。

### 变量

文件头块以下是一系列变量：

* `EAPI`，参见[EAPI用法和描述](chapters/ebuild-writing/eapi.md)。
* `DESCRIPTION`，包的简要描述和作用。
* `HOMEPAGE`，包的首页链接（不要忘记包含`http://`）。
* `SRC_URI`，源代码包的下载地址。在上述ebuild中，`mirror://sourceforge`是一种特殊标记法，其含义是“任意的Sourceforce镜像”。`${P}`是一个只读变量，由portage设置，表示包的名称和版本，在这个示例中，值为`ctags-5.5.4`。
* `LICENCE`，许可证，示例中是`GPL-2`(GNU通用公共许可证第二版)。
* `SLOT`，包将要装到哪个*slot*中。如果你不知道slot是什么鬼，使用`"0"`，或者阅读[Slot](chapters/general-concepts/slotting.md)。
* `KEYWORDS`，定义为软件通过测试的架构，示例中使用的`~`，表示最新添加的ebuild。即使包能够正常运行，也不会立刻提交到stable分支下。详情请参见[Keyword](keywording.md)。

### 构建函数

接着是一个`src_configure`函数。当portage对包进行*configure*操作时会调用此函数。`econf`函数是对`./configure`调用的封装。如果`econf`因故失败，则portage会停止安装。

当portage对包进行*install*操作时，会调用`src_install`函数。有一个细节要注意：portage并不会直接安装到目标文件系统中，而是装到一个由`${D}`变量指定的特殊系统中，该变量由portage设置（参见[安装目的地](chapters/general-concepts/install-destinations.md)和[沙盒](chapters/general-concepts/sandbox.md)）

> 注意：标准的安装方式是`emake DESTDIR="${D}" install`，对于规范的Makefile来说都适用。如果报sandbox错误，请尝试使用`einstall`。如果还是失败，请参见[src_install](chapters/ebuild-writing/functions/src_install.md)了解如何手动安装。

`dodoc`是一个helper函数，将文件安装在`/usr/share/doc`的对应位置下。

ebuild中还可以定义新的函数（参见[ebuild函数](ebuild-writing/functions.md)）

portage为每个函数提供了一种合理的默认实现，在很多情况下使用默认实现即可。比如，这里不需要定义`src_unpack`和`src_compile`函数——`src_unpack`函数用于对源码包和源码补丁进行解压，示例中默认实现已经可以满足我们的需求。类似地，默认的`src_compile`函数将调用`emake`（一个`make`命令的封装）。

> 注意，在以前的版本中必须在每个命令后加上`|| die`进行错误检查。从EAPI 4开始不再需要，portage提供的函数报错会自动停止。


## 带有依赖关系的ebuild

在上面的ctags示例中，我们没有指定任何依赖关系，然而这样做并没有任何问题，因为ctags只需要基础工具链就可以编译运行（参见[隐式的系统依赖](chapters/general-concepts/dependencies.md#隐式的系统依赖)）。现实中，情况通常会更加复杂。

下面是`app-misc/detox/detox-1.1.1.ebuild`的内容：

```ebuild
# Copyright 1999-2017 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

EAPI=5

DESCRIPTION="detox safely removes spaces and strange characters from filenames"
HOMEPAGE="http://detox.sourceforge.net/"
SRC_URI="mirror://sourceforge/${PN}/${P}.tar.bz2"

LICENSE="BSD"
SLOT="0"
KEYWORDS="~hppa ~mips sparc x86"

RDEPEND="dev-libs/popt"
DEPEND="${RDEPEND}
    sys-devel/flex
    sys-devel/bison"

src_configure() {
    econf --with-popt
}

src_install() {
    emake DESTDIR="${D}" install
    dodoc README CHANGES
}

```

和第一个示例一样，这里也有ebuild头和各种变量。在`SRC_URI`中，`${PN}`用于获取不带版本号后缀的包名（还有其他类似的变量，参见[预定义的只读变量](chapters/ebuild-writing/variables.md#预定义的只读变量)。

和第一个示例一样，我们也定义了`src_configure`和`src_install`函数。

`DEPEND`和`RDEPEND`变量告知portage该包依赖于哪些其他包，`DEPEND`代表构建时依赖，`RDEPEND`代表编译时依赖。参见[依赖关系](general-concepts/dependencies)来浏览更加复杂的示例。


## 带补丁的ebuild

我们经常需要对源码包应用补丁，在ebuild中通过在`src_prepare`函数中调用`epatch` helper函数来达成。

在调用`epatch`之前，必须告诉portage要引用`eutils` eclass（eclass相当于一个类库）——通过在ebuild顶部添加`inherit eutils`语句来实现。

下面是`app-misc/detox/detox-1.1.0.ebuild`的内容：

```ebuild
# Copyright 1999-2017 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

EAPI=5

inherit eutils

DESCRIPTION="detox safely removes spaces and strange characters from filenames"
HOMEPAGE="http://detox.sourceforge.net/"
SRC_URI="mirror://sourceforge/${PN}/${P}.tar.bz2"

LICENSE="BSD"
SLOT="0"
KEYWORDS="~hppa ~mips ~sparc ~x86"

RDEPEND="dev-libs/popt"
DEPEND="${RDEPEND}
    sys-devel/flex
    sys-devel/bison"

src_prepare() {
    epatch "${FILESDIR}"/${P}-destdir.patch \
        "${FILESDIR}"/${P}-parallel_build.patch
}

src_configure() {
    econf --with-popt
}

src_install() {
    emake DESTDIR="${D}" install
    dodoc README CHANGES
}
```

注意里面的`${FILESDIR}/${P}-destdir.patch`——实际指向Gentoo仓库中`files/`子文件夹下的`detox-1.1.0-destdir.patch`文件。对于更大的补丁文件，必须上传到你在`dev.gentoo.org`中的开发者空间中。参见[Gentoo镜像](chapters/general-concepts/mirrors.md#Gentoo镜像)和[使用epatch函数打补丁](chapters/ebuild-writing/functions/src_prepare/epatch.md)。


## 带USE标记的ebuild

下面是`dev-libs/libiconv/libiconv-1.9.2.ebuild`文件，*libiconv*为不带iconv的libc实现提供了替代方案。

```ebuild
# Copyright 1999-2017 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

EAPI=5

DESCRIPTION="GNU charset conversion library for libc which doesn't implement it"
HOMEPAGE="http://www.gnu.org/software/libiconv/"
SRC_URI="ftp://ftp.gnu.org/pub/gnu/libiconv/${P}.tar.gz"

LICENSE="LGPL-2.1"
SLOT="0"
KEYWORDS="~amd64 ~ppc ~sparc ~x86"
IUSE="nls"

DEPEND="!sys-libs/glibc"

src_configure() {
    econf $(use_enable nls)
}

src_install() {
    emake DESTDIR="${D}" install
}
```
注意到其中的`IUSE`变量，其中列举出ebuild所使用的所有非特殊USE标记。USE标记会在`emerge -pv`命令中输出。

libiconv包的`./configure`脚本接收常见的`--enable-nls`或`--disable-nls`参数。我们使用`use_enable`工具函数来根据用户设置的USE标记自动生成该参数（参见[查询函数参考](chapters/function-reference/query-functions.md)）。

下面是一个更复杂的示例，基于`mail-client/sylpheed/sylpheed-1.0.4.ebuild`：

```ebuild
# Copyright 1999-2017 Gentoo Foundation
# Distributed under the terms of the GNU General Public License v2

EAPI=5

inherit eutils

DESCRIPTION="A lightweight email client and newsreader"
HOMEPAGE="http://sylpheed.good-day.net/"
SRC_URI="mirror://sourceforge/${PN}/${P}.tar.bz2"

LICENSE="GPL-2"
SLOT="0"
KEYWORDS="alpha amd64 hppa ia64 ppc ppc64 sparc x86"
IUSE="crypt imlib ipv6 ldap nls pda ssl xface"

RDEPEND="=x11-libs/gtk+-2*
    crypt? ( >=app-crypt/gpgme-0.4.5 )
    imlib? ( media-libs/imlib2 )
    ldap? ( >=net-nds/openldap-2.0.11 )
    pda? ( app-pda/jpilot )
    ssl? ( dev-libs/openssl )
    xface? ( >=media-libs/compface-1.4 )
    app-misc/mime-types
    x11-misc/shared-mime-info"
DEPEND="${RDEPEND}
    dev-util/pkgconfig
    nls? ( >=sys-devel/gettext-0.12.1 )"

src_prepare() {
    epatch "${FILESDIR}"/${PN}-namespace.diff \
        "${FILESDIR}"/${PN}-procmime.diff
}

src_configure() {
    econf \
        $(use_enable nls) \
        $(use_enable ssl) \
        $(use_enable crypt gpgme) \
        $(use_enable pda jpilot) \
        $(use_enable ldap) \
        $(use_enable ipv6) \
        $(use_enable imlib) \
        $(use_enable xface compface)
}

src_install() {
    emake DESTDIR="${D}" install

    doicon sylpheed.png
    domenu sylpheed.desktop

    dodoc [A-Z][A-Z]* ChangeLog*
}
```

注意这里包含可选依赖。

有些`use_enable`调用带两个参数，这种方式适用于USE标记和实际参数名称不匹配的情况，其中第一个参数代表USE标记，第二个参数代表当USE标记激活时的替代字符串，比如上面的`use_enable crypt gpgme`，其含义为：当激活`crypt`USE标记时，对`./configure`添加`--enable-gpgme`参数。
