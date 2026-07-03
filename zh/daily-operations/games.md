# 游戏

## 概述

本章精选介绍 OpenBSD 上可用的游戏，分为终端游戏与图形游戏两类。OpenBSD 并不以游戏性能为优先，但通过 ports 与软件包系统支持多种游戏，包括策略、模拟、Roguelike 以及模拟器游戏。本章还提供安装示例、系统要求与性能说明。

## 终端与文本游戏

OpenBSD ports 树中包含许多可在终端或文本界面下直接运行的游戏，适合资源受限的系统或偏好极简玩法的用户。

### Roguelike 与互动小说

经典的迷宫探险游戏和文字冒险游戏十分丰富：

- `adom`：Ancient Domains of Mystery，一款带任务和技能系统的奇幻 Roguelike。
- `cataclysm-dda`：Cataclysm: Dark Days Ahead，一款末世生存 Roguelike。
- `crawl`：Dungeon Crawl Stone Soup，以战术深度和活跃社区著称。
- `frotz`：用于解释 Infocom 风格的文字冒险游戏，如《Zork》。
- `nethack`：一款复杂耐玩的 Roguelike，玩家深入"危险之迷宫"（Mazes of Menace）。
- `tome4`：Tales of Maj'Eyal，一款现代、剧情驱动的 Roguelike（也有图形版）。

### 解谜与街机类游戏

体积小、独立完整的游戏，适合快速游玩：

- `2048-cli`：合并数字解谜游戏的终端版。
- `bastet`："混蛋俄罗斯方块"，会给玩家最差的方块。
- `bs`：简单的终端战棋游戏。
- `cgames`：终端经典游戏合集，包括俄罗斯方块、贪吃蛇和扫雷。
- `greed`：数字迷宫挑战，移动时会消耗方块。
- `moon-buggy`：驾驶月球车避开陨石坑。
- `nsnake`：贪吃蛇克隆版，带撞墙检测和计分。
- `robotfindskitten`：超现实文字游戏，机器人需在 ASCII 物件中找到小猫。
- `sudoku`：流行逻辑解谜游戏的控制台版。
- `tint`：俄罗斯方块克隆版，终端界面流畅。

## 图形游戏

这些游戏需要 X Window System，提供更丰富的画面与玩法。性能视硬件加速支持和系统资源而定。

### 策略、角色扮演与模拟

- `endless-sky`：受《Escape Velocity》启发的太空探索与贸易游戏。
- `freeciv`：文明风格的帝国建设与科技发展游戏。
- `opencity`：带 3D 图像的城市建设模拟游戏。
- `openra`：重制《Command & Conquer》系列，支持模组。
- `openttd`：《Transport Tycoon Deluxe》的开源版本。
- `simutrans`：带物流与多人模式的交通运输模拟游戏。
- `tome4`：也有图形版 Roguelike。
- `wesnoth`：《Battle for Wesnoth》，一款精致的回合制奇幻策略游戏。

### 动作、街机与平台游戏

- `abe`：受《Oddworld: Abe's Oddysee》启发的平台游戏。
- `openarena`：《Quake III》风格的竞技场射击游戏。
- `supertux`：类似《Super Mario Bros》的横向卷轴平台游戏。
- `supertuxkart`：卡通主题的 3D 竞速游戏。
- `teeworlds`：多人 2D 平台射击游戏，节奏明快。
- `xonotic`：快节奏的竞技场第一人称射击游戏，物理表现优秀。

### 模拟器与复古游戏

OpenBSD 通过模拟器提供对众多历史系统的访问：

- `dosbox`：模拟 MS-DOS，兼容众多基于 DOS 的游戏。
- `fuse`：ZX Spectrum 模拟器。
- `mame`：多系统街机模拟器。
- `mednafen`：多功能模拟器，支持 NES、SNES、Game Boy 等。
- `scummvm`：支持 LucasArts 和 Sierra 的经典点击式冒险游戏。
- `vice`：模拟 Commodore 64 及相关系统。

ROM 必须合法获取，不含在模拟器软件包中。

## 安装与启动游戏

游戏可用 `pkg_add` 安装。例如：

```sh
# pkg_add tome4 wesnoth crawl
$ tome4
```

之后游戏会出现在用户的 `PATH` 中，按名称即可启动。终端游戏可在文本控制台或终端模拟器中运行，图形游戏则需要 X 会话。

## 系统注意事项

### 图形硬件

OpenBSD 通过 `drm(4)` 为部分 Intel、AMD 和较新的 NVIDIA 设备提供硬件加速图形支持。SDL 和 OpenGL 游戏在受支持的 GPU 上运行良好，但由于缺乏 Vulkan 支持和专有驱动，对性能要求高的游戏或基于 Vulkan 的游戏无法运行。

### 音频配置

图形游戏通常需要 `sndiod(8)` 输出声音。确保声音守护进程正在运行：

```sh
# rcctl enable sndiod
# rcctl start sndiod
```

可用 `sndioctl(1)` 和 `mixerctl(1)` 调节音量和设备参数。

### 输入设备

键盘和鼠标输入全面支持。USB 手柄可能由 `uhid(4)` 识别，但按钮重映射可能需要手动完成。OpenBSD 没有类似 `evdev` 或 `xpad` 的原生手柄映射层。

### 性能说明

OpenBSD 优先保证正确性和安全性，而非极致性能。轻量级 2D 和策略游戏通常运行良好。但高端 3D 游戏（尤其是需要 Vulkan 的游戏）可能性能受限或无法运行。这类需求更适合在虚拟机或专用游戏系统中运行现代游戏。

## 浏览可用游戏

用以下命令搜索软件包仓库：

```sh
$ pkg_info -Q games
$ pkg_info -Q emulators
```

查看单个软件包信息：

```sh
$ pkg_info openttd
```
