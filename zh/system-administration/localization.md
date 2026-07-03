# 本地化

## 概要

本地化指配置系统行为以支持区域语言、字符编码、日期格式与输入设置。OpenBSD 默认使用美式英语与 POSIX 标准，但可通过环境变量、UTF-8 字符集、备选键位映射与时区设置实现本地化。

本章说明如何：

- 通过 `LANG` 与 `LC_*` 变量设置语言与区域偏好
- 在基本系统中启用 UTF-8 字符编码
- 使用 `tzsetup(8)` 配置时区
- （可选）使用 `wsconsctl(8)` 更改控制台键盘布局

本地化适用于编辑其他语言文本、处理 Unicode 文件或为国际化用途准备系统。这些设置作用于**基本系统**（控制台、shell 与文本工具），不作用于 X11 等图形环境，后者需另行配置。

## 配置区域与 UTF-8 支持

OpenBSD 默认使用 `C` 区域（即 POSIX 区域），输出美式英语信息并采用 7 位 ASCII 编码。要处理其他语言的文件名、文本文件或终端输入，可通过环境变量启用 UTF-8 支持并设置区域相关行为。

### 区域环境变量

下表列出了控制区域设置的环境变量：

| 变量 | 用途 |
| --- | --- |
| `LANG` | 所有类别使用的默认区域 |
| `LC_CTYPE` | 字符分类与编码 |
| `LC_COLLATE` | 字符串比较与排序顺序 |
| `LC_MESSAGES` | 程序信息的语言 |
| `LC_TIME` | 日期与时间格式 |
| `LC_NUMERIC` | 数字格式（如小数点） |
| `LC_ALL` | 覆盖上述所有变量（不建议长期使用） |

查看当前会话生效的区域设置：

```sh
$ locale
```

该命令显示 `LANG`、`LC_CTYPE` 等变量的当前值。

例如启用德语信息输出与 UTF-8 编码：

```sh
$ export LANG=de_DE.UTF-8
```

要对所有用户全局生效，可将变量加入 `/etc/profile`，或在 `/etc/login.conf` 中配置。

### 可用区域

OpenBSD 在 `/usr/share/locale/` 下提供了一组已编译的区域设置，安装时生成，包含 UTF-8 变体。

列出所有可用区域：

```sh
$ locale -a
```

筛选支持 UTF-8 的区域：

```sh
$ locale -a | grep UTF-8
```

示例如下：

- `en_US.UTF-8` — 美式英语（UTF-8）
- `de_DE.UTF-8` — 德语
- `fr_FR.UTF-8` — 法语
- `ja_JP.UTF-8` — 日语

并非所有系统信息都有对应语言的翻译。若所需区域不可用，回退到 `en_US.UTF-8` 即可在保留 UTF-8 支持的同时使用英语信息。

### 在控制台中使用 UTF-8

在终端会话中启用 UTF-8 行为：

1. 将 `LC_CTYPE` 或 `LANG` 设为 UTF-8 区域：

   ```sh
   $ export LC_CTYPE=en_US.UTF-8
   ```
2. 确保终端支持 UTF-8 编码。多数控制台终端（`tmux`、`xterm`、`ssh`）默认支持 UTF-8。
3. 必要时使用 `wsfontload(8)` 加载兼容 Unicode 的字体（参见键盘与显示配置）。

区域配置正确的前提下，`less`、`awk`、`grep` 等标准工具即可正确处理 UTF-8 输入与输出。

### 持久化区域配置

要在登录时设置区域变量，可修改用户的 shell 启动文件，或在 `/etc/login.conf` 中定义。

#### 示例：`.profile`（适用于 `sh`、`ksh` 等）

```sh
export LANG=en_US.UTF-8
export LC_TIME=de_DE.UTF-8
```

#### 示例：`/etc/login.conf`

找到合适的登录类（如 `default`），设置：

