# 存储与文件系统

## 概要

OpenBSD 为磁盘、分区、文件系统以及加密、软件 RAID 等高级存储选项提供了一套一致且安全的框架。本章介绍如何检测新磁盘，选用合适的分区方案（MBR 或 GPT）进行初始化，定义 disklabel 分区，创建并挂载文件系统，并通过 `/etc/fstab` 完成引导时配置。同时涵盖可移动介质、文件系统检查以及磁盘监控工具的使用。

## 磁盘设备与分区命名

OpenBSD 对存储硬件采用清晰、结构化的设备命名约定。

### 整盘

磁盘按驱动类型与发现顺序命名：

| 设备 | 说明 |
| --- | --- |
| `sd0` | 第一块 SCSI 或 SATA 磁盘 |
| `sd1` | 第二块 SCSI 或 SATA 磁盘 |
| `wd0` | 第一块 IDE 磁盘（现代系统中已少见） |
| `vnd0` | 第一块虚拟磁盘（如内存盘） |

这些名称在 `fdisk(8)`、`gpt(8)`、`disklabel(8)` 及文件系统工具中通用。

### 分区

每块磁盘通过 `disklabel(8)` 划分为若干分区。OpenBSD 允许每块磁盘最多 16 个分区，编号为 `a` 到 `p`。

| 分区 | 典型用途 |
| --- | --- |
| `a` | 根文件系统或主挂载点 |
| `b` | 交换分区 |
| `c` | 整块磁盘（系统自动定义，禁止修改或挂载） |
| `d`–`h` | 常用于 `/usr`、`/home`、`/var` 等 |
| `i`–`p` | 用于额外文件系统或特殊用途 |

`c` 分区始终代表整块磁盘，是系统必需的，禁止格式化、挂载或手动修改。

创建与修改 disklabel 分区使用：

```sh
# disklabel -E sd0
```

分区定义完成后，可格式化、挂载、加密或用于 RAID 集。

### 设备节点

OpenBSD 同时使用块设备文件与字符设备文件访问分区：

| 设备节点 | 用途 |
| --- | --- |
| `/dev/sd0a` | 块设备（用于挂载） |
| `/dev/rsd0a` | 原始字符设备（用于 `newfs`、`fsck`） |

格式化与文件系统检查应使用原始设备（带 `r` 前缀），挂载则使用块设备。

## 磁盘检测

新存储设备由内核自动识别。查看已检测到的磁盘：

```sh
# dmesg | grep ^sd
```

上述命令列出 `sd0`、`sd1` 等设备。检测到的每块磁盘都可随后查看并初始化。

## 磁盘初始化

定义 disklabel 分区前，须先用 MBR 或 GPT 分区方案初始化磁盘。OpenBSD 在基于 BIOS 的系统上默认使用 MBR，在基于 UEFI 的系统上默认使用 GPT。

### 选择 MBR 或 GPT

分区方案取决于引导固件：

| 方案 | 平台 | 工具 |
| --- | --- | --- |
| MBR | BIOS（传统） | `fdisk(8)` |
| GPT | UEFI | `gpt(8)` |

不用于引导的纯数据盘可任选其一。

### 使用 `fdisk` 初始化 MBR

在 MBR 系统上，将磁盘初始化为单一 OpenBSD 分区（类型 `A6`）：

```sh
# fdisk -iy sd0
```

该命令覆写 MBR，创建一个标记为活动的 OpenBSD 分区。查看分区布局：

```sh
# fdisk sd0
```

必要时可手动设置活动（可引导）标志：

```sh
# fdisk -e sd0
fdisk: 1> flag 0
fdisk:*1> write
fdisk: 1> quit
```

某些系统需手动安装 MBR 引导程序：

```sh
# fdisk -b /usr/mdec/mbr sd0
```

不用于引导的磁盘无需配置 MBR。

### 使用 `fdisk` 初始化 GPT

基于 UEFI 的系统使用 GPT。OpenBSD 不提供 `gpt(8)`，GPT 分区表通过 `fdisk(8)` 创建与管理。

初始化 GPT 并创建 EFI 系统分区：

```sh
# fdisk -gy sd0
# fdisk -e sd0
fdisk: 1> setpid 1 EFI System
fdisk: 1> edit 1
Partition id ('0' to disable)  [0 - FF]: EF
Do you wish to edit in CHS mode? [n] n
Partition offset: [default] (press enter)
Partition size: [+512M]
fdisk: 1> write
fdisk: 1> quit
```

