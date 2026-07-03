# Multimedia

## Synopsis

This chapter explains how to configure and use audio and video hardware under OpenBSD, manage sound devices using the base system’s `sndio(7)` framework, and install multimedia applications such as video players, editors, and streaming tools from packages. It also documents webcam usage, ripping and transcoding media, home media sharing via DLNA, and game or screen broadcasting to services such as Twitch or YouTube.

## Audio Subsystem

### Sound Hardware Support

OpenBSD supports various sound interfaces through drivers such as:

- `azalia(4)` — Intel HD Audio
- `uaudio(4)` — USB audio devices
- `hdafg(4)` — HDA codec front-end
- `emu(4)`, `eap(4)` — Older PCI sound cards

To check audio device detection:

```sh
$ dmesg | grep audio
```

To verify the sound daemon is running:

```sh
$ pgrep sndiod
```

To start `sndiod(8)` with default settings:

```sh
$ sndiod -f rsnd/0 -d -s default
```

To enable `sndiod` at boot:

```sh
# rcctl enable sndiod
# rcctl start sndiod
```

### Volume Control with mixerctl

The `mixerctl(1)` utility adjusts volume and toggles input/output channels.

List current settings:

```sh
$ mixerctl -av
```

Set master volume:

```sh
$ mixerctl outputs.master=200,200
```

Audio levels persist across reboots when `sndiod` is active.

### Spotify Support

Spotify’s web player does not function on OpenBSD, as it requires DRM (Widevine), which is unavailable.

A viable alternative is `ncspot`, a TUI client using the Spotify API:

```sh
# pkg_add ncspot
```

It requires:

- A Spotify Premium account
- A Spotify developer token (configured once)

Credentials are stored in `~/.config/ncspot/config.toml`.

For background playback via Spotify Connect, install `spotifyd`:

```sh
# pkg_add spotifyd
```

Use it in conjunction with `ncspot` or a mobile Spotify client.

## Playing Audio Files

The base system includes:

- `aucat(1)` — Playback and capture via `sndio`
- `sndioctl(1)` — Runtime control of `sndiod`

Play a WAV file:

```sh
$ aucat -i sound.wav
```

### Audio Players from Packages

Popular options include:

- `mpv` — Modern CLI video/audio player
- `cmus` — Terminal music player
- `audacious` — Lightweight GTK+ player
- `moc` — Music on Console

Install `mpv`:

```sh
# pkg_add mpv
```

## Recording Audio

To record from the default input:

```sh
$ aucat -o record.wav
```

Check input settings:

```sh
$ mixerctl inputs.reclvl
```

Advanced applications such as `audacity` are also available:

```sh
# pkg_add audacity
```

## MIDI Support

To play MIDI files:

```sh
# pkg_add fluidsynth
$ fluidsynth -a sndio soundfont.sf2 song.mid
```

Virtual keyboards are available, such as `vmpk`.

## Video Playback

OpenBSD uses the `drm(4)` subsystem for graphics acceleration. Hardware support includes:

- `amdgpu(4)`, `radeondrm(4)` — AMD GPUs
- `i915drm(4)` — Intel GPUs
- `nouveaudrm(4)` — NVIDIA (limited)

Verify driver usage:

```sh
$ dmesg | grep drm
```

Install `mpv` or `vlc`:

```sh
# pkg_add mpv
# pkg_add vlc
```

Example usage:

```sh
$ mpv video.mkv
```

## Webcam Support

Webcams using `uvideo(4)` are supported. Check detection:

```sh
$ dmesg | grep uvideo
```

Basic webcam preview with `ffplay`:

```sh
# pkg_add ffmpeg
$ ffplay -f v4l2 -video_size 640x480 -i /dev/video0
```

Webcam support is limited to devices compatible with the Video4BSD interface.

## Bluetooth Audio

Bluetooth is **not supported** on OpenBSD.

The Bluetooth stack has been removed from the kernel and userland. No support exists for audio profiles such as A2DP. For wireless audio, use USB audio adapters or analog connections.

## CD Audio and Ripping

To play an audio CD:

```sh
$ cdio play
```

To rip tracks:

```sh
# pkg_add abcde
$ abcde -d /dev/rcd0c
```

`cdparanoia` and `cdda2wav` are also available in packages.

## DVD and Video Discs

Encrypted DVDs require `libdvdcss`:

```sh
# pkg_add libdvdcss vlc
```

Play a disc:

```sh
$ vlc dvd://
```

To inspect disc structure:

```sh
$ lsdvd
```

## Home Media Streaming and Casting

### DLNA and UPnP

Available media servers:

- `minidlna` — Lightweight
- `gerbera` — Metadata-rich web interface

Install and start `minidlna`:

```sh
# pkg_add minidlna
# rcctl enable minidlna
# rcctl start minidlna
```

Configure `/etc/minidlna.conf` to set media directories.

### KDE Media Integration

KDE Plasma supports `upnp://`, `smb://`, and other protocols via `kio-extras`.

To install KDE:

```sh
# pkg_add kde
```

Features such as Plasma’s Media Center, audio routing, and file sharing are partially supported, but full casting functionality (e.g., Chromecast) is not available on OpenBSD.

## Live Streaming and Capture (Twitch, YouTube, etc.)

### OBS Studio

`obs` provides screen and window capture, with support for livestreaming to Twitch, YouTube, and others.

Install OBS:

```sh
# pkg_add obs
```

Launch:

```sh
$ obs
```

Only software-based encoding is supported. No hardware acceleration is available.

### ffmpeg Streaming

For scripted streaming or recording:

```sh
$ ffmpeg -f x11grab -s 1920x1080 -i :0.0 -f alsa -i default -c:v libx264 -preset veryfast -c:a aac output.mp4
```

Edit the input/output options based on screen resolution and available devices.

## Reencoding and Transmuxing Media

### ffmpeg

Convert MKV to MP4:

```sh
$ ffmpeg -i input.mkv -codec copy output.mp4
```

Extract audio:

```sh
$ ffmpeg -i input.mp4 -vn -acodec copy track.aac
```

Transcode audio:

```sh
$ ffmpeg -i track.flac -acodec libmp3lame output.mp3
```

### HandBrake

A GUI-based transcoder for DVD ripping and format conversion:

```sh
# pkg_add handbrake
$ handbrake
```

## Tuning sndiod

`sndiod` can be customized with flags. Example:

```sh
# rcctl set sndiod flags "-f rsnd/0 -z 48000 -b 960"
# rcctl restart sndiod
```

This sets a 48 kHz sample rate and 960-frame block size.

## Troubleshooting

- Ensure `sndiod` is running with the correct device
- Use `mixerctl -av` to inspect all levels
- Check `/dev/audio*` permissions
- Restart `sndiod` after reconnecting devices

## Additional Applications

To search for available packages:

```sh
$ pkg_info -Q audio
$ pkg_info -Q video
```
