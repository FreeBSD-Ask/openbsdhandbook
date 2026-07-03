# Linux 兼容性

## 概述

本章介绍 OpenBSD 上 Linux 二进制兼容性的现状，描述历史上曾经用于执行 Linux ELF 二进制文件的 `linux(4)` 子系统、移除原因，以及访问 Linux 专属软件的现代替代方案，包括虚拟化、远程执行和原生软件包。

## 历史说明

OpenBSD 此前曾通过 `linux(4)` 内核子系统支持部分 Linux 兼容性。该层可执行部分 Linux ELF 二进制文件，主要用于兼容旧版商业软件或仅以二进制形式发布的软件。它需要 Linux 共享库，并在受控环境中模拟系统调用。

该特性使用不多，维护难度却不断增加。OpenBSD 6.3 中已从内核正式移除。

## 移除原因

移除 `linux(4)` 实现的原因如下：

- 维护与外来 ABI 的二进制兼容性与 OpenBSD 注重正确性、简洁性和安全性的方向相冲突。
- 曾经需要 Linux 兼容性的多数软件已开源，并提供了原生版本。
- 兼容层需要持续跟进 Linux 行为和 libc 接口的变化，但用户需求寥寥。

鉴于现代应用与系统设计的发展，该子系统已过时且难以维护。

## 替代方案与变通方法

### 原生软件包与 Ports

许多最初为 Linux 开发的软件包如今已作为 OpenBSD 原生 ports 提供，包括：

- 桌面软件：Firefox、Chromium、LibreOffice
- 开发工具：GCC、Rust、Go
- 网络工具：WireGuard、Docker 客户端、Kubernetes 工具

搜索可用软件：

```sh
$ pkg_info -Q firefox
$ pkg_info -Q libreoffice
```

若软件包不存在，只要不依赖 Linux 内核专属接口，可尝试从源码编译。

### 虚拟化

需要完整 Linux 环境的 Linux 应用，可借助 OpenBSD 内置的虚拟机管理框架（`vmm(4)` 和 `vmd(8)`）在虚拟机中运行。

在虚拟机中安装 Linux 客户机的步骤见 [虚拟化](../system-administration/virtualization.md) 一章。

### 远程执行

对于涉及 Linux 专属软件的工作流，在 Linux 主机上远程执行可能是可行的方案。可通过 SSH 进行，并按需开启 X11 转发：

```sh
$ ssh -X user@linuxhost gimp
```

运行仅命令行的应用：

```sh
$ ssh user@linuxhost ffmpeg -i input.avi -vf scale=640:360 output.mp4
```

可按需用 `scp` 或 `rsync` 传输文件。

也可用容器（如 `podman`、`docker`、`lxc`）或完整虚拟机提供远程 Linux 环境。