创建 EFI 系统分区后，在其上建立 FAT32 文件系统并安装引导加载程序：

```sh
# newfs_msdos /dev/rsd0i
# mount_msdos /dev/sd0i /mnt
# mkdir -p /mnt/efi/boot
# cp /usr/mdec/BOOTX64.EFI /mnt/efi/boot/
# umount /mnt
```

磁盘内部的 OpenBSD 分区随后通过 `disklabel(8)` 管理。

查看分区表：

```sh
# fdisk sd0
```

与 MBR 不同，GPT 无需将分区标记为活动。UEFI 固件依据分区类型自动定位 EFI 系统分区。

## 使用 `disklabel` 分区

外层 MBR 或 GPT 方案就绪后，使用 `disklabel(8)` 在其内部创建 OpenBSD 专用分区。

创建分区：

```sh
# disklabel -E sd1
sd1> a a
offset: [64]
size: [*]
FS type: [4.2BSD]
sd1*> w
sd1> q
```

上述操作创建分区 `a`，自标准偏移（64 扇区）起占用全部剩余空间，类型为 `4.2BSD`。

查看生成的 disklabel：

```sh
# disklabel sd1
```

自动创建的 `c` 分区始终覆盖整块磁盘，禁止修改。

## 创建与挂载文件系统

分区定义完成后，可格式化为文件系统并挂载。

### 使用 `newfs` 格式化

为分区初始化标准 FFS 文件系统：

```sh
# newfs /dev/rsd1a
```

必须使用原始字符设备（`rsd1a`）。

### 挂载文件系统

创建挂载点并挂载文件系统：

```sh
# mkdir -p /mnt/storage
# mount /dev/sd1a /mnt/storage
```

确认文件系统已挂载：

```sh
# df -h /mnt/storage
```

### 配置 `/etc/fstab`

要让文件系统在引导时自动挂载，在 `/etc/fstab` 中添加一行：

```
/dev/sd1a /mnt/storage ffs rw 1 2
```

`/etc/fstab` 每行的六个字段含义如下：

| 字段 | 用途 | 示例 |
| --- | --- | --- |
| 1 | 设备路径 | `/dev/sd1a` |
| 2 | 挂载点 | `/mnt/storage` |
| 3 | 文件系统类型 | `ffs` |
| 4 | 挂载选项 | `rw`、`noauto` 等 |
| 5 | dump 标志（遗留字段；通常为 0 或 1） | `1` |
| 6 | `fsck` 检查顺序 | `1`（根）、`2`（其他）、`0`（跳过） |

#### 常见文件系统类型

| 类型 | 说明 |
| --- | --- |
| `ffs` | OpenBSD 快速文件系统 |
| `cd9660` | ISO 9660 CD/DVD 介质 |
| `msdos` | FAT 格式的文件系统 |
| `ext2fs` | 只读访问 Linux EXT2/EXT3 卷 |
| `swap` | 交换分区 |
| `sw` | 交换分区的推荐关键字 |

#### 挂载选项

| 选项 | 说明 |
| --- | --- |
| `rw` | 读写 |
| `ro` | 只读 |
| `noexec` | 不允许执行二进制文件 |
| `nodev` | 忽略设备节点 |
| `nosuid` | 忽略 setuid/setgid 位 |
| `noauto` | 引导时不挂载 |
| `wxallowed` | 允许可写且可执行的映射（如 JIT） |
| `softdep` | 使用软更新（元数据日志；FFS 默认启用） |
| `userquota` | 启用用户配额 |
| `groupquota` | 启用组配额 |

辅助数据卷的 `/etc/fstab` 示例：

```
/dev/sd1a /mnt/data ffs rw,nodev,nosuid 1 2
```

### 文件系统一致性检查

若文件系统未正常卸载，引导时会用 `fsck(8)` 检查。手动检查：

```sh
# fsck /dev/sd1a
```

自动修复：

```sh
# fsck -y /dev/sd1a
```

正常卸载或干净关机可确保文件系统保持干净状态：

```sh
# umount /mnt/storage
# shutdown -p now
```

## 交换分区配置

交换分区通过将内存内容分页到磁盘来提供虚拟内存，供物理内存不足时使用。OpenBSD 使用专用的 disklabel 分区作为交换分区，按惯例编号为 `b`。

### 创建交换分区

安装过程中通常会自动创建 `b` 交换分区。必要时可使用 `disklabel(8)` 手动添加：

```sh
# disklabel -E sd1
sd1> a b
offset: [next available]
size: [e.g., 4G]
FS type: [swap]
sd1*> w
sd1> q
```

