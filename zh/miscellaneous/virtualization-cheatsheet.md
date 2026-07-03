# 虚拟化速查表

## 概览

OpenBSD 原生虚拟化栈包含以下组件：

- **[vmm(4)](https://man.openbsdhandbook.com/vmm.4/)**：内核 hypervisor。
- **[vmd(8)](https://man.openbsdhandbook.com/vmd.8/)**：虚拟机守护进程。
- **[vmctl(8)](https://man.openbsdhandbook.com/vmctl.8/)**：命令行控制工具。

详细指引请参阅本手册的 [虚拟化章节](https://www.openbsdhandbook.com/virtualization/)
。

## 目录布局

按 OpenBSD 惯例，把虚拟机资源放在 `/var/vmm/` 目录下。

| 用途 | 路径 | 说明 |
| --- | --- | --- |
| 磁盘镜像 | `/var/vmm/images/<name>.img` | 由 `vmctl` 创建的稀疏裸文件。 |
| 安装 ISO | `/var/vmm/iso/<file>.iso` | 存放官方 ISO（OpenBSD、FreeBSD 等）。 |
| 配置文件 | `/var/vmm/conf/` | 可选：存放 `install.conf`、kickstart 等。 |
| 日志（宿主） | `/var/log/vmd.log` | 用于排查问题的 `vmd(8)` 日志。 |

## 磁盘与镜像创建

| 任务 | 命令 | 说明 |
| --- | --- | --- |
| 创建 10G 稀疏磁盘镜像 | `# vmctl create -s 10G /var/vmm/images/obsd.img` | 按写入量增长，上限为指定大小。 |
| 下载 OpenBSD ISO | `# ftp https://cdn.openbsd.org/pub/OpenBSD/78/amd64/install78.iso -o /var/vmm/iso/install78.iso` | 使用 `ftp(1)` 或类似工具。 |

## 单台 OpenBSD 虚拟机（交互式安装）

```sh
# vmctl create -s 10G /var/vmm/images/obsd.img
# vmctl start -c -m 1G -r /var/vmm/iso/install78.iso -d /var/vmm/images/obsd.img obsd
```

- `-c`：连接控制台。
- `-m`：内存（1G = 1024M）。
- `-r`：安装用 ISO。
- `-d`：磁盘镜像。
- 最后一个参数：虚拟机名称。

安装完成后，去掉 ISO 重启：

```sh
# vmctl stop obsd
# vmctl start -c -m 1G -d /var/vmm/images/obsd.img obsd
```

## 网络选项

用 `vmctl start` 配合 `-n` 选项配置网络。

| 模式 | 说明 | 命令示例 |
| --- | --- | --- |
| `-n local` | 私有 NAT 网络；默认值，简单易用。 | `-n local` |
| `-n <switch>` | 接入虚拟交换机（由 `vmd(8)` 管理）。 | 先创建交换机，再 `-n switch0`。 |
| `-n <tapX>` | 接到 `tap(4)` 接口，用于自定义配置。 | `ifconfig tap0 create` 后再 `-n tap0`。 |

### 示例：两台虚拟机接入私有 NAT 网络

```sh
# vmctl create -s 5G /var/vmm/images/vm1.img
# vmctl create -s 5G /var/vmm/images/vm2.img
# vmctl start -c -m 1G -d /var/vmm/images/vm1.img -n local vm1
# vmctl start -c -m 1G -d /var/vmm/images/vm2.img -n local vm2
```

### 示例：把虚拟机桥接到物理局域网

1. 创建虚拟交换机与网桥：

```sh
# ifconfig vether0 create
# ifconfig bridge0 create
# ifconfig bridge0 add em0 add vether0 up
```

把 `em0` 换成你的物理网卡。

2. 启动虚拟机：

```sh
# vmctl start -c -m 2G -d /var/vmm/images/srv.img -n switch0 srv
# vmctl start -c -m 1G -d /var/vmm/images/cli.img -n switch0 cli
```

> **提示**：如需 VLAN，可用 `vether0.<vlan>`，或在网桥上配置 VLAN 标记。参见 [ifconfig(8)](https://man.openbsdhandbook.com/ifconfig.8/)
> 与 [vether(4)](https://man.openbsdhandbook.com/vether.4/)
> 。

## 非 OpenBSD 客户机

使用 amd64 ISO，并分配足够内存（建议 2G 以上）。

### FreeBSD

```sh
# vmctl create -s 20G /var/vmm/images/fbsd.img
# vmctl start -c -m 2G -r /var/vmm/iso/FreeBSD-14.1-RELEASE-amd64-dvd1.iso -d /var/vmm/images/fbsd.img fbsd
```

### Linux（以 Debian 为例）

```sh
# vmctl create -s 10G /var/vmm/images/debian.img
# vmctl start -c -m 2G -r /var/vmm/iso/debian-12.7.0-amd64-netinst.iso -d /var/vmm/images/debian.img debian
```

### Windows（支持有限）

Windows 支持尚处实验阶段，功能可能受限。

```sh
# vmctl create -s 30G /var/vmm/images/win10.img
# vmctl start -c -m 4G -r /var/vmm/iso/Win10_22H2_English_x64.iso -d /var/vmm/images/win10.img win10
```

## 虚拟机管理

| 操作 | 命令 |
| --- | --- |
| 列出虚拟机 | `# vmctl status` |
| 连接控制台 | `# vmctl console <name>` |
| 优雅关机 | 客户机内执行 `halt` 或对应系统的关机命令 |
| 强制停止 | `# vmctl stop -f <name>` |
| 暂停 / 恢复 | `# vmctl pause <name>` / `# vmctl unpause <name>` |

非 root 用户如需访问，请配置 [doas(1)](https://man.openbsdhandbook.com/doas.1/)
。

## 故障排查

- 查看 `vmd` 日志：`tail -f /var/log/vmd.log`。
- 检查网络接口：`ifconfig -a`。
- 确认路径：`/var/vmm/images/`、`/var/vmm/iso/`。
- 自定义网络出问题时，检查 `pf(4)` 规则。
