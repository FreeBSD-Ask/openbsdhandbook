# 软件管理：软件包与 Ports

## Synopsis

本章详细讲解 OpenBSD 上的软件管理，重点介绍用 `pkg_*` 工具安装与维护预编译软件包，以及用 `ports(7)` 从 Ports Collection 构建自定义应用，并涵盖安装后追加文件集的方法。

## 概述

OpenBSD 提供两种安装软件的主要方式：预编译二进制软件包，以及用 Ports Collection 从源码构建。软件包便捷、安装快，对关键软件及时更新。Ports Collection 支持自定义构建，但需要更多时间、磁盘空间与系统资源。

## 使用软件包系统

OpenBSD 软件包是经过压缩与签名的二进制归档，由 `pkg_*` 工具套件管理，安装便捷，多数场景下都推荐使用。

### 选择镜像

系统默认使用 **/etc/installurl** 中列出的镜像。手动配置：

```
https://cdn.openbsd.org/pub/OpenBSD
```

也可使用 `PKG_PATH` 环境变量（除非用于脚本或调试，否则不推荐）：

```sh
export PKG_PATH=https://cdn.openbsd.org/pub/OpenBSD/7.8/packages/amd64/
```

将 `7.8` 替换为系统发行版本。用 `uname -r` 核对。

其他选项参见 `installurl(5)` 与 `pkg_add(1)`。

### 安装软件包

用 `pkg_add(1)` 安装软件：

```sh
doas pkg_add vim
```

若软件包有多个 flavor 或子包，系统会提示：

```
Ambiguous: choose package for rsync
a       0: <None>
        1: rsync-3.1.2p0
        2: rsync-3.1.2p0-iconv
Your choice:
```

可输入编号，或直接指定完整 flavor：

```sh
doas pkg_add rsync--iconv
```

直接从 URL 安装：

```sh
doas pkg_add https://mirror.example.org/packages/vim-9.0.1234p0.tgz
```

部分软件包在 **/usr/local/share/doc/pkg-readmes/** 下提供说明，若有提示务必阅读。

### 搜索软件包

按包名搜索：

```sh
pkg_info -Q unzip
```

在软件包中查找文件：

```sh
pkglocate mutool
```

此命令需要安装 `pkglocatedb` 软件包。

### 更新软件包

升级所有已安装软件包：

```sh
doas pkg_add -Uu
```

更新指定软件包：

```sh
doas pkg_add -u unzip
```

OpenBSD 会为受支持版本重建部分软件包，例如 Firefox。这些更新可通过 `pkg\_add -Uu` 获取。

## 更新系统

### 基本系统更新

OpenBSD 通过 `syspatch(8)` 为基本系统提供带签名的二进制补丁。这些补丁适用于受支持的 `-release` 版本，通常针对安全漏洞或其他重要修复发布。应用所有可用补丁：

```sh
doas syspatch
```

建议定期运行 `syspatch`，保持基本系统最新。

### 软件包更新

软件包可与基本系统独立更新。下列命令升级所有已安装软件包：

```sh
doas pkg_add -Uu
```

该命令会联系已配置的软件包镜像，将已安装软件包升级到更新版本（若有）。请定期运行此命令，以获取浏览器、邮件客户端等受支持软件包的更新。

## 删除软件包

用 `pkg_delete(1)` 删除软件：

```sh
doas pkg_delete screen
```

删除不再使用的依赖：

```sh
doas pkg_delete -a
```

### 复制软件包列表

将软件包集合复制到另一台机器：

```sh
pkg_info -mz > list
doas pkg_add -l list
```

### 处理不完整或损坏的软件包

若安装失败并留下 `partial-*` 条目，用以下命令清理：

```sh
doas pkg_delete partial-vim
```

若软件包元数据损坏：

```sh
doas pkg_check
```

## 使用 Ports Collection

Ports Collection 允许用自定义选项从源码构建软件。

### 安装 Ports 树

获取 ports 树：

```sh
doas cvs -qd anoncvs@anoncvs.openbsd.org:/cvs get -rOPENBSD_7_6 -P ports
```

更新：

```sh
doas cd /usr/ports && cvs update -Pd
```

### 构建一个 Port

```sh
cd /usr/ports/editors/vim
doas make install
```

配置选项：

```sh
make config
```

### 管理依赖

列出所需软件包：

```sh
make show-DEPENDS
```

手动安装：

```sh
doas pkg_add $(make show-DEPENDS)
```

### 自定义构建

构建不带 X11 的 `vim`：

```sh
cd /usr/ports/editors/vim
env NO_X11=Yes doas make install
```

构建 ports 可能耗时数小时，并占用大量磁盘空间。除非确需自定义编译，否则请使用二进制软件包。

## 安装后追加文件集

`comp{{ .Site.Params.short_version }}.tgz` 或 `xbase{{ .Site.Params.short_version }}.tgz` 等文件集可在初次安装后追加。

### 方法一：使用 bsd.rd 与升级模式

1. 从现有磁盘或 USB 引导 `bsd.rd`。
2. 选择 `(U)pgrade`。
3. 选择并安装缺失的文件集。
4. 重启进入完整系统。

### 方法二：手动解包

```sh
cd /tmp
ftp https://cdn.openbsd.org/pub/OpenBSD/{{ .Site.Params.full_version }}/amd64/comp{{ .Site.Params.short_version }}.tgz
cd /
doas tar xzvphf /tmp/comp{{ .Site.Params.short_version }}.tgz
```

`-p` 选项用于保留正确的权限。
