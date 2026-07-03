# 图形环境

## 概述

本章介绍如何在 OpenBSD 上配置和使用图形环境，涵盖轻量级窗口管理器、完整桌面环境、图形登录管理器，以及支持完整桌面体验所需的常用工具。

X Window System 的基本配置（包括输入设备与图形硬件）在 [X11](../install-and-configure/x11.md) 一章中介绍，本章假定相关配置已经完成。

## 窗口管理器

**窗口管理器（WM）**负责控制 X Window System 中应用程序窗口的位置、外观与行为。

窗口管理器通常分为以下几类：

- **堆叠式（浮动式）**：窗口可以自由重叠，通常用鼠标移动和调整大小。
- **平铺式**：窗口以不重叠的方式并排放置，通常用键盘快捷键管理。
- **动态式**：提供平铺、浮动和标签三种模式，可动态切换。

### OpenBSD 上的常见窗口管理器

下表概述了通过软件包安装的常见窗口管理器：

| 窗口管理器 | 类型 | 软件包 | 备注 |
| --- | --- | --- | --- |
| `cwm` | 堆叠式 | 基本系统 | 极简、键盘驱动，OpenBSD 默认 |
| `fvwm` | 堆叠式 | `fvwm` | 经典，可配置性高 |
| `i3` | 平铺式 | `i3` | 流行的平铺式窗口管理器，维护活跃 |
| `openbox` | 堆叠式 | `openbox` | 轻量且支持脚本扩展 |
| `fluxbox` | 堆叠式 | `fluxbox` | 快速简洁，Blackbox 的衍生分支 |
| `icewm` | 堆叠式 | `icewm` | 传统外观，带任务栏 |
| `jwm` | 堆叠式 | `jwm` | 体积小，资源占用低 |
| `spectrwm` | 平铺式 | `spectrwm` | 紧凑的平铺式窗口管理器，对 OpenBSD 支持良好 |
| `dwm` | 平铺式 | `dwm` | 极简，需重新编译 |
| `awesome` | 动态式 | `awesome` | 可用 Lua 配置的平铺/动态窗口管理器 |

用 `pkg_add` 安装上述任一窗口管理器，并通过 **.xinitrc** 或 **.xsession** 配置启动。

以 `i3` 为例：

```sh
# pkg_add i3
$ echo 'exec i3' > ~/.xinitrc
$ startx
```

## 桌面环境

桌面环境通常包含窗口管理器、图形工具、文件管理器、终端模拟器和配置面板，提供更完整的体验，但需要更多系统资源。

### Xfce

Xfce 资源占用低，在 OpenBSD 上广受欢迎。

```sh
# pkg_add xfce
$ echo 'exec startxfce4' > ~/.xinitrc
```

### GNOME

GNOME 是一款现代、功能完整的桌面环境。

```sh
# pkg_add gnome
$ echo 'exec gnome-session' > ~/.xinitrc
```

需要启用 `messagebus` 和 `polkit` 等辅助服务。

### KDE Plasma

KDE 提供精致、可配置性高的桌面环境。

```sh
# pkg_add kde
$ echo 'exec startplasma-x11' > ~/.xinitrc
```

KDE 采用模块化设计，需要更多软件包和内存。

## 显示管理器（登录管理器）

**显示管理器**（又称**登录管理器**）是一种图形应用程序，用于启动 X 并显示用户登录界面，无需再手动执行 `startx`。

显示管理器通常使用 **.xsession** 而非 **.xinitrc**。

### xenodm

`xenodm` 包含在基本系统中，提供极简的图形登录界面。

开机时启用：

```sh
# rcctl enable xenodm
# rcctl start xenodm
```