```conf
:lang=en_US.UTF-8:
```

随后重建能力数据库：

```sh
# cap_mkdb /etc/login.conf
```

`login.conf` 中的区域设置会自动应用于该登录类下的用户。`.profile` 或 `.login` 中的会话级覆盖优先级更高。

## 时区配置

OpenBSD 将系统时区存储为 `/etc/localtime` 到 `zoneinfo` 数据库中某文件的符号链接，该数据库位于 `/usr/share/zoneinfo/`。默认使用 UTC，除非安装时选择了本地时区。

更改时区会影响系统显示时间、记录事件与格式化时间戳的方式，不影响硬件时钟，硬件时钟应保持为 UTC。

### 使用 `tzsetup` 设置时区

以交互方式配置或更改系统时区：

```sh
# tzsetup
```

该命令会显示大洲与城市菜单，选择后即相应设置 `/etc/localtime`。

示例：

1. 选择 `Europe`
2. 选择 `Amsterdam`

生成的符号链接如下：

```sh
# ls -l /etc/localtime
lrwxr-xr-x  1 root  wheel  33 Jul 29 14:21 /etc/localtime -> /usr/share/zoneinfo/Europe/Amsterdam
```

### 手动设置时区

也可不使用 `tzsetup` 直接设置时区：

```sh
# ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
```

新时区必须对所有用户与进程可读，方可立即生效，无需重启。

### 查看当前时区

显示系统当前时间与时区设置：

```sh
$ date
```

校验符号链接：

```sh
$ readlink /etc/localtime
/usr/share/zoneinfo/Europe/Berlin
```

### UTC 与 RTC 处理

系统时钟应始终使用 UTC。OpenBSD 默认按 UTC 处理，以避免夏令时切换与多系统共存环境下的冲突。

确认当前时间以 UTC 跟踪：

```sh
# sysctl kern.utc_offset
```

硬件时钟由 `ntpd(8)` 管理，也可用 `date(1)` 与 `hwclock(8)` 手动更新。

时区配置会影响系统日志时间戳（`syslog`、`dmesg`）、`date` 输出，以及任何调用 C 库 `localtime(3)` 函数的应用。

## 控制台键盘布局

OpenBSD 系统控制台默认使用美式英语 QWERTY 键盘布局。需要其他布局（如德语 QWERTZ、法语 AZERTY）的用户可使用 `wsconsctl(8)` 重新配置控制台键盘。

这些设置仅作用于文本控制台（如 `ttyC0` 至 `ttyC4`），不影响 X11 等图形环境。

### 列出可用键位映射

可用键位映射位于 `/usr/share/wscons/keymaps/`。列出这些文件：

```sh
$ ls /usr/share/wscons/keymaps/
```

示例如下：

- `de.nodead` — 不带死键的德语布局
- `fr` — 法语 AZERTY 布局
- `uk` — 英式 QWERTY 布局
- `us.colemak` — 美式 Colemak 布局

`wsconsctl` 使用的布局名须与这些文件名一致（不含 `.wscons` 扩展名）。

### 临时设置键位映射

为当前会话立即应用某键位映射：

```sh
# wsconsctl keyboard.encoding=de.nodead
```

设置立即生效，重启后失效。确认设置：

```sh
$ wsconsctl keyboard.encoding
keyboard.encoding=de.nodead
```

### 持久化设置

要让键盘布局在每次引导时生效，编辑 `/etc/wsconsctl.conf`：

```conf
keyboard.encoding=de.nodead
```

该配置文件在系统启动时由 `/etc/rc.d/` 下定义的 `wsconsctl` 服务处理。

### 恢复默认布局

恢复默认的美式英语布局：

```sh
# wsconsctl keyboard.encoding=us
```

更改键盘布局对使用非美式物理键盘或在多语言环境中工作的用户很有用。该设置仅影响控制台会话，图形环境使用各自的输入系统单独管理键盘设置。
