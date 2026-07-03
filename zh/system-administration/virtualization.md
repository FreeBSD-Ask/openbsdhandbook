# 虚拟化

虚拟化是一种创建隔离环境的技术，这些隔离环境称为虚拟机（VM），软件在其中运行如同在物理机上一样。该技术允许多个虚拟实例共享物理资源，每个实例表现为独立系统。OpenBSD 通过 **vmm(4)** 子系统提供轻量、安全的虚拟化方案。

## OpenBSD 的虚拟化方案

OpenBSD 原生虚拟化支持以 **vmm(4)** 驱动为核心，又称“虚拟机监视器”。该子系统可在 OpenBSD 内直接创建、管理并运行虚拟机。**vmm(4)** 最早出现于 OpenBSD 5.9，每个版本持续改进。OpenBSD 的虚拟化设计以安全为重点。

OpenBSD 上的虚拟机可运行 OpenBSD 自身或其他操作系统，前提是它们与 **vmm(4)** 的硬件和虚拟化要求兼容。

### 关键组件

1. **vmm(4) — 虚拟机监视器**：该内核驱动负责 OpenBSD 的核心虚拟化功能，管理虚拟机的创建与控制，并与宿主系统交互以分配资源。
2. **vmd(8) — 虚拟机守护进程**：**vmd** 是 **vmm(4)** 的用户态对应组件，提供创建、启动、停止与管理虚拟机所需的控制机制。该守护进程可通过简单的配置文件进行配置，便于维护。
3. **vmctl(8) — 虚拟机控制工具**：**vmctl** 是与虚拟化系统交互的主要接口，管理员可用它管理虚拟机、创建新的虚拟机镜像、分配 CPU 与内存等资源，以及查看现有虚拟机的状态。

### 硬件虚拟化支持

OpenBSD 需要硬件辅助虚拟化功能才能高效运行虚拟机。这些功能在 modern Intel 与 AMD 处理器上提供，通常通过 BIOS 或 UEFI 设置启用。所需功能为：

- **Intel VT-x**：Intel 的硬件虚拟化技术。
- **AMD SVM（Secure Virtual Machine）**：AMD 对应 Intel VT-x 的技术。

若不具备这些功能，OpenBSD 的虚拟化将无法支持，因其依赖硬件提供必要的虚拟化扩展。

#### 检查 CPU 是否支持虚拟化

检查系统 CPU 是否支持硬件辅助虚拟化：

```sh
$ dmesg | egrep '(VMX/EPT|SVM/RVI)'
```

- **VMX/EPT** 表示支持 Intel 的硬件辅助虚拟化。
- **SVM/RVI** 表示支持 AMD 的硬件辅助虚拟化。

若输出包含这些标志，则系统支持所需的硬件虚拟化技术。

### 关键文件

OpenBSD 的虚拟化系统涉及若干重要文件，用于配置与管理虚拟机。这些文件涵盖从定义虚拟机设置到管理网络接口与路由流量的方方面面。

#### /etc/vm.conf

**vm.conf** 是 **vmd(8)** 守护进程的主配置文件。该文件用于持久化配置虚拟机，包括内存、CPU、磁盘与网络设置，还可定义虚拟交换机，让虚拟机接入宿主机网络或与其他虚拟机通信。

