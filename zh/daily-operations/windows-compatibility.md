# Windows 兼容性

## 概述

本章介绍 OpenBSD 上 Windows 兼容性的现状。由于架构、安全和内核相关的不兼容性，OpenBSD 上无法使用 Wine。本章说明 Wine 未能成功移植的原因，记录过去的尝试，并提供虚拟化、远程访问和原生软件等实用替代方案。

## Wine 与 OpenBSD

Wine（Wine Is Not an Emulator）是一款兼容层，允许在类 Unix 操作系统上运行 Microsoft Windows 应用。它在 FreeBSD、NetBSD 以及多数 Linux 发行版上支持良好。

然而，OpenBSD 的官方 ports 树和软件包集合中**并不提供 Wine**。历史上对 Wine 的移植尝试均告失败或被放弃，原因在于与 OpenBSD 设计存在若干根本性不兼容。

## 技术障碍

将 Wine 移植到 OpenBSD 的努力遇到了以下技术障碍：

- **缺乏 32 位二进制支持**：OpenBSD/amd64 不支持运行 32 位二进制文件。这破坏了与许多 Windows 应用和子系统的兼容性，因为 Wine 即便运行 64 位 Windows 程序，也严重依赖 32 位执行环境。许多安装程序、DLL 和系统组件都期望 32 位运行时环境。
- **安全限制**：OpenBSD 严格的内存保护策略（如 W^X，即写与执行互斥）以及禁止映射地址页 0 的规定，与 Wine 模拟 Windows 内存布局和执行模型的需求相冲突。这些限制干扰了 Wine 的可移植可执行（PE）加载器、信号处理和即时编译代码执行。
- **内核 ABI 差异**：OpenBSD 内核与 Linux 及其他 BSD 系统差异显著。Wine 假定进程模型和信号行为类似 Linux 或 FreeBSD，因此不修改内核就难以低层模拟 Windows API。
- **依赖与构建失败**：从源码构建 Wine 时遇到了库缺失或不兼容（如 `lcms`、`hal`、`alsa`），以及与进程克隆和信号传递相关的错误。这些问题悬而未决，也无人维护。

最近一次已知的 Wine 移植尝试已停止，该 port 也已从 OpenBSD ports 树中移除。

## 历史尝试

OpenBSD 上对 Wine 的移植尝试最早可追溯到 2008 年，断断续续持续到 2018 年左右。这些尝试从未达到可用或可维护的状态。Wine 曾一度存在于 ports 树的 `emulators/wine` 下，但后来被标记为损坏并归档至 attic。

早期版本的 Wine 有时在 i386 系统上借助 OpenBSD 已移除的 `linux(4)` 兼容层运行。但该方法现已不再支持，也不推荐使用。

## 替代方案

### 虚拟化

在 OpenBSD 上运行 Windows 软件最可靠的方式是虚拟化。OpenBSD 内置 `vmm(4)` 虚拟机管理器和 `vmd(8)` 管理守护进程，可在用户空间运行完整的 Windows 虚拟机。

该方法完全避开模拟，与 Windows 应用和系统调用完全兼容。由于没有 KVM，性能不及 Linux，但仍安全可用。

安装步骤见 [虚拟化](../system-administration/virtualization.md) 一章。

### 远程访问

另一种方案是在 Windows 系统上远程运行 Windows 软件，并通过标准协议访问：

- `ssh` 配合 X11 转发
- `rdesktop` 或 `xfreerdp` 用于 RDP
- `vncviewer` 用于 VNC
- 远程主机上的 `xrdp` 或 `x11vnc`

示例：

```sh
$ rdesktop winhost.example.org
```

此方案特别适合企业环境或已有专用 Windows 机器的场景。

### 原生替代软件

在条件允许时，优先用原生软件替代 Windows 应用。许多开源和跨平台程序已为 OpenBSD 打包。

示例：

| Windows 软件 | OpenBSD 替代方案 |
| --- | --- |
| Notepad++ | `mg`、`mousepad`、`vim` |
| PuTTY | `ssh`（基本系统内置） |
| WinSCP | `sftp`、`rsync`、`scp` |
| MS Paint | `gimp`、`xpaint` |
| MS Word | `libreoffice` |

搜索可用的软件包：

```sh
$ pkg_info -Q libreoffice
$ pkg_info -Q gimp
```

部分较旧的 Windows 软件也可在 `dosbox` 下运行。`dosbox` 通过软件包提供，适合运行 16 位或 DOS 时代的 Windows 应用（例如在 DOSBox 中安装 Windows 3.1 后运行 SimCity 2000）。

### 其他操作系统

经常需要使用 Wine 的用户，可考虑双系统启动，或在虚拟机中安装 FreeBSD 或 Linux 客户机。这两个平台都原生支持 Wine，并有大量关于运行 Windows 软件的文档。
