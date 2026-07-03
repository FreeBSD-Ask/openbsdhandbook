# NFS

## 概述

**NFS（Network File System）** 是一种分布式文件系统协议，OpenBSD 基本系统支持该协议。它允许一台系统通过网络与其他系统共享目录。客户端可以挂载这些共享目录，像使用本地文件系统一样使用它们。

本章介绍 OpenBSD 上的 NFS 用法，涵盖服务器与客户端两种角色、守护进程管理、安全选项以及部署最佳实践。

## 服务器配置

作为 NFS 服务器的 OpenBSD 系统必须运行 `mountd(8)` 与 `nfsd(8)` 守护进程。若需要文件锁定，还应启用 `rpc.lockd(8)` 与 `rpc.statd(8)`。此外，NFS 的运行离不开 `portmap(8)`。

共享目录必须在 **/etc/exports** 中声明。例如，要将 **/export/projects** 共享给本地网络：

```conf
/export/projects -alldirs -maproot=root: -network 192.168.1.0 -mask 255.255.255.0
```

`-alldirs` 选项允许客户端挂载子目录。`-maproot=root:` 指令为受信任的客户端保留 root 访问权限。`-network` 与 `-mask` 选项将访问限制在特定子网。

编辑 **/etc/exports** 后，必须重新加载 `mountd` 守护进程：

```sh
# kill -HUP $(cat /var/run/mountd.pid)
```

创建共享目录并设置合适权限：

```sh
# mkdir -p /export/projects
# chown root:users /export/projects
# chmod 775 /export/projects
```

服务器守护进程可按以下方式启用并启动：

```sh
# rcctl enable portmap mountd nfsd
# rcctl start portmap
# rcctl start mountd
# rcctl start nfsd
```

支持锁定：

```sh
# rcctl enable rpc.lockd rpc.statd
# rcctl start rpc.lockd
# rcctl start rpc.statd
```

若启用了 `pf(4)`，可能需要放行 `sunrpc` 与 `2049` 上的 TCP 和 UDP 流量，或者在 **/etc/rc.conf.local** 中通过 `mountd_flags` 与 `nfsd_flags` 指定静态端口。

## 客户端配置

NFS 客户端不需要额外软件包。手动挂载远程导出：

```sh
# mount -t nfs 192.168.1.10:/export/projects /mnt
```

要使挂载持久化，将其添加到 **/etc/fstab**：

```
192.168.1.10:/export/projects /mnt nfs rw 0 0
```

`soft`、`bg` 或 `nolock` 等额外挂载选项在某些环境中可能有用，尤其是远程服务器不可靠或不支持锁定时。例如：

```
192.168.1.10:/export/projects /mnt nfs rw,bg,soft 0 0
```

客户端需要 `portmap` 服务：

```sh
# rcctl enable portmap
# rcctl start portmap
```

卸载文件系统：

```sh
# umount /mnt
```

## 服务行为与选项

NFS 请求可使用 TCP 或 UDP。OpenBSD 默认使用 TCP。若使用防火墙，必须放行对 `mountd`、`nfsd` 与 `portmap` 端口的访问。OpenBSD 的 `nfsd` 支持这两种协议，但除非显式配置，否则不默认动态分配端口。

`mount_nfs(8)` 工具识别多种影响行为的选项。`hard` 选项（默认）使系统调用无限重试。`soft` 选项允许调用在超时后失败。使用 `bg` 会使失败的挂载尝试在后台重试。这些设置应根据应用对延迟或失败的容忍度来选择。

## 访问控制

**/etc/exports** 中的 `maproot` 与 `mapall` 选项控制客户端用户 ID 如何映射到本地用户。将 root 映射为 `nobody` 是常见的防范远程 root 访问的安全措施：

```conf
/export/projects -maproot=nobody -network 192.168.1.0 -mask 255.255.255.0
```

向不受信任的客户端导出可写目录时须谨慎。建议按网络限制访问、使用只读导出，并在不需要 root 映射时赋予最小权限。

## 监控与调试

确认导出：

```sh
# showmount -e localhost
```

验证服务状态：

```sh
# rcctl check mountd
# rcctl check nfsd
```

使用 `tail -f /var/log/messages` 或 `dmesg` 查看错误或挂载失败。在客户端，加 `-v` 挂载可能提供更多输出：

```sh
# mount -v -t nfs server:/path /mnt
```

## 兼容性说明

OpenBSD 的 NFS 实现与使用 NFSv2 或 NFSv3 的其他 Unix 系统兼容。不支持 NFSv4。某些 Linux 客户端可能默认使用 NFSv4，与 OpenBSD 互操作时必须显式配置为使用更早的版本：

```sh
# mount -t nfs -o vers=3 openbsd:/export /mnt
```

`nfsd` 与 `mountd` 守护进程不使用 `inetd`，必须通过 `rcctl` 运行或用本地 `rc` 钩子管理。