相关自定义配置位于 **/etc/X11/xenodm/**。

### 其他显示管理器

以下显示管理器可通过软件包安装：

- **GDM**（GNOME）：`pkg_add gdm`
- **SDDM**（KDE）：`pkg_add sddm`
- **LightDM**（轻量级，基于 GTK）：`pkg_add lightdm`

启用方法如下：

```sh
# rcctl enable gdm   # 或者 sddm、lightdm
# rcctl start gdm
```

为正确启动会话，确保用户拥有合适的 **.xsession**：

```sh
$ echo 'exec startxfce4' > ~/.xsession
```

显示管理器通常会自动处理会话选择。

## 会话管理

使用 `startx` 启动 X 时用 **.xinitrc**；使用 `xenodm`、`gdm` 或 `lightdm` 等显示管理器时用 **.xsession**。

为共享配置，可让 **.xinitrc** 引入 **.xsession**：

```sh
$ echo 'exec ~/.xsession' > ~/.xinitrc
```

## 多显示器设置

用 `xrandr` 工具可动态配置屏幕布局。

示例：

```sh
$ xrandr --output eDP-1 --primary --output HDMI-1 --right-of eDP-1 --auto
```

将此类命令加入 **.xinitrc** 或 **.xsession**，登录时即可应用设置。

## 字体与外观

桌面环境与应用程序所用字体可按以下方式安装：

```sh
# pkg_add fontconfig dejavu-fonts noto-fonts ttf-bitstream-vera
```

字体渲染设置可通过 **~/.Xresources** 或 **~/.config/fontconfig/fonts.conf** 调整。

更多信息见 [X11](../install-and-configure/x11.md) 一章。

## 故障排查

- 若 `startx` 失败，检查用户主目录下的 **.xsession-errors**。
- 确认会话命令（如 `startxfce4`）已正确安装。
- 确保所需守护进程正在运行：

```sh
# rcctl enable messagebus
# rcctl start messagebus
# rcctl enable polkit
# rcctl start polkit
```

## 网页浏览器

OpenBSD 支持多种网页浏览器，既有功能完整的图形浏览器，也有极简的文本浏览器。这些浏览器均通过 ports 系统提供，可用 `pkg_add` 安装。

### 资源限制与浏览器稳定性

OpenBSD 默认对每个进程施加严格的内存限制。数据段大小的默认软限制（`datasize-cur`）为每进程 **1536 MB（1.5 GB）**。Firefox 和 Chromium 等现代浏览器在复杂页面或多标签场景下可能超出此阈值，导致崩溃或意外退出。

为登录类或用户提高限制：

1. 编辑 **/etc/login.conf**：

   ```conf
   default:\
   	:datasize-cur=4096M:\
   	:datasize-max=4096M:\
   	:\
   	...
   ```

2. 重建登录数据库：

   ```sh
   # cap_mkdb /etc/login.conf
   ```

3. 注销并重新登录后更改生效。

> 提高内存限制前，请确保系统有足够的内存与交换空间。

### 功能完整的浏览器

推荐大多数用户使用此类浏览器，它们支持 JavaScript、CSS、多媒体播放和扩展等现代 Web 标准。

#### Firefox

Firefox 是 OpenBSD 默认的图形浏览器，维护良好。

```sh
# pkg_add firefox
```

衍生版本包括：

- `firefox-esr`：延长支持版
- `firefox-i18n`：添加多语言包

确保启用以下辅助服务：

```sh
# rcctl enable messagebus
# rcctl start messagebus
# rcctl enable avahi_daemon
# rcctl start avahi_daemon
```

#### Chromium

Chromium 是一款开源浏览器，与 Google Chrome 同源。

```sh
# pkg_add chromium
```

> 由于沙箱或库依赖关系，Chromium 并非在所有硬件平台上都可用。可用 `pkg_info -Q chromium` 查询支持的架构。

### 其他图形浏览器

#### qutebrowser

基于 Qt 的极简浏览器，采用 Vim 风格按键绑定。

```sh
# pkg_add qutebrowser
```

需要 `python`、`qt5` 等支持包。

#### epiphany

GNOME 默认的 WebKitGTK 浏览器。

```sh
# pkg_add epiphany
```

与 GNOME 会话集成良好，支持书签和标签页。

#### surf

suckless.org 开发的极简 WebKit 浏览器。

```sh
# pkg_add surf
```

默认无标签页，可配合 `tabbed` 实现多窗口。

### 轻量级与文本浏览器

此类浏览器资源占用少，适合通过 SSH 使用或在低内存环境中运行。

#### netsurf

轻量级图形浏览器，自带渲染引擎。

```sh
# pkg_add netsurf
```

不支持 JavaScript 和复杂的 HTML5 特性。

#### lynx, w3m, links

文本模式浏览器，适用于终端或无图形环境。

```sh
# pkg_add lynx w3m links
```

适合阅读文档、访问网页邮件或浏览轻量网站。

### 区域设置与字体配置

浏览器需要正确的区域设置和字体配置才能正常显示多语言文本和抗锯齿效果。

按需设置区域：

```sh
$ export LANG=en_US.UTF-8
$ firefox
```

安装字体包：

```sh
# pkg_add fontconfig dejavu-fonts noto-fonts ttf-bitstream-vera
```

更多配置选项见 [X11](../install-and-configure/x11.md) 一章。

### 多媒体支持

部分浏览器依赖外部工具播放音视频，按需安装以下软件：

```sh
# pkg_add ffmpeg gst-plugins-good gst-plugins-bad
```

在浏览器首选项中启用或禁用硬件加速。

### 故障排查

- 浏览器突然退出可能是超出了内存限制（`datasize-cur`）。
- 字体或符号缺失：检查字体包与 `fontconfig` 配置。
- 视频播放失败：检查 `ffmpeg`、`gstreamer` 或 `sndio` 是否有问题。

## 开发工具

OpenBSD 自带许多适用于软件开发的工具，从编译器、调试器到图形编辑器和 IDE。多数工具通过软件包安装，部分已包含在基本系统中。