查看结果：

```sh
# disklabel sd1
```

应能看到类似如下的行：

```
  b:  8388608  1953600  swap
```

### 运行时启用交换分区

立即激活交换分区：

```sh
# swapctl -a /dev/sd1b
```

查看活动交换设备：

```sh
# swapctl -l
```

### 在 `/etc/fstab` 中配置持久化交换分区

要让交换分区在引导时自动启用，在 `/etc/fstab` 中添加一条记录。挂载点字段恒为 `none`，文件系统类型应为 `swap` 或推荐的同义词 `sw`。

```
/dev/sd1b none swap sw 0 0
```

OpenBSD 推荐使用 `sw` 关键字，但两种写法均受支持。

### 多个交换设备

若定义了多个交换分区，OpenBSD 会在它们之间分摊分页负载。运行时可通过多次调用 `swapctl -a` 添加更多交换设备。

### 移除交换分区

停用交换分区：

```sh
# swapctl -d /dev/sd1b
```

该命令仅将设备从活动交换列表中移除，不会修改 disklabel 或删除数据。

交换分区对物理内存有限的系统、大型构建或工作负载至关重要，也是启用系统休眠（若平台支持）所需。

## 挂载可移动介质

USB 存储设备、ISO 镜像、内存盘等可移动介质可像普通文件系统一样挂载使用，通常按需手动挂载。

### USB 存储设备

USB 大容量存储设备（如 U 盘）与其他磁盘一样显示为 `sdN`。插入 USB 设备时，内核会分配下一个可用设备号（例如 `sd2`）。

确定正确的设备与分区：

```sh
# dmesg | grep ^sd
# disklabel sd2
```

USB 设备常用 FAT 文件系统格式化。挂载 FAT 格式的 USB 设备：

```sh
# mkdir -p /mnt/usb
# mount -t msdos /dev/sd2i /mnt/usb
```

若不确定具体分区，查看 `disklabel sd2` 的输出，目标分区类型通常为 `MSDOS`。

安全移除设备：

```sh
# umount /mnt/usb
```

卸载前请确认没有 shell 或后台进程在访问挂载点。设备正在使用时，`umount` 会因 “device busy” 失败。

可移动磁盘不应列入 `/etc/fstab`，除非配置 `noauto` 选项，以免引导时被挂载。

### 挂载 ISO 镜像

ISO 文件（如 OpenBSD 安装镜像）可通过 `vnd(4)` 配置的虚拟磁盘挂载为只读文件系统。

将 ISO 镜像关联到虚拟磁盘：

```sh
# vnconfig vnd0 install78.iso
# mount -t cd9660 /dev/vnd0c /mnt/iso
```

文件系统类型 `cd9660` 对应 CD-ROM 介质使用的 ISO 9660 标准。`vnd0` 的 `c` 分区始终指向完整镜像。

卸载并解除关联：

```sh
# umount /mnt/iso
# vnconfig -u vnd0
```

### 创建与使用内存盘

OpenBSD 通过 `vnd(4)` 驱动支持基于内存的虚拟磁盘，适用于临时文件系统、测试或恢复环境。其后端可以是普通文件，也可以是匿名系统内存。

#### 使用后端文件

创建 64 MB 镜像文件：

```sh
# dd if=/dev/zero of=/tmp/ramdisk.img bs=1m count=64
# vnconfig vnd0 /tmp/ramdisk.img
# disklabel -E vnd0
vnd0> a a
offset: [64]
size: [*]
FS type: [4.2BSD]
vnd0*> w
vnd0> q
# newfs /dev/rvnd0a
# mount /dev/vnd0a /mnt/ram
```

使用完毕后卸载并解除关联：

```sh
# umount /mnt/ram
# vnconfig -u vnd0
```

#### 使用匿名内存

创建无后端文件的纯内存盘：

```sh
# vnconfig -s labels -c vnd0
# disklabel -E vnd0
# newfs /dev/rvnd0a
# mount /dev/vnd0a /mnt/tmpfs
```

上述操作创建一个完全驻留在 RAM 中的临时文件系统。设备卸载或系统重启后，其内容将丢失。

`vnd(4)` 驱动支持多个实例（`vnd0`、`vnd1` 等），ISO 镜像与内存盘可同时挂载。

### 创建 ISO 镜像文件

OpenBSD 支持使用镜像文件进行归档、传输与安全存储。镜像文件可以是普通 ISO 9660 镜像，也可以是基于 `softraid(4)` 的加密容器文件。镜像文件通常通过 `vnconfig(8)` 关联，并像普通文件系统一样挂载。

