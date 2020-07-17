---
title: "为什么没有 debuginfo-xxx.rpm ？"
date: 2020-07-17T14:35:10+08:00
description:
draft: false
hideToc: false
enableToc: true
enableTocContent: false
tocPosition: inner
tocLevels: ["h2", "h3", "h4"]
tags:
- rpmbuild
series:
-
categories:
-
image:
---

CentOS7 默认情况下，用 `rpmbuild` 命令制作 RPM 包，还会额外自动生成一个 debuginfo 的 RPM 包，无需多做配置。这个 debuginfo 包很有用， GDB、Systemtap 等调试工具都需要它，没理由不需要它。

不过，凡事总会有例外，如果么有按照 rpm 的标准流程构建 rpm 包，可能就不会自动生成。要找出”为什么不自动生成“的原因，可以先捋一捋“怎么自动生成”。

先捋捋，如果查过怎么禁止生成 `debuginfo` 的办法的话，那大体上知道有这么个 `%{debug_package}` 宏，它其实是个子包，同时存在 spec 文件中，不过并不是手动添加的，而是在构建时自动加到 spec 文件里面的，或者说是自动展开的。

而所有的宏几乎都是定义在 `/usr/lib/rpm` 目录里面，找线索就搜索这个目录就好。先找到 `%{debug_package}` 宏的定义，它位于 **/usr/lib/rpm/redhat/macros** 文件中：

```bash
#       Template for debug information sub-package.
# NOTE: This is a copy from rpm to get the ifnarch noarch fix, it can be removed later
%debug_package \
%ifnarch noarch\
%global __debug_package 1\
%package debuginfo \
Summary: Debug information for package %{name}\
Group: Development/Debug\
AutoReqProv: 0\
%description debuginfo\
This package provides debug information for package %{name}.\
Debug information is useful when developing applications that use this\
package or when debugging this package.\
%files debuginfo -f debugfiles.list\
%defattr(-,root,root)\
%endif\
%{nil}

%install %{?_enable_debug_packages:%{?buildsubdir:%{debug_package}}}\
%%install\
%{nil}
```

分析一下这个宏的构成：添加了一个 %package 宏，定义了一个全局宏 `__debug_package`，定义了打包阶段的命令 `debuginfo -f debugfiles.list`。总体意思就是打包 debugfiles.list 文件中的包含文件列表。

什么情况下会自动添加这个 `%debug_package`？紧挨着定义，它是被 `%{install}` 宏引用了。
意思是会在 `%install` 阶段执行，不过这里有逻辑判断，检查了 `%buildsubdir` 和 `%_enable_debug_packages` 是不是有值。

先找找 `%{buildsubdir}` 的定义，它是在 `%setup` 之后才有值，默认值是 `%name-%version`，也可以传入参数改变。
```sh
#==============================================================================
# ---- Optional rpmrc macros.
#       Macros that are initialized as a side effect of rpmrc and/or spec
#       file parsing.
#
#       The sub-directory (relative to %{_builddir}) where sources are compiled.
#       This macro is set after processing %setup, either explicitly from the
#       value given to -n or the default name-version.
#
#%buildsubdir
```

再看看 `_enable_debug_packages` 的定义，在 **/usr/lib/rpm/redhat/macros** 文件中找到，它的值默认就是 `1`。
```
%_enable_debug_packages 1
%_include_minidebuginfo 1
```

这样的话，那关键点就落在了 `%{buildsubdir}` 宏，而如果只要是按照正常逻辑写的 spec 文件，其中就是有 `%setup` 宏，就满足了自动添加 `%{debug_package}` 宏的条件。关系清楚了。

`%{setup}` 宏用途是解压包，而我的环境中是跳过了解压，直接转到源码目录编译的。原因，就是因为没有使用 `%setup` 宏。

继续。如果没有使用 `%setup` 宏，主动给 `%{buildsubdir}` 设置一个值，我的构建过程应该就可以了。看注释，它的值应该是一个 `%{_builddir}` 的子目录。

搜索下 `%{_builddir}` 宏定义，它的值是顶级目录下的 `BUILD` 子目录。这目录是用来编译的，不是用来打包的。`%setup` 会把源码解压到一个 `%name-%version` 的子目录。
```bash
#       The directory where sources/patches will be unpacked and built.
%_builddir              %{_topdir}/BUILD
```

我的 case 因为是通过别的方式编译的，所以需要把编译好的的可执行文件放到这样的子目录里面，然后设定好 `%{_buildsubdir}` 的值，应该就可以工作了。我在编译时添加指令 `--define "_buildsubdir %name-%version"` 了。不过没有使用 `%setup` 宏，就要事先创建好子目录，并把编译好的文件复制子目录中。

`%debug_package` 宏定义中还定义了一个 `%__debug_package` 宏，查看下它有什么用途。在 **/usr/lib/rpm/macros** 文件中。
```bash
%__spec_install_post\
%{?__debug_package:%{__debug_install_post}}\
%{__arch_install_post}\
%{__os_install_post}\
%{nil}
```

`%__spec_install_post` 宏是在 `%{install}` 之后执行的，主要目的是生成 debuginfo 文件。而 `%install` 阶段其实是要把可执行文件安装到 `%{buildroot}` 路径中。

搜索 `%__debug_install_post` 宏，在 **/usr/lib/rpm/macros** 文件中找到。
```sh
%__debug_install_post   \
   %{_rpmconfigdir}/find-debuginfo.sh %{?_missing_build_ids_terminate_build:--strict-build-id} %{?_include_minidebuginfo:-m} %{?_find_debuginfo_dwz_opts} %{?_find_debuginfo_opts} "%{_builddir}/%{?buildsubdir}"\
%{nil}
```

`find-debuginfo.sh` 脚本会搜索哪些可执行文件含有调试符号，另外还使用了之前的两个宏  `%{_builddir}/%{buildsubdir}`。脚本内容就不细查了，只知道这个脚本会生成一些文件，比如 "debugfiles.list" 文件，可以在 `%{_builddir}/BUILD` 目录中找到。

回头再看看 `%debug_package` 的定义（这里去掉了目前不用关心的内容）：
```sh
%debug_package \
.....
.....
%files debuginfo -f debugfiles.list\
%defattr(-,root,root)\
%endif\
%{nil}
```

这里的 `%files`  宏会把 `debugfiles.list` 里面的文件打包到 debuginfo 包里。

至此基本上就分析完了，自己的问题也解决了。

最后，就是平时编辑 spec 文件的时间不多，语法和逻辑感觉也有点怪，而开源软件几乎都不提供各种平台包管理器的 spec 文件，经过这么分析一通，似乎也有点理解。

也难怪会有一个 [`fpm`](https://github.com/jordansissel/fpm) 项目把其他平台所有的包封装过程改成了参数和命令，使用起来简单，只需要记住几个参数就行了。但相对的，对于某种格式独有的特性并不完全支持、兼容，如果不用某些特有的功能，算是很简单的。 `fpm` 目前就不支持自动生成 debuginfo ，它生成的 spec 文件中把关键的几个打包阶段都给禁止了。

难办！