### 编译器与构建工具

OpenBSD 基本系统内置来自 LLVM 的 `clang` 和 `lld` 工具链，支持 C、C++ 和 Objective-C。

其他构建工具可通过软件包安装：

```sh
# pkg_add gmake cmake autoconf automake libtool pkgconf
```

从源码编译软件时常需这些工具。

Rust 或 Go 开发：

```sh
# pkg_add rust go
```

Java 开发：

```sh
# pkg_add openjdk
```

### 版本控制系统

用于处理源码仓库：

```sh
# pkg_add git cvs mercurial subversion
```

- `git`：最为常用
- `cvs`：OpenBSD 源码树所需
- `subversion`、`mercurial`：较少使用，但可用

### 编辑器与 IDE

可用的图形编辑器与 IDE 如下：

| 工具 | 软件包 | 备注 |
| --- | --- | --- |
| `gvim` | `vim-gtk2` | 图形版 Vim，基于 GTK2 |
| `emacs` | `emacs` | 完整的图形 Emacs，带 GUI |
| `kate` | `kate` | KDE 高级文本编辑器 |
| `geany` | `geany` | 轻量级 GTK IDE |
| `lite-xl` | `lite-xl` | 极简编辑器，使用 Lua 配置 |
| `code` | `vscode` | 软件包中无，需手动安装 |
| `neovim` | `neovim` | 终端版，但存在图形前端 |

安装示例：

```sh
# pkg_add geany
```

`geany` 可从终端或桌面菜单启动。

### 终端模拟器

终端模拟器虽不属于严格意义上的开发工具，但开发过程中常用。除基本系统自带的 `xterm` 外，还可选用：

- `st`（来自 suckless）：`pkg_add st`
- `rxvt-unicode`：`pkg_add rxvt-unicode`
- `alacritty`：`pkg_add alacritty`
- `kitty`：`pkg_add kitty`
- `tilix`：`pkg_add tilix`（基于 GTK）

选择能与所用桌面环境良好集成、并支持所需特性（如真彩色、Unicode、剪贴板）的终端模拟器。

### 调试工具

OpenBSD 基本系统默认包含 `lldb` 调试器。其他工具：

```sh
# pkg_add valgrind gdb strace
```

注意：`valgrind` 的支持因架构而异，功能有限。

图形化调试可用 `ddd`：

```sh
# pkg_add ddd
```

### 文档与手册页

多数 OpenBSD 开发工具的文档见手册页。

查找手册页：

```sh
$ man clang
$ man git
```