关于**桥接网络**的设置，详见[桥接网络章节](#bridged-networking)
。

**vm.conf** 中的示例条目：

```
switch "uplink" {
    interface bridge0
}

vm "vm1" {
    memory 1G
    disk "/path/to/disk.img"
    interface "uplink"
}
```

对该文件的修改需通过 `rcctl(8)` 命令重启或重载 **vmd** 后方可生效。

#### /var/run/vmd.sock

这是 **vmctl(8)** 与 **vmd** 守护进程通信的控制套接字，是发送给 **vmd** 的所有命令（如启动、停止或修改虚拟机）的连接点。用户通常不直接与此文件交互，但它对管理虚拟机至关重要。

#### 虚拟磁盘镜像

虚拟机磁盘镜像通常存放在用户指定的目录，例如 `/var/vm/`。这些镜像充当虚拟机的硬盘，可使用 **vmctl(8)** 或外部工具创建。镜像可在 **vm.conf** 中引用，或在用 **vmctl** 启动虚拟机时直接指定。

创建磁盘镜像示例：

```
# vmctl create /var/vm/disk.img -s 40G
```

#### /etc/hostname.if

该文件配置宿主机的网络接口，包括用于虚拟机网络的桥接接口。在**桥接网络**部署中，需在此定义类似 **bridge0** 的桥接接口，将宿主机的物理网络接口与虚拟机相连。详见[桥接网络章节](#bridged-networking)
。

**hostname.bridge0** 示例：

```
add em0
up
```

上述示例将 `bridge0` 配置为包含物理接口 `em0`。

#### /etc/pf.conf

为虚拟机使用 **NAT（网络地址转换）** 时，须配置 OpenBSD 的包过滤器 **pf(4)** 以转换虚拟机的网络流量。**pf.conf** 文件中包含处理此类流量转换的规则。

关于 NAT 的详细设置，参见 [NAT 网络章节](#nat-network-address-translation-networking)
。

**pf.conf** 中的 NAT 规则示例：

```pf
ext_if="em0"
nat on $ext_if from 10.0.0.0/24 to any -> ($ext_if)
```

上述规则将来自虚拟机（`10.0.0.0/24` 子网）的流量转换成宿主机的外部 IP 地址，用于对外通信。

## 运行虚拟机

### 运行虚拟机的准备工作

启用并正确配置 **vmm(4)** 子系统后，即可在 OpenBSD 上创建并运行虚拟机。本节介绍启用虚拟机监视器、准备虚拟磁盘镜像以及启动一台基础虚拟机的步骤。

#### 1. 启用并启动虚拟机监视器（vmm）

运行虚拟机前，须启用并启动管理虚拟机的 **vmd(8)** 守护进程。要让 **vmm(4)** 在系统引导时启用，需添加以下配置：

##### 开机启用 vmd

编辑 `/etc/rc.conf.local`，加入以下行，指示 OpenBSD 在引导时自动启动 **vmd**：

```conf
vmd_flags=""
```

若该文件不存在，则创建并加入此条目。这样系统会以无特殊标志启动 **vmd**。

##### 手动启动 vmd

不重启系统而手动启动 **vmd**：

```sh
# rcctl start vmd
```

该命令启动 **vmd** 守护进程，使系统具备创建与管理虚拟机的能力。要让 **vmd** 在后续重启时自动启动：

```sh
# rcctl enable vmd
```

#### 2. 验证 vmd 是否运行

验证 **vmd** 是否正在运行：

```sh
# rcctl check vmd
```

该命令显示 **vmd** 守护进程的状态，指明其是否处于活动状态。

### 创建并运行虚拟机

**vmd** 运行后，下一步是创建虚拟机。这需要创建虚拟磁盘镜像、获取安装 ISO，并启动虚拟机。

#### 1. 创建虚拟磁盘镜像

虚拟磁盘镜像充当虚拟机的硬盘。可使用 **vmctl(8)** 工具创建指定大小的虚拟磁盘镜像。例如创建一个 40 GB 的磁盘镜像：

```
# vmctl create -s 40G disk.img
```

该命令创建名为 `disk.img` 的文件，作为虚拟机的存储。

#### 2. 获取安装 ISO

要在虚拟机中安装操作系统，需要安装 ISO。例如安装 OpenBSD，可使用 **ftp** 工具下载安装 ISO：

```sh
$ ftp https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.iso
```

该命令下载 OpenBSD 7.8 安装 ISO。该 ISO 文件用于引导虚拟机并安装操作系统。

#### 3. 启动虚拟机

虚拟磁盘镜像与安装 ISO 准备就绪后，即可使用 **vmctl(8)** 启动虚拟机。以下命令启动虚拟机并连接到其控制台，便于直接交互：

```
# vmctl start -c -m 1G -L -r install78.iso -d disk.img "vm1"
```

- `-c`：连接到控制台以便交互使用。
- `-m 1G`：为虚拟机分配 1 GB 内存。
- `-L`：添加本地网络接口。
- `-r install78.iso`：指定引导用的安装 ISO。
- `-d disk.img`：指定先前创建的虚拟磁盘镜像。
- `"vm1"`：虚拟机名称。

启动后，虚拟机将从 ISO 引导，可在虚拟环境中安装操作系统。过程中用户通过控制台与虚拟机交互。

**需要重定向控制台**
安装过程中提示时，请将控制台重定向至 **com0**，否则虚拟控制台不会出现登录提示。网络配置完成后仍可使用 SSH 登录。

**首次引导后的网络接口**
虚拟机首次引导后，宿主系统会有一块与该虚拟机对应的 **tap** 接口，客户系统则有一块 **vio0** 接口用于网络访问。

**宿主机 Tap 接口（tap0）：**

```conf
tap0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
  lladdr fe:e1:ba:d4:3e:e2
  description: vm1-if0-vm1
  index 10 priority 0 llprio 3
  groups: tap
  status: active
  inet 100.64.1.2 netmask 0xfffffffe
```

宿主机上的 **tap0** 接口对应虚拟机的网络连接，具有链路本地地址，充当宿主网络与客户网络之间的桥梁。

**客户机网络接口（vio0）：**

```conf
vio0: flags=808b43<UP,BROADCAST,RUNNING,PROMISC,ALLMULTI,SIMPLEX,MULTICAST,AUTOCONF4> mtu 1500
  lladdr fe:e1:bb:d1:1e:fb
  index 1 priority 0 llprio 3
  groups: egress
  media: Ethernet autoselect
  status: active
  inet 100.64.1.3 netmask 0xfffffffe
```

客户机内的 **vio0** 接口提供网络连接，并自动配置。客户机 IP 地址与宿主机上的 tap 接口位于同一子网，二者可相互通信。

#### 4. 管理虚拟机

操作系统安装完成后，可按需停止或启动虚拟机。停止虚拟机：

```
# vmctl stop "vm1"
```

该命令停机虚拟机。之后不使用安装 ISO 重启虚拟机：

```
# vmctl start -c -m 1G -d disk.img "vm1"
```

虚拟机将从虚拟磁盘而非 ISO 引导，加载初始安装时安装的操作系统。

#### 5. 查看虚拟机状态

查看运行中虚拟机的状态：

```
# vmctl status
```

该命令提供所有活动虚拟机的信息，包括名称、ID、已分配内存与当前状态。

### 使用 vmctl 连接虚拟机

虚拟机运行后，可通过 **vmctl(8)** 经控制台访问。控制台提供与虚拟机的直接交互，功能类似物理机的终端。

#### 启动虚拟机时连接控制台

启动虚拟机时可使用 `-c` 标志，自动打开控制台并建立直接交互：

```
# vmctl start -c -m 1G -d disk.img "vm1"
```

`-c` 标志打开控制台，允许在虚拟机启动后立即与之交互，适用于安装或配置操作系统。

#### 连接到已在运行的虚拟机

若虚拟机已在运行但启动时未打开控制台，仍可使用 **vmctl(8)** 连接。

首先列出所有活动虚拟机：

```
# vmctl status
```

该命令显示运行中的虚拟机及其对应 ID。例如：

```
   ID   PID VCPUS  MAXMEM  CURMEM     TTY        OWNER    STATE NAME
    1 85075     1    1.0G   1006M   ttyp8         root  running vm1
```

使用 **vmctl status** 输出中的 ID 连接到指定虚拟机的控制台。例如虚拟机 ID 为 `1`：

```
# vmctl console 1
```

该命令打开 ID 为 `1` 的虚拟机控制台，允许直接与其终端交互。

#### 退出控制台

要在不停止虚拟机的情况下退出控制台，按 `Enter` 后接 `~.`（波浪号后接句点）：

```
Enter ~.
```

该组合键断开控制台会话，虚拟机仍在后台运行。

#### 使用 SSH 连接（操作系统安装后）

在虚拟机中安装操作系统并配置网络后，若网络可达，可通过 **SSH** 连接到虚拟机。

使用虚拟机的 IP 地址通过 SSH 连接：

```sh
$ ssh user@<vm_ip_address>
```

将 `<vm_ip_address>` 替换为虚拟机的 IP 地址，`user` 替换为相应用户名。

## 让虚拟机配置持久化

要让虚拟机配置在重启后仍然保留，可将设置加入 **/etc/vm.conf** 文件。该配置允许 **vmd(8)** 自动管理虚拟机，并确保其在引导时启动。

### 配置虚拟机

打开或创建 **/etc/vm.conf** 文件：

```
# vi /etc/vm.conf
```

为虚拟机添加配置，指定内存、磁盘镜像与接口配置。例如创建名为 `vm1` 的虚拟机，配置 1 GB 内存、一个磁盘镜像并自动启动：

```
vm "vm1" {
    memory 1G
    enable
    disk /var/vm/disk.img
    local interface
}
```

上述示例中：

- `vm "vm1"` 定义虚拟机名称。
- `memory 1G` 为虚拟机分配 1 GB 内存。
- `enable` 确保系统引导启动 **vmd(8)** 时该虚拟机自动启动。
- `disk /var/vm/disk.img` 指定虚拟机磁盘镜像路径。
- `local interface` 创建一个虚拟网络接口，让虚拟机通过仅本地网络连接与宿主机通信。

放置好 **/etc/vm.conf** 后（重新）启动 **vmd** 守护进程，运行 `vmctl status` 即可看到该虚拟机，无论其处于何种状态。例如：

```
   ID   PID VCPUS  MAXMEM  CURMEM     TTY        OWNER    STATE NAME
    1     -     1    1.0G       -       -         root  stopped vm1
```

### 管理本地接口

配置中指定 `local interface` 后，**vmd(8)** 会在宿主机上自动创建 **tap** 接口，并在客户机中创建对应的 **vio0** 接口。该仅本地接口允许宿主机与虚拟机之间通信，不接入外部网络。

- 宿主机上的 **tap** 接口（如 `tap0`）如下：

  ```conf
  tap0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> mtu 1500
    lladdr fe:e1:ba:d4:3e:e2
    description: vm1-if0-vm1
    index 10 priority 0 llprio 3
    groups: tap
    status: active
    inet 100.64.1.2 netmask 0xfffffffe
  ```
- 客户机内的 **vio0** 接口配置如下：

  ```conf
  vio0: flags=808b43<UP,BROADCAST,RUNNING,PROMISC,ALLMULTI,SIMPLEX,MULTICAST,AUTOCONF4> mtu 1500
    lladdr fe:e1:bb:d1:1e:fb
    index 1 priority 0 llprio 3
    groups: egress
    media: Ethernet autoselect
    status: active
    inet 100.64.1.3 netmask 0xfffffffe
  ```

上述配置允许客户机与宿主机之间进行本地通信，无需外部网络访问。

### 确保 vmd 开机启动

要让 **vmd(8)** 与已配置的虚拟机在引导时自动启动，需使用 **rcctl** 启用 **vmd**：

```sh
# rcctl enable vmd
```

立即启动 **vmd**：

```sh
# rcctl start vmd
```

### 重载 vmd 配置

重载 **vmd(8)** 配置以应用更改：

```sh
# rcctl reload vmd
```

该命令指示 **vmd** 从 **/etc/vm.conf** 加载更新后的配置，无需重启服务。

## 网络选项

OpenBSD 的虚拟化子系统 **vmm(4)** 支持多种网络配置，允许虚拟机与宿主系统及外部网络通信。虚拟机可接入物理接口、被隔离，或在私有环境中联网。**vmd(8)** 守护进程负责为虚拟机设置网络，可根据所需的网络行为选择不同选项。

### 桥接网络

桥接网络让虚拟机在本地网络上表现为常规设备，相当于共享宿主机的物理网络接口。此模式下，虚拟机可从网络路由器或 DHCP 服务器获取自己的 IP 地址（如通过 DHCP），并与网络上的其他机器通信，如同独立系统。

#### 设置桥接网络

须创建桥接接口，将宿主机网络接口与虚拟机的虚拟接口相连。桥接把虚拟机的网络流量与宿主机流量聚合。以下命令演示如何创建并配置桥接：

```sh
# ifconfig bridge0 create
# ifconfig bridge0 add em0
```

此处 `em0` 是宿主机的物理网络接口。创建桥接后，将虚拟机的虚拟网络接口分配给该桥接，即可让虚拟机接入。

在 **vm.conf(5)** 配置文件中，将桥接接口作为虚拟机配置的一部分：

```
switch "uplink" {
    interface bridge0
}

vm "vm1" {
    memory 1G
    disk "/path/to/disk.img"
    interface { switch "uplink" }
}
```

虚拟机启动后，其网络流量将经由 `bridge0` 接口，可在与宿主机相同的网络上通信。

#### 让桥接配置持久化

要让 **bridge0** 配置在 OpenBSD 中持久化，须将配置保存到 **/etc/hostname.bridge0** 对应的接口配置文件中。这样可在引导时自动创建并配置桥接，无需每次手动执行 `ifconfig` 命令。

1. **为 bridge0 创建 hostname 文件：**

   打开或创建 **/etc/hostname.bridge0** 文件。该文件存储引导时自动设置 **bridge0** 所需的配置。

   ```
   # vi /etc/hostname.bridge0
   ```
2. **添加必要的配置：**

   在 **/etc/hostname.bridge0** 中加入以下行，永久创建桥接并加入物理接口（如 `em0`）：

   ```
   add em0
   up
   ```
   - `add em0`：将物理接口 `em0` 加入桥接。
   - `up`：确保引导时激活桥接接口。
3. **配置物理接口（可选）：**

   若 **em0** 接口需要特定配置，可在 **/etc/hostname.em0** 中定义。例如让 `em0` 在引导时启动且不指定 IP 配置：

   ```
   up
   ```

   这样确保 **em0** 已就绪供桥接使用。
4. **应用更改：**

   要在不重启的情况下测试配置，运行以下命令手动启用桥接并应用新配置：

   ```
   # sh /etc/netstart bridge0
   ```

   该命令应用 **/etc/hostname.bridge0** 中的配置，启用 **bridge0** 并加入 `em0` 接口。

##### 验证桥接

验证 **bridge0** 是否处于活动状态并已正确配置：

```sh
# ifconfig bridge0
```

输出应显示 **bridge0** 已启用并包含 **em0** 接口。

### NAT（网络地址转换）网络

采用网络地址转换（NAT）时，虚拟机对外通信共享宿主系统的 IP 地址。此模式适用于虚拟机需要访问互联网、但网络 IP 地址有限，或虚拟机不应暴露给外部网络的场景。

#### 设置 NAT 网络

在该配置中，宿主系统充当路由器，将虚拟机的流量转发到外部网络。**pf(4)**（包过滤器）用于设置 NAT。确保 **pf** 已启用并配置为处理 NAT：

##### 启用 pf

OpenBSD 默认启用 **pf**，若未运行，可使用以下命令启动：

```sh
# pfctl -e
```

##### 在 /etc/pf.conf 中设置 NAT 规则

在 **/etc/pf.conf** 文件中加入以下 NAT 规则，为虚拟机启用 NAT：

```pf
# 为虚拟机设置 NAT
ext_if="em0"  # 外部接口
match out on $ext_if from 100.64.0.0/10 to any nat-to $ext_if
pass out on $ext_if from 100.64.0.0/10 to any nat-to $ext_if
```

上述示例中：

- `ext_if="em0"`：`em0` 是宿主系统的外部接口。
- 虚拟机使用 `100.64.0.0/10` 地址段，宿主机会将其流量转换为外部 IP 地址。

##### 测试 pf 配置

应用更改前，可测试 **pf.conf** 文件的语法是否正确：

```sh
# pfctl -n -f /etc/pf.conf
```

##### 加载 pf 配置

应用更改，加载更新后的 **/etc/pf.conf** 配置：

```sh
# pfctl -f /etc/pf.conf
```

##### 启用包转发

宿主系统须启用包转发以充当路由器。可使用以下命令：

```sh
# sysctl net.inet.ip.forwarding=1
```

要让该配置持久化，在 **/etc/sysctl.conf** 中加入以下行：

```conf
net.inet.ip.forwarding=1        # 允许 IPv4 转发（路由）
```

##### 为虚拟机配置 /etc/vm.conf

接下来配置 **/etc/vm.conf**，为虚拟机创建本地接口：

```
vm "vm1" {
    memory 1G
    disk "/var/vm/disk.img"
    local interface
}
```

该配置定义名为 `vm1` 的虚拟机，配置 1 GB 内存、位于 `/var/vm/disk.img` 的磁盘镜像，以及一个本地网络接口。

##### 启动虚拟机

虚拟机（重新）启动时，将使用内部网络接口（如 `vmtap0`），**pf** 处理 NAT，使虚拟机可经宿主机访问外部网络。

### 仅主机网络

仅主机网络将虚拟机的网络流量与外部网络隔离，仅允许宿主机与虚拟机之间通信。此模式适用于测试环境，或不需要外部网络访问但需要虚拟机与宿主机通信的场景。

#### 设置仅主机网络

在该配置中，虚拟交换机将虚拟机连接到宿主机，但不将流量路由到外部网络。桥接中不加入任何外部接口，保持通信隔离。**vm.conf** 配置如下：

```
vm "vm1" {
    memory 1G
    disk "/path/to/disk.img"
    local interface
}
```

该配置不含任何物理接口，虚拟机只能通过虚拟交换机与宿主机通信。

### 隔离网络

在隔离网络中，虚拟机与宿主机及外部网络完全断开。此模式适用于安全测试，或必须完全禁止外部通信的环境。

#### 设置隔离网络

不为虚拟机配置任何网络接口，即可让其与任何网络通信完全隔离。在 **/etc/vm.conf** 中完全省略 `interface` 行：

```
vm "vm1" {
    memory 1G
    disk "/path/to/disk.img"
}
```

该配置创建一台无网络接口的虚拟机，确保与其他机器、宿主机及外部网络完全隔离。

## 命令参考

### vmctl(8) 命令概览

| 命令 | 说明 | 示例 |
| --- | --- | --- |
| `vmctl create -s <size> <disk.img>` | 创建指定大小的虚拟磁盘镜像。 | `vmctl create -s 40G /var/vm/disk.img` |
| `vmctl start -m <memory> -d <disk.img> <name>` | 以指定名称、内存分配与磁盘镜像启动虚拟机。 | `vmctl start -m 1G -d /var/vm/disk.img "vm1"` |
| `vmctl start -c <name>` | 启动虚拟机并连接到其控制台。`-c` 选项在启动时打开虚拟机控制台。 | `vmctl start -c -m 1G -d /var/vm/disk.img "vm1"` |
| `vmctl stop <id>` | 按 ID 停止运行中的虚拟机。 | `vmctl stop 1` |
| `vmctl console <id>` | 按 ID 连接到运行中虚拟机的控制台。 | `vmctl console 1` |
| `vmctl status` | 显示所有虚拟机的状态，包括 ID、名称、内存与状态（运行/停止）。 | `vmctl status` |
| `vmctl reload` | 从 `/etc/vm.conf` 文件重载 **vmd(8)** 守护进程配置。 | `vmctl reload` |
| `vmctl wait` | 等待 **vmd** 完成引导并加载配置。 | `vmctl wait` |
| `vmctl terminate` | 终止 **vmd** 守护进程（类似关机）。 | `vmctl terminate` |
| `vmctl load <filename>` | 从文件加载已保存的虚拟机状态。 | `vmctl load /var/vm/saved_vm_state.img` |
| `vmctl dump <id> -f <filename>` | 将运行中虚拟机的状态转储到文件，便于后续恢复。 | `vmctl dump 1 -f /var/vm/saved_vm_state.img` |
| `vmctl reset <id>` | 重置运行中的虚拟机，类似物理机上的硬复位。 | `vmctl reset 1` |
| `vmctl pause <id>` | 按 ID 暂停运行中的虚拟机。 | `vmctl pause 1` |
| `vmctl unpause <id>` | 按 ID 取消暂停处于暂停状态的虚拟机。 | `vmctl unpause 1` |
| `vmctl show <id>` | 显示虚拟机的详细信息，包括磁盘用量、内存分配与网络设置。 | `vmctl show 1` |

### 其他相关命令

| 命令 | 说明 | 示例 |
| --- | --- | --- |
| `rcctl enable vmd` | 启用 **vmd(8)** 守护进程开机自启。 | `rcctl enable vmd` |
| `rcctl start vmd` | 手动启动 **vmd(8)** 守护进程。 | `rcctl start vmd` |
| `rcctl stop vmd` | 手动停止 **vmd(8)** 守护进程。 | `rcctl stop vmd` |
| `rcctl reload vmd` | 从 **/etc/vm.conf** 重载 **vmd(8)** 守护进程配置。 | `rcctl reload vmd` |
| `ifconfig bridge0 create` | 创建用于虚拟机的网络桥接接口。 | `ifconfig bridge0 create` |
| `ifconfig bridge0 add <iface>` | 将物理接口加入桥接以实现网络桥接。 | `ifconfig bridge0 add em0` |
| `ifconfig bridge0 up` | 启用桥接接口。 | `ifconfig bridge0 up` |
| `dmesg | egrep ‘(VMX/EPT | SVM/RVI)'` |
| `pfctl -f /etc/pf.conf` | 从 **/etc/pf.conf** 重载 **pf(4)** 规则。为虚拟机设置 NAT 时有用。 | `pfctl -f /etc/pf.conf` |
| `ftp <url>` | 下载文件，如安装 ISO，供虚拟机使用。 | `ftp https://cdn.openbsd.org/pub/OpenBSD/7.8/amd64/install78.iso` |
