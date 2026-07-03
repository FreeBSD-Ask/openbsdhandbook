# 串口通信

## 概述

串口通信为 OpenBSD 系统提供关键功能，尤其在无头服务器、嵌入式系统、硬件调试与网络设备等场景中不可或缺。OpenBSD 基本系统完整支持串口终端的配置与使用，包括通过串口控制台登录、用 `cu` 与 `tip` 等工具连接串口设备、配置 USB 转串口适配器，以及验证串口功能。

本章介绍如何通过 **/etc/ttys** 配置串口登录访问、如何通过 **/etc/boot.conf** 在启动时使用串口控制台、如何用基本系统工具操作串口，以及如何测试与验证串口硬件和通信设置。

## 启用串口控制台登录

OpenBSD 上串口通常以 **/dev/tty00**、**/dev/tty01** 等形式出现。要在串口上启用登录，需修改 **/etc/ttys** 文件。

配置第一个串口（`tty00`）用于登录：

```
tty00   "/usr/libexec/getty std.9600"   vt220   on secure
```

此条目指示 `init` 在 **/dev/tty00** 上启动 `getty(8)`，使用 `gettytab(5)` 中的 `std.9600` 条目。终端类型 `vt220` 适用于大多数串口终端。`secure` 关键字允许在该线路上以 root 登录。

激活更改：

```sh
# kill -HUP 1
```

此命令向 `init(8)` 进程发送信号，使其重新加载配置并在指定线路上启动 `getty`。

确保没有其他进程占用串口。可用 `fstat` 或 `ps` 确认。

## 使用 cu 与 tip

OpenBSD 基本系统内置两个用于与串口设备交互的终端通信程序：`cu(1)` 与 `tip(1)`。

### 使用 `cu`

`cu` 工具可直接连接串口设备。以 9600 波特率打开 **/dev/tty00** 的连接：

```sh
# cu -l /dev/tty00 -s 9600
```

退出会话时，在新行首输入 `~.`。

### 使用 `tip`

`tip` 工具可交互使用，也可配合 **/etc/remote** 中定义的配置条目。不定义别名直接调用：

```sh
# tip -9600 tty00
```

在 **/etc/remote** 中创建可复用的条目：

```conf
console:\
	:dv=/dev/tty00:br#9600:pa=none:
```

然后连接：

```sh
# tip console
```

退出的转义序列相同：在新行输入 `~.`。

## 在启动时使用串口控制台

配置 OpenBSD 将引导加载器与内核消息输出到串口，需编辑 **/etc/boot.conf**。

通过第一个串口（`com0`，对应 **/dev/tty00**）启用串口控制台输出：

```
set tty com0
```

此命令将引导加载器与内核的输出重定向到串口，通常为 9600 波特率。

从另一台机器通过 USB 转串口适配器与本系统交互时，使用 `cu`：

```sh
# cu -l /dev/cuaU0 -s 115200
```

中断引导加载器并输入：

```
set tty com0
boot
```

此操作将引导与内核输出同时定向到串口。

## USB 串口适配器

现代系统通常没有传统 RS-232 端口。OpenBSD 通过以下驱动支持多种 USB 转串口适配器：

- `ucom(4)` — 通用 USB 串口接口
- `uplcom(4)` — 基于 Prolific 的设备
- `umodem(4)` — USB 调制解调器
- `uvscom(4)` — SUNTAC 串口设备

适配器连接后，内核会记录检测结果：

```
uplcom0 at uhub1 port 2: Prolific Technology USB-Serial Controller
ucom0 at uplcom0
```

设备节点将以如下形式出现：

- **/dev/cuaU0** — 用于外发连接（`cu`、`tip`）
- **/dev/ttyU0** — 用于传入连接（如登录）

以 115200 波特率连接：

```sh
# cu -l /dev/cuaU0 -s 115200
```

## 测试串口

串口可用多种技术测试。

### 检查设备是否存在

确认相关设备节点存在：

```sh
# ls -l /dev/tty00
```

### 环回测试

物理连接发送与接收引脚（如 DB9 接口的 2、3 脚），然后：

```sh
# cu -l /dev/tty00 -s 9600
```

输入的字符应回显到屏幕。

### 监控端口使用情况

用 `fstat` 确定是否有进程正在使用某串口设备：

```sh
# fstat | grep tty00
```

也可用 `systat` 监控实时活动：

```sh
# systat tty
```

### 检查内核消息

检查内核日志中检测到的串口接口：

```
com0 at isa0 port 0x3f8/8 irq 4: ns16550a, 16 byte fifo
```

基于 USB 的适配器会显示 `ucom`、`uplcom` 或类似设备行。

## 串口参数与流控

串口通信依赖两端一致的设置。常见参数包括：

- **波特率**：如 9600、19200 或 115200
- **数据位**：通常为 8（`cs8`）
- **校验位**：无（`-parenb`），或 `even`、`odd`
- **停止位**：通常为 1（`-cstopb`）
- **流控**：无、XON/XOFF 或 RTS/CTS

手动配置这些设置：