其他文档可能安装在 **/usr/local/share/doc/** 下，或通过项目网站在线查阅。

### 语言服务器与自动补全

`neovim`、`emacs`、`lite-xl` 等现代编辑器可与语言服务器协议（LSP）后端集成，实现智能代码补全、代码检查和重构。

安装方法：

```sh
# pkg_add py-pylsp node clang-tools
```

- `py-pylsp`：Python LSP
- `clangd`：C/C++ 语言服务器
- `typescript-language-server`：用于 JavaScript/TypeScript（需要 `node`）

配置方式因编辑器而异。

## 办公应用

OpenBSD 提供多种办公工具，从完整的办公套件到用于编辑文档、管理表格、制作演示文稿的轻量级应用一应俱全，可用 `pkg_add` 安装。

### LibreOffice

LibreOffice 是 OpenBSD 上功能最完整的办公套件，包括：

- **Writer**：文字处理器
- **Calc**：表格编辑器
- **Impress**：演示文稿制作
- **Draw**：矢量图形
- **Math**：公式编辑器
- **Base**：数据库前端

安装：

```sh
# pkg_add libreoffice
```

启动：

```sh
$ libreoffice
```

LibreOffice 支持 ODF 和 Microsoft Office 文件格式（.docx、.xlsx、.pptx），可与多数桌面环境集成。

### Calligra 套件

KDE 办公套件同样可用：

```sh
# pkg_add calligra
```

包含 `words`、`sheets`、`stage` 等组件，采用基于 Qt 的界面，与 KDE 环境集成良好。

### AbiWord 与 Gnumeric

资源有限的系统可选用独立应用：

```sh
# pkg_add abiword gnumeric
```

- `abiword`：轻量级文字处理器
- `gnumeric`：轻量级表格编辑器

两者均基于 GTK，内存占用比 LibreOffice 小。

### PDF 编辑

多数办公套件可导出 PDF，但编辑 PDF 文件需要专门工具。

- `pdfarranger`：用于合并/拆分页面的图形工具 — `pkg_add pdfarranger`
- `pdfsam`：基于 Java 的 PDF 编辑器 — `pkg_add pdfsam`
- `xournalpp`：在 PDF 上批注与绘图 — `pkg_add xournalpp`

PDF 查看见"文档查看器"一节。

### Markdown 与纯文本格式

偏好轻量工作流的用户可使用纯文本或 Markdown 写作工具：

- `ghostwriter`：极简 Markdown 编辑器 — `pkg_add ghostwriter`
- 用 `zathura` 或 `lowriter` 预览输出
- 用 Pandoc 转换格式：

  ```sh
  # pkg_add pandoc
  ```

示例：

```sh
$ pandoc -o report.pdf report.md
```

## 文档查看器

OpenBSD 支持多种文档查看应用，从极简 PDF 阅读器到功能丰富的多格式工具。这些应用通过 ports 系统提供，可用 `pkg_add` 安装。

### PDF 查看器

#### zathura

极简、键盘驱动的 PDF 查看器，支持插件。

```sh
# pkg_add zathura zathura-pdf-poppler
```

- Vim 风格按键绑定
- 基于插件的架构
- 支持书签与搜索

打开文件：

```sh
$ zathura document.pdf
```

#### mupdf

超轻量级 PDF 查看器。

```sh
# pkg_add mupdf
```

- 快速、节省资源
- 极简 GUI
- 键盘导航

#### qpdfview

带标签页的 PDF 查看器，可选支持 DjVu 和 PostScript。

```sh
# pkg_add qpdfview
```

- 支持批注与搜索
- 适合图形桌面使用

#### xpdf

传统、极简的 PDF 查看器，使用 Motif 工具包。

```sh
# pkg_add xpdf
```

简单、可移植，但缺乏现代特性。

### PostScript 与 DjVu

#### gv

PostScript 文件的 Ghostview 前端。

```sh
# pkg_add gv
```

依赖 Ghostscript（`pkg_add ghostscript`）进行渲染。

#### djview

DjVu 文件（扫描文档格式）的图形查看器。

```sh
# pkg_add djview
```

或者用 `zathura-djvu` 通过插件提供 DjVu 支持。

### 图像查看器

图像查看器虽不属于严格意义上的文档查看器，但常用于查看扫描件或文档图像。

- `feh`：轻量、支持脚本 — `pkg_add feh`
- `sxiv`：简单的 X 图像查看器 — `pkg_add sxiv`
- `nomacs`：功能更丰富，基于 Qt — `pkg_add nomacs`

### EPUB 与电子书查看器

#### foliate

基于 GTK 的现代电子书阅读器，支持 EPUB。

```sh
# pkg_add foliate
```

功能包括：

- 书签
- 高亮
- 文本转语音（需配置）

#### fbreader

轻量级电子书阅读器，支持 EPUB、MOBI 等格式。

```sh
# pkg_add fbreader
```

### 终端中的文档查看

- 用 `less` 查看纯文本文件
- 用 `w3m` 和 `lynx` 查看 HTML
- 用 `mupdf-gl` 在帧缓冲中渲染 PDF

示例：

```sh
$ less README.md
$ mupdf-gl presentation.pdf
```

## 财务

OpenBSD 提供多种财务应用，适用于个人预算、小型企业会计和投资跟踪。这些工具通过软件包提供，可与多种桌面环境集成。

### GnuCash

GnuCash 是一款功能完整的会计软件，适用于个人和小型企业。

```sh
# pkg_add gnucash
```

功能：

- 复式记账
- 定时交易
- 支持 OFX、QIF 和 CSV 导入
- 图形报表与图表

GnuCash 基于 GTK，与 Xfce 或 GNOME 集成良好，支持加密备份和多币种。

### HomeBank

HomeBank 是一款个人财务管理软件，界面更简洁。

```sh
# pkg_add homebank
```

功能：

- 预算与趋势分析
- 支出分类
- 支持 QIF 和 CSV 导入
- 收支的图形化展示

适合基本会计需求的用户。

### KMyMoney

KDE 的个人财务管理软件。

```sh
# pkg_add kmymoney
```

功能：

- 熟悉的分类账界面
- 支持银行与投资账户
- 对账与定时交易
- OFX/QIF 导入

最适合 KDE Plasma，但也能在其他环境中使用。

### beancount 与 hledger

偏好纯文本记账和版本控制财务数据的用户，OpenBSD 支持：

```sh
# pkg_add beancount hledger
```

- `beancount`：基于 Python 的复式记账工具，使用 `.beancount` 文件
- `hledger`：基于 Haskell 的命令行工具，可选 TUI/HTTP 接口

示例：

```
2025-07-11 * "Coffee Shop"
  expenses:food:coffee    $3.50
  assets:cash
```

这些工具注重简洁性、可审计性，并能与编辑器和脚本集成。

### 基于表格的记账

轻量记账可使用 `gnumeric` 或 `libreoffice-calc` 等表格软件管理个人账本或小型企业财务。

```sh
# pkg_add gnumeric
```

模板可在线获取，或自行创建。
