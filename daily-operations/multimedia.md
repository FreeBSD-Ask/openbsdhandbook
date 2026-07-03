# 多媒体

## 概述

本章介绍如何在 OpenBSD 上配置和使用音频与视频硬件，如何用基本系统的 `sndio(7)` 框架管理声音设备，以及如何从软件包安装视频播放器、编辑器和流媒体工具等多媒体应用。同时介绍网络摄像头使用、媒体抓轨与转码、通过 DLNA 共享家庭媒体，以及向 Twitch 或 YouTube 等服务进行游戏或屏幕直播。

## 音频子系统

### 声音硬件支持

OpenBSD 通过以下驱动支持多种声音接口：

- `azalia(4)` — Intel HD Audio
- `uaudio(4)` — USB 音频设备
- `hdafg(4)` — HDA 编解码前端
- `emu(4)`、`eap(4)` — 旧式 PCI 声卡

检查音频设备检测情况：

```sh
$ dmesg | grep audio
```

确认声音守护进程是否在运行：

```sh
$ pgrep sndiod
```

以默认设置启动 `sndiod(8)`：

```sh
$ sndiod -f rsnd/0 -d -s default
```

开机时启用 `sndiod`：

```sh
# rcctl enable sndiod
# rcctl start sndiod
```

### 用 mixerctl 控制音量

`mixerctl(1)` 工具用于调节音量、切换输入输出声道。

列出当前设置：

```sh
$ mixerctl -av
```

设置主音量：

```sh
$ mixerctl outputs.master=200,200
```

当 `sndiod` 运行时，音量级别在重启后保留。

### Spotify 支持

Spotify 的网页播放器在 OpenBSD 上无法运行，因为它需要 DRM（Widevine），而 OpenBSD 不支持。

可行的替代方案是 `ncspot`，一款使用 Spotify API 的 TUI 客户端：

```sh
# pkg_add ncspot
```

它需要：

- Spotify Premium 账户
- Spotify 开发者令牌（配置一次）

凭据保存在 **~/.config/ncspot/config.toml**。

要通过 Spotify Connect 实现后台播放，安装 `spotifyd`：

```sh
# pkg_add spotifyd
```

配合 `ncspot` 或移动端 Spotify 客户端使用。

## 播放音频文件

基本系统包含：

- `aucat(1)` — 通过 `sndio` 播放和录制
- `sndioctl(1)` — 运行时控制 `sndiod`

播放 WAV 文件：

```sh
$ aucat -i sound.wav
```

### 软件包中的音频播放器

常见选择：

- `mpv` — 现代命令行视频/音频播放器
- `cmus` — 终端音乐播放器
- `audacious` — 轻量级 GTK+ 播放器
- `moc` — Music on Console

安装 `mpv`：

```sh
# pkg_add mpv
```

## 录制音频

从默认输入录制：

```sh
$ aucat -o record.wav
```

检查输入设置：

```sh
$ mixerctl inputs.reclvl
```

也有 `audacity` 等高级应用：

```sh
# pkg_add audacity
```

## MIDI 支持

播放 MIDI 文件：

```sh
# pkg_add fluidsynth
$ fluidsynth -a sndio soundfont.sf2 song.mid
```

也有虚拟键盘软件，如 `vmpk`。

## 视频播放

OpenBSD 使用 `drm(4)` 子系统实现图形加速。硬件支持包括：

- `amdgpu(4)`、`radeondrm(4)` — AMD GPU
- `i915drm(4)` — Intel GPU
- `nouveaudrm(4)` — NVIDIA（有限支持）

查看驱动使用情况：

```sh
$ dmesg | grep drm
```

安装 `mpv` 或 `vlc`：

```sh
# pkg_add mpv
# pkg_add vlc
```

用法示例：

```sh
$ mpv video.mkv
```

## 网络摄像头支持

使用 `uvideo(4)` 驱动的网络摄像头受支持。检查检测情况：

```sh
$ dmesg | grep uvideo
```

用 `ffplay` 实现基本的摄像头预览：

```sh
# pkg_add ffmpeg
$ ffplay -f v4l2 -video_size 640x480 -i /dev/video0
```

网络摄像头支持仅限于兼容 Video4BSD 接口的设备。

## 蓝牙音频

OpenBSD **不支持**蓝牙。

蓝牙协议栈已从内核和用户态中移除。不支持 A2DP 等音频配置文件。无线音频请使用 USB 音频适配器或模拟连接。

## CD 音频与抓轨

播放音频 CD：

```sh
$ cdio play
```

抓轨：

```sh
# pkg_add abcde
$ abcde -d /dev/rcd0c
```

软件包中还提供 `cdparanoia` 和 `cdda2wav`。

## DVD 与视频光盘

加密 DVD 需要 `libdvdcss`：

```sh
# pkg_add libdvdcss vlc
```

播放光盘：

```sh
$ vlc dvd://
```

查看光盘结构：

```sh
$ lsdvd
```

## 家庭媒体串流与投放

### DLNA 与 UPnP

可用的媒体服务器：

- `minidlna` — 轻量级
- `gerbera` — 元数据丰富，带 Web 界面

安装并启动 `minidlna`：

```sh
# pkg_add minidlna
# rcctl enable minidlna
# rcctl start minidlna
```

编辑 **/etc/minidlna.conf** 设置媒体目录。

### KDE 媒体集成

KDE Plasma 通过 `kio-extras` 支持 `upnp://`、`smb://` 等协议。

安装 KDE：

```sh
# pkg_add kde
```

Plasma 的媒体中心、音频路由和文件共享等特性部分可用，但 OpenBSD 上不具备完整的投放功能（如 Chromecast）。

## 直播与捕获（Twitch、YouTube 等）

### OBS Studio

`obs` 提供屏幕与窗口捕获，支持直播到 Twitch、YouTube 等平台。

安装 OBS：

```sh
# pkg_add obs
```

启动：

```sh
$ obs
```

仅支持软件编码，无硬件加速。

### ffmpeg 流媒体

用于脚本化直播或录制：

```sh
$ ffmpeg -f x11grab -s 1920x1080 -i :0.0 -f alsa -i default -c:v libx264 -preset veryfast -c:a aac output.mp4
```

根据屏幕分辨率和可用设备调整输入输出选项。

## 媒体重编码与转封装

### ffmpeg

将 MKV 转为 MP4：

```sh
$ ffmpeg -i input.mkv -codec copy output.mp4
```

提取音频：

```sh
$ ffmpeg -i input.mp4 -vn -acodec copy track.aac
```

转码音频：

```sh
$ ffmpeg -i track.flac -acodec libmp3lame output.mp3
```

### HandBrake

一款图形化转码器，用于 DVD 抓轨和格式转换：

```sh
# pkg_add handbrake
$ handbrake
```

## 调优 sndiod

`sndiod` 可用标志参数自定义。示例：

```sh
# rcctl set sndiod flags "-f rsnd/0 -z 48000 -b 960"
# rcctl restart sndiod
```

此处设置采样率为 48 kHz，块大小为 960 帧。

## 故障排查

- 确保 `sndiod` 正在使用正确的设备运行
- 用 `mixerctl -av` 查看所有音量级别
- 检查 **/dev/audio*** 权限
- 重新连接设备后重启 `sndiod`

## 其他应用

搜索可用的软件包：

```sh
$ pkg_info -Q audio
$ pkg_info -Q video
```