#### 创建普通 ISO 镜像

从目录生成标准 ISO 9660 镜像：

```sh
# mkhybrid -o archive.iso -R -J /path/to/data
```

该命令生成 `archive.iso`，即包含 `/path/to/data` 内容的只读 ISO 镜像。

挂载该镜像：

```sh
# vnconfig vnd0 archive.iso
# mount -t cd9660 /dev/vnd0c /mnt
```

卸载并解除关联：

```sh
# umount /mnt
# vnconfig -u vnd0
```

任何具备该镜像文件与 `vnd(4)` 支持的系统都可重复上述操作。

#### 创建加密镜像文件

为敏感数据创建基于文件的加密卷：

1. **创建后端文件并关联到虚拟磁盘：**

   ```sh
   # dd if=/dev/zero of=secure.img bs=1m count=128
   # vnconfig vnd0 secure.img
   ```
2. **使用 CRYPTO discipline 创建 `softraid` 卷：**

   ```sh
   # bioctl -c C -l vnd0 softraid0
   New passphrase:
   Re-type passphrase:
   softraid0: CRYPTO volume attached as sd3
   ```
3. **为加密文件系统建立 disklabel、格式化并挂载：**

   ```sh
   # disklabel -E sd3
   sd3> a a
   offset: [64]
   size: [*]
   FS type: [4.2BSD]
   sd3*> w
   sd3> q
   # newfs /dev/rsd3a
   # mkdir -p /mnt/secure
   # mount /dev/sd3a /mnt/secure
   ```

后续使用该加密卷：

1. 关联镜像并解锁：

   ```sh
   # vnconfig vnd0 secure.img
   # bioctl -c C -l vnd0 softraid0
   Passphrase:
   softraid0: CRYPTO volume attached as sd3
   ```
2. 挂载文件系统：

   ```sh
   # mount /dev/sd3a /mnt/secure
   ```
3. 使用完毕后卸载并解除关联：

   ```sh
   # umount /mnt/secure
   # bioctl -d sd3
   # vnconfig -u vnd0
   ```

这种方式可创建安全、可移植的加密容器，行为类似可移动存储。加密镜像与系统相关，需要 OpenBSD `softraid(4)` 驱动才能访问。

## 高级特性

OpenBSD 内置若干存储管理高级特性，包括文件系统快照、配额、加密卷与软件 RAID。这些特性在标准文件系统之上提供更强的控制力、灵活性与数据完整性。

### 使用 `fssnap` 创建快照

文件系统快照是已挂载文件系统在某一时刻的只读映像。快照可在不停机的情况下进行一致性备份或取证检查。

OpenBSD 通过 `fssnap(8)` 为 FFS2（快速文件系统版本 2）提供快照支持。

为 `/home` 创建快照：

```sh
# fssnap -c /home/.snap0 /home
```

快照必须位于同一文件系统上，呈现为只读文件。挂载快照：

```sh
# mount -o ro /home/.snap0 /mnt/snap
```

此后即可通过 `/mnt/snap` 检视或备份。删除快照：

```sh
# umount /mnt/snap
# fssnap -d /home/.snap0
```

快照在显式删除前会跨重启保留。可同时创建多个快照。

### 磁盘配额

配额允许管理员按用户或组限制各文件系统的磁盘用量。仅 `ffs` 类型文件系统支持配额。

启用配额步骤：

1. 修改 `/etc/fstab`，在选项字段中加入 `userquota` 或 `groupquota`：

   ```
   /dev/sd1a /home ffs rw,userquota 1 2
   ```
2. 在文件系统根目录下创建必要的配额文件：

   ```sh
   # touch /home/quota.user
   # chmod 600 /home/quota.user
   # edquota -u alice
   ```
3. 在文件系统上启用配额：

   ```sh
   # quotaon /home
   ```

查看当前用量：

```sh
# quota -u alice
# repquota /home
```

配额一经设置立即生效，只要在 `/etc/fstab` 中定义，重启后仍保持启用。

### 使用 `softraid` 创建加密卷

OpenBSD 通过 `softraid(4)` 驱动配合 `CRYPTO` discipline 支持全盘加密。该方式在底层分区之上建立加密卷。

用未使用的分区（如 `sd1a`）创建加密卷：

```sh
# bioctl -c C -l sd1a softraid0
New passphrase:
Re-type passphrase:
softraid0: CRYPTO volume attached as sd3
```