```sh
# stty -f /dev/tty00 115200 cs8 -cstopb -parenb
```

此命令在 **/dev/tty00** 上设置 115200 波特率、8 数据位、1 停止位、无校验。

## 通过串口调试内核

在无头或嵌入式系统上使用 OpenBSD 内核调试器（`ddb`），或远程调试内核 panic 时，串口通信至关重要。串口控制台配置正确后，`ddb` 的输入输出可重定向到串口线路。

要通过串口启用内核调试，系统须先配置为使用串口控制台。包括：

1. **配置 `/etc/boot.conf`**：

   将控制台输出重定向到第一个串口（`com0`，通常对应 **/dev/tty00**）：

   ```
   set tty com0
   ```
2. **确保 `ddb` 已启用**：

   默认 OpenBSD 内核包含 `ddb`。要确认其可访问，检查以下 `sysctl` 值：

   ```sh
   # sysctl ddb.console ddb.panic
   ddb.console=1
   ddb.panic=1
   ```

   这些设置表示：

   - `ddb.console=1`：可从控制台访问内核调试器
   - `ddb.panic=1`：内核 panic 时进入 `ddb`

   要使这些设置持久化，在 **/etc/sysctl.conf** 中添加：

   ```conf
   ddb.console=1
   ddb.panic=1
   ```
3. **从串口使用调试器**：

   配置完成后，系统 panic 或手动进入 `ddb`（如用 `sysctl ddb.trigger=1`）时，提示符会出现在串口控制台。可输入 `trace`、`ps`、`show registers`、`reboot` 等命令。

   ```sh
   ddb> trace
   ddb> ps
   ddb> reboot
   ```
4. **手动触发 `ddb`**：

   用以下命令手动中断进入调试器（用于测试或调试）：

   ```sh
   # sysctl ddb.trigger=1
   ```

   此命令将系统从当前控制台转入 `ddb`。使用串口控制台时，提示符会出现在串口线路上。
5. **处理无本地显示的系统**：

   缺少 VGA 或帧缓冲输出的系统（如许多嵌入式平台），串口控制台是唯一的 `ddb` 交互途径。因此，在系统启动或平台移植期间，尽早配置串口控制台输出至关重要。

通过串口调试内核可在 panic 后恢复、检查系统状态，配合远程串口捕获还能实现自动崩溃记录。此配置完全由基本系统工具支持，无需额外软件。

## 将 OpenBSD 用作控制台服务器

OpenBSD 系统可作为可靠的控制台服务器，为交换机、无头服务器、防火墙或嵌入式平台等一台或多台远程设备提供串口访问。此类部署常见于数据中心、实验室或需要带外管理的远程站点。

### 硬件准备

多数现代系统没有传统串口，但可用 USB 转串口适配器扩展。OpenBSD 通过 `uplcom`、`umodem`、`uvscom`、`ucom` 等驱动支持多种此类适配器。基于 `ns8250` 的多端口 PCI 或 PCIe 串口卡也通过 `com(4)` 驱动支持。

连接 USB 串口适配器后，设备会以 **/dev/cuaU0**、**/dev/cuaU1** 等形式出现。可用 `dmesg` 查看内核消息：

```
uplcom0 at uhub1 port 3: Prolific USB-Serial Controller
ucom0 at uplcom0
```

在 OpenBSD 系统与受管理设备之间使用正确的空调制解调器线缆。

### 连接设备

每个串口可用 `cu` 交互访问：

```sh
# cu -l /dev/cuaU0 -s 115200
```

为简化访问，可在 **/etc/remote** 中定义命名条目：

```conf
router1:\
	:dv=/dev/cuaU0:br#115200:pa=none:
switch1:\
	:dv=/dev/cuaU1:br#9600:pa=none:
```

然后连接：

```sh
# tip router1
```

### 管理多个连接

运维人员同时管理多台设备时，可用 `tmux` 为每个串口会话分配一个窗格。示例：

```sh
$ tmux
$ cu -l /dev/cuaU0 -s 9600   # 窗格 1
$ cu -l /dev/cuaU1 -s 115200 # 窗格 2
```

也可用包装脚本或 shell 别名实现快速访问。

### 可选：在串口上启用登录

若需要通过串口入站访问 OpenBSD 控制台服务器，可修改 **/etc/ttys** 在每个端口上启动登录会话：

```
ttyU0   "/usr/libexec/getty std.9600"   vt220   on secure
ttyU1   "/usr/libexec/getty std.115200" vt220   on secure
```

重新加载配置：

```sh
# kill -HUP 1
```

这样用户即可通过串口连接 OpenBSD 系统执行管理任务。

### 安全考量

为确保适当隔离：

- 用 `doas.conf` 限制可运行 `cu` 或 `tip` 的用户
- 配置 `login.conf` 限制资源使用
- 通过受限 shell 或 shell 包装器记录访问与命令
- 仅需外发连接的端口不要启动 `getty`

控制台服务器本身的访问应通过基于密钥认证的 `ssh` 保护，并尽可能限制物理访问。