新卷以设备形式关联（如 `sd3`）。格式化并挂载该卷：

```sh
# newfs /dev/rsd3c
# mkdir -p /secure
# mount /dev/sd3c /secure
```

重启时，系统会在关联该卷前提示输入加密口令。如有需要，可使用 keydisk 配置加密卷的自动解锁。

查看已关联卷及其状态：

```sh
# bioctl -v softraid0
```

### 使用 `softraid` 实现软件 RAID

`softraid(4)` 子系统支持使用标准磁盘组建 RAID。常见用法包括用于冗余的镜像（RAID 1）和用于性能的条带（RAID 0）。

用两个分区（`sd1a` 与 `sd2a`）创建镜像 RAID 1 卷：

```sh
# bioctl -c 1 -l sd1a,sd2a softraid0
softraid0: RAID 1 volume attached as sd3
# newfs /dev/rsd3c
# mount /dev/sd3c /mnt/raid
```

新设备 `sd3` 与其他磁盘无异，可正常挂载、检查与使用。

查看状态：

```sh
# bioctl -v softraid0
```

系统会报告该卷是健康、重建中还是降级。

OpenBSD 支持的 RAID 级别：

| RAID | 名称 | 是否支持 | 说明 |
| --- | --- | --- | --- |
| 0 | 条带 | 是 | 高性能，无冗余 |
| 1 | 镜像 | 是 | 冗余，适合容错 |
| 5 | 条带+校验 | 是 | 兼顾冗余与容量 |
| 10 | RAID 1+0 | 手动 | 先镜像后条带；需手动管理 |
| C | 加密 | 是 | 加密卷，严格来说不属于 RAID 级别 |

OpenBSD 基本系统不支持 RAID 6、50、60。

使用软件 RAID 时，应确保所有成员盘在容量与对齐方式上配置一致。务必事先测试恢复与重建流程。从 softraid 卷引导受支持，但可能需要平台相关的配置。

## 监控与健康状态

OpenBSD 基本系统不原生支持 S.M.A.R.T.（自监测、分析与报告技术）。但可通过 `smartmontools` 软件包获得有限的 S.M.A.R.T. 功能，其中提供 `smartctl(8)` 工具与 `smartd(8)` 监控守护进程。

该功能可用于监控磁盘健康状态、预测受支持 ATA 与 SATA 磁盘的潜在故障。对 NVMe 与 USB 设备的支持有限，可能取决于控制器。

### 安装 `smartmontools`

使用 `pkg_add` 安装该软件包：

```sh
# pkg_add smartmontools
```

安装后即可使用 `smartctl` 与 `smartd` 工具。

### 使用 `smartctl` 检查磁盘健康

查询受支持的磁盘：

```sh
# smartctl -i /dev/sd0c
```

该命令显示设备身份信息，并指明是否启用 S.M.A.R.T. 支持。查看所有健康与错误属性：

```sh
# smartctl -a /dev/sd0c
```

设备名应对应整块磁盘（通常是 `c` 分区）。并非所有控制器都会暴露 S.M.A.R.T. 数据，部分设备可能返回有限信息或无结果。

### 使用 `smartd` 启用后台监控

`smartd` 守护进程周期性检查一个或多个设备，并在阈值被超越时发出警告。

#### 第 1 步：配置 `/etc/smartd.conf`

每行定义一个设备及其选项。示例：

```
/dev/sd0c -a -m root -M exec /usr/local/libexec/smartmontools/smartd-runner
```

上述配置以全部默认检查（`-a`）监控 `/dev/sd0c`，并通过执行默认处理脚本向 `root` 用户发送告警。

要监控多块磁盘，添加更多行。

#### 第 2 步：启用并启动守护进程

启用开机自启：

```conf
smartd_flags=""
```

随后启动守护进程：

```sh
# rcctl enable smartd
# rcctl start smartd
```

#### 第 3 步：模拟与测试

手动模拟检查或触发测试：

```sh
# smartctl -t short /dev/sd0c
```

日志会写入 `/var/log/daemon`。告警通过本地邮件发送，请确保系统邮件投递功能正常，且 `/etc/mail/aliases` 中为 `root` 配置了有效条目。

### 局限性

OpenBSD 上的 S.M.A.R.T. 功能取决于硬件兼容性与驱动支持。USB 外接磁盘与 NVMe 设备通常不受支持或仅部分功能可用，对其监控可能不可靠。

`smartmontools` 软件包适用于使用直连 SATA 磁盘、并希望获得故障预测报告的系统。
