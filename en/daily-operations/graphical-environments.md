# Graphical Environments

## Synopsis

This chapter describes how to configure and use graphical environments on OpenBSD. It covers lightweight window managers, full desktop environments, graphical login managers, and common tools required to support a complete desktop experience.

Basic configuration of the X Window System, including input devices and graphics hardware, is described in the [X11](../install-and-configure/x11.md)
chapter and is assumed to be already completed.

## Window Managers

A **window manager (WM)** controls the placement, appearance, and behavior of application windows in the X Window System.

Window managers generally fall into the following categories:

- **Stacking (floating)**: Windows can overlap freely and are usually moved/resized with the mouse.
- **Tiling**: Windows are placed side by side in a non-overlapping layout, usually managed by keyboard shortcuts.
- **Dynamic**: Offers tiling, floating, and tabbed modes that can switch dynamically.

### Common Window Managers on OpenBSD

The following table provides an overview of common window managers available via packages:

| Window Manager | Type | Package | Notes |
| --- | --- | --- | --- |
| `cwm` | Stacking | Base system | Minimal, keyboard-driven, OpenBSD default |
| `fvwm` | Stacking | `fvwm` | Classic, highly configurable |
| `i3` | Tiling | `i3` | Popular tiling WM, actively maintained |
| `openbox` | Stacking | `openbox` | Lightweight and scriptable |
| `fluxbox` | Stacking | `fluxbox` | Fast and minimal, Blackbox fork |
| `icewm` | Stacking | `icewm` | Traditional look, with taskbar |
| `jwm` | Stacking | `jwm` | Very small and resource-efficient |
| `spectrwm` | Tiling | `spectrwm` | Compact tiling WM, good OpenBSD support |
| `dwm` | Tiling | `dwm` | Extremely minimal, must be recompiled |
| `awesome` | Dynamic | `awesome` | Lua-configurable tiling/dynamic WM |

Install any of these with `pkg_add` and configure startup using `.xinitrc` or `.xsession`.

Example for `i3`:

```sh
# pkg_add i3
$ echo 'exec i3' > ~/.xinitrc
$ startx
```

## Desktop Environments

Desktop environments include a window manager, graphical utilities, file manager, terminal emulator, and configuration panels. These offer a more complete experience but require more system resources.

### Xfce

Xfce is a popular choice on OpenBSD due to its low resource usage.

```sh
# pkg_add xfce
$ echo 'exec startxfce4' > ~/.xinitrc
```

### GNOME

GNOME is a modern, full-featured desktop.

```sh
# pkg_add gnome
$ echo 'exec gnome-session' > ~/.xinitrc
```

Requires supporting services such as `messagebus` and `polkit`.

### KDE Plasma

KDE provides a polished, highly configurable desktop.

```sh
# pkg_add kde
$ echo 'exec startplasma-x11' > ~/.xinitrc
```

Due to its modular design, KDE requires more packages and memory.

## Display Managers (Login Managers)

A **display manager** (or **login manager**) is a graphical application that starts X and presents a user login screen. It replaces the need to use `startx` manually.

Display managers typically use `.xsession` instead of `.xinitrc`.

### xenodm

`xenodm` is included in the base system and provides a minimal graphical login.

Enable it at boot:

```sh
# rcctl enable xenodm
# rcctl start xenodm
```

Customizations are found in `/etc/X11/xenodm/`.

### Other Display Managers

The following display managers are available via packages:

- **GDM** (GNOME): `pkg_add gdm`
- **SDDM** (KDE): `pkg_add sddm`
- **LightDM** (lightweight, GTK-based): `pkg_add lightdm`

Enable one as follows:

```sh
# rcctl enable gdm   # or sddm, or lightdm
# rcctl start gdm
```

To launch the correct session, ensure the user has a proper `.xsession`:

```sh
$ echo 'exec startxfce4' > ~/.xsession
```

Display managers typically handle session selection automatically.

## Session Management

Use `.xinitrc` when launching X with `startx`. Use `.xsession` when using a display manager like `xenodm`, `gdm`, or `lightdm`.

To share configuration, `.xinitrc` may source `.xsession`:

```sh
$ echo 'exec ~/.xsession' > ~/.xinitrc
```

## Multi-Monitor Setup

The `xrandr` tool can be used to configure screen layouts dynamically.

Example:

```sh
$ xrandr --output eDP-1 --primary --output HDMI-1 --right-of eDP-1 --auto
```

Add such commands to `.xinitrc` or `.xsession` to apply settings on login.

## Fonts and Appearance

Fonts for desktop environments and applications may be installed as follows:

```sh
# pkg_add fontconfig dejavu-fonts noto-fonts ttf-bitstream-vera
```

Font rendering settings can be adjusted using `~/.Xresources` or `~/.config/fontconfig/fonts.conf`.

More information is available in the [X11](../install-and-configure/x11.md)
chapter.

## Troubleshooting

- If `startx` fails, check `.xsession-errors` in the user’s home directory.
- Ensure session commands (e.g., `startxfce4`) are installed and correct.
- Make sure required daemons are running:

```sh
# rcctl enable messagebus
# rcctl start messagebus
# rcctl enable polkit
# rcctl start polkit
```

## Web Browsers

OpenBSD supports several web browsers, ranging from full-featured GUI-based browsers to minimalist and text-based alternatives. Each is available through the ports system and may be installed via `pkg_add`.

### Resource Limits and Browser Stability

By default, OpenBSD enforces strict per-process memory limits. The default soft limit on data segment size (`datasize-cur`) is **1536 MB (1.5 GB)** per process. Modern web browsers such as Firefox and Chromium can exceed this threshold, especially on complex pages or with multiple tabs, resulting in a crash or unexpected termination.

To increase the limit for a login class or user:

1. Edit `/etc/login.conf`:

   ```conf
   default:\
   	:datasize-cur=4096M:\
   	:datasize-max=4096M:\
   	:\
   	...
   ```
2. Rebuild the login database:

   ```sh
   # cap_mkdb /etc/login.conf
   ```
3. Log out and back in for changes to take effect.

> Ensure adequate system RAM and swap are available before raising memory limits.

### Full-Featured Browsers

These browsers are recommended for most users and support modern web standards including JavaScript, CSS, multimedia playback, and extensions.

#### Firefox

Firefox is the default graphical browser on OpenBSD and is well maintained.

```sh
# pkg_add firefox
```

Variants include:

- `firefox-esr`: Extended Support Release
- `firefox-i18n`: Adds international language packs

Ensure supporting services are enabled:

```sh
# rcctl enable messagebus
# rcctl start messagebus
# rcctl enable avahi_daemon
# rcctl start avahi_daemon
```

#### Chromium

Chromium is an open-source browser from the same codebase as Google Chrome.

```sh
# pkg_add chromium
```

> Chromium is not available on all hardware platforms due to sandboxing or library dependencies. Check `pkg_info -Q chromium` for architecture availability.

### Other GUI Browsers

#### qutebrowser

Minimal Qt-based browser with Vim-style keybindings.

```sh
# pkg_add qutebrowser
```

Requires `python`, `qt5`, and other supporting packages.

#### epiphany

GNOME’s default WebKitGTK browser.

```sh
# pkg_add epiphany
```

Integrates well with GNOME sessions and supports bookmarks and tabs.

#### surf

Minimal WebKit-based browser developed by suckless.org.

```sh
# pkg_add surf
```

No tabs by default; may be used with `tabbed` for multiple windows.

### Lightweight and Text-Based Browsers

These browsers consume fewer resources and may be used over SSH or in low-memory environments.

#### netsurf

Lightweight graphical browser with its own rendering engine.

```sh
# pkg_add netsurf
```

Does not support JavaScript or complex HTML5 features.

#### lynx, w3m, links

Text-mode browsers used in terminals or headless environments.

```sh
# pkg_add lynx w3m links
```

Useful for reading documentation, accessing webmail, or browsing lightweight sites.

### Locale and Font Configuration

Browsers require proper locale and font settings for multilingual display and antialiasing.

Set locale as needed:

```sh
$ export LANG=en_US.UTF-8
$ firefox
```

Install font packages:

```sh
# pkg_add fontconfig dejavu-fonts noto-fonts ttf-bitstream-vera
```

Further configuration options are discussed in the [X11](../install-and-configure/x11.md)
chapter.

### Multimedia Support

Some browsers rely on external tools for video and audio playback. Install the following if needed:

```sh
# pkg_add ffmpeg gst-plugins-good gst-plugins-bad
```

Check browser preferences to enable or disable hardware acceleration.

### Troubleshooting

- Sudden browser exits may indicate a memory limit was exceeded (`datasize-cur`).
- Missing fonts or symbols: verify font packages and `fontconfig` setup.
- Video playback failure: check for `ffmpeg`, `gstreamer`, or `sndio` issues.

## Development Tools

OpenBSD includes many tools suitable for software development, ranging from compilers and debuggers to graphical editors and IDEs. Most are installed via packages, while some are included in the base system.

### Compilers and Build Tools

OpenBSD ships with the `clang` and `lld` toolchain from LLVM in base. It supports C, C++, and Objective-C.

Additional build tools are available via packages:

```sh
# pkg_add gmake cmake autoconf automake libtool pkgconf
```

These are often required when building software from source.

For Rust or Go development:

```sh
# pkg_add rust go
```

For Java:

```sh
# pkg_add openjdk
```

### Version Control Systems

To work with source repositories:

```sh
# pkg_add git cvs mercurial subversion
```

- `git`: Most commonly used
- `cvs`: Required for OpenBSD source trees
- `subversion`, `mercurial`: Less common, but available

### Editors and IDEs

The following graphical editors and IDEs are available:

| Tool | Package | Notes |
| --- | --- | --- |
| `gvim` | `vim-gtk2` | Graphical Vim, uses GTK2 |
| `emacs` | `emacs` | Full graphical Emacs with GUI |
| `kate` | `kate` | KDE Advanced Text Editor |
| `geany` | `geany` | Lightweight GTK IDE |
| `lite-xl` | `lite-xl` | Minimal editor with Lua config |
| `code` | `vscode` | Not available in packages (manual install) |
| `neovim` | `neovim` | Terminal-based, but GUI frontends exist |

Example installation:

```sh
# pkg_add geany
```

`geany` can be launched from a terminal or desktop menu.

### Terminal Emulators

While not strictly development tools, terminal emulators are often used during development. In addition to `xterm` from base, these are available:

- `st` (from suckless): `pkg_add st`
- `rxvt-unicode`: `pkg_add rxvt-unicode`
- `alacritty`: `pkg_add alacritty`
- `kitty`: `pkg_add kitty`
- `tilix`: `pkg_add tilix` (GTK-based)

Choose one that integrates well with the chosen desktop environment and supports required features (e.g., truecolor, Unicode, clipboard).

### Debugging Tools

OpenBSD includes `lldb` as its default debugger in base. Additional tools:

```sh
# pkg_add valgrind gdb strace
```

Note: `valgrind` support is limited depending on architecture.

For graphical debugging, `ddd` is also available:

```sh
# pkg_add ddd
```

### Documentation and Manual Pages

Most OpenBSD development tools are documented in manual pages.

To find a man page:

```sh
$ man clang
$ man git
```

Additional documentation may be installed under `/usr/local/share/doc/` or accessed online via project websites.

### Language Servers and Autocompletion

Modern editors like `neovim`, `emacs`, and `lite-xl` can integrate with Language Server Protocol (LSP) backends for intelligent code completion, linting, and refactoring.

Install with:

```sh
# pkg_add py-pylsp node clang-tools
```

- `py-pylsp`: Python LSP
- `clangd`: C/C++ language server
- `typescript-language-server`: for JavaScript/TypeScript (requires `node`)

Configuration is editor-specific.

## Office Productivity

OpenBSD provides access to a range of office productivity tools, from full office suites to lightweight applications for editing documents, managing spreadsheets, and creating presentations. These can be installed using `pkg_add`.

### LibreOffice

LibreOffice is the most complete office suite available on OpenBSD. It includes:

- **Writer**: Word processor
- **Calc**: Spreadsheet editor
- **Impress**: Presentation creator
- **Draw**: Vector graphics
- **Math**: Equation editor
- **Base**: Database front-end

Install it with:

```sh
# pkg_add libreoffice
```

Launch with:

```sh
$ libreoffice
```

LibreOffice supports ODF and Microsoft Office file formats (.docx, .xlsx, .pptx). It integrates with most desktop environments.

### Calligra Suite

The KDE office suite is also available:

```sh
# pkg_add calligra
```

Includes `words`, `sheets`, `stage`, and other components. Uses Qt-based UI and integrates well with KDE environments.

### AbiWord and Gnumeric

For systems with limited resources, standalone applications are available:

```sh
# pkg_add abiword gnumeric
```

- `abiword`: Lightweight word processor
- `gnumeric`: Lightweight spreadsheet editor

Both are GTK-based and have a smaller footprint than LibreOffice.

### PDF Editing

While most office suites can export to PDF, editing PDF files requires separate tools.

- `pdfarranger`: GUI to merge/split pages — `pkg_add pdfarranger`
- `pdfsam`: Java-based PDF editor — `pkg_add pdfsam`
- `xournalpp`: Annotate and draw on PDFs — `pkg_add xournalpp`

PDF viewing is described in the Document Viewers section.

### Markdown and Text-Based Formats

For users preferring lightweight workflows, tools for writing in plain text or Markdown are available:

- `ghostwriter`: Minimalist Markdown editor — `pkg_add ghostwriter`
- `zathura` or `lowriter` for previewing output
- Pandoc for format conversion:

  ```sh
  # pkg_add pandoc
  ```

Example:

```sh
$ pandoc -o report.pdf report.md
```

## Document Viewers

OpenBSD supports a variety of document viewing applications, ranging from minimal PDF readers to feature-rich multi-format tools. These applications are available through the ports system and can be installed with `pkg_add`.

### PDF Viewers

#### zathura

Minimal, keyboard-driven PDF viewer with plugin support.

```sh
# pkg_add zathura zathura-pdf-poppler
```

- Vim-like keybindings
- Plugin-based architecture
- Supports bookmarks and searching

To open a file:

```sh
$ zathura document.pdf
```

#### mupdf

Ultra-lightweight PDF viewer.

```sh
# pkg_add mupdf
```

- Fast and resource-efficient
- Minimal GUI
- Keyboard-based navigation

#### qpdfview

Tabbed PDF viewer with optional support for DjVu and PostScript.

```sh
# pkg_add qpdfview
```

- Supports annotations and searching
- Suitable for graphical desktop use

#### xpdf

Traditional, minimal PDF viewer using the Motif toolkit.

```sh
# pkg_add xpdf
```

Simple and portable, but lacks modern features.

### PostScript and DjVu

#### gv

Ghostview frontend for PostScript files.

```sh
# pkg_add gv
```

Relies on Ghostscript (`pkg_add ghostscript`) for rendering.

#### djview

Graphical viewer for DjVu files (scanned document format).

```sh
# pkg_add djview
```

Alternatively, `zathura-djvu` provides DjVu support via plugin.

### Image Viewers

While not strictly document viewers, image viewers are often used for scanned files or document images.

- `feh`: Lightweight and scriptable — `pkg_add feh`
- `sxiv`: Simple X image viewer — `pkg_add sxiv`
- `nomacs`: More advanced, Qt-based — `pkg_add nomacs`

### EPUB and eBook Viewers

#### foliate

Modern GTK-based eBook reader supporting EPUB.

```sh
# pkg_add foliate
```

Features include:

- Bookmarking
- Highlighting
- Text-to-speech (if configured)

#### fbreader

Lightweight eBook reader supporting EPUB, MOBI, and more.

```sh
# pkg_add fbreader
```

### Document Viewing from the Terminal

- `less` for plain text files
- `w3m` and `lynx` can be used to view HTML
- `mupdf-gl` offers framebuffer PDF rendering

Example:

```sh
$ less README.md
$ mupdf-gl presentation.pdf
```

## Finance

OpenBSD provides a selection of financial applications suitable for personal budgeting, small business accounting, and investment tracking. These tools are available through packages and integrate with a variety of desktop environments.

### GnuCash

GnuCash is a full-featured accounting package for personal and small business use.

```sh
# pkg_add gnucash
```

Features:

- Double-entry accounting
- Scheduled transactions
- Import support for OFX, QIF, and CSV
- Graphical reports and charts

GnuCash uses GTK and integrates well with Xfce or GNOME. It supports encrypted backups and multiple currencies.

### HomeBank

HomeBank is a personal finance manager with a simpler interface.

```sh
# pkg_add homebank
```

Features:

- Budgeting and trend analysis
- Categorization of expenses
- Import support for QIF and CSV
- Graphical visualization of income and spending

Recommended for users with basic accounting needs.

### KMyMoney

KDE’s personal finance manager.

```sh
# pkg_add kmymoney
```

Features:

- Familiar ledger interface
- Support for bank and investment accounts
- Reconciliation and scheduled transactions
- OFX/QIF import

Best used in KDE Plasma, but works in other environments.

### beancount and hledger

For users who prefer plaintext accounting and version-controlled finances, OpenBSD supports:

```sh
# pkg_add beancount hledger
```

- `beancount`: Python-based double-entry bookkeeping using `.beancount` files
- `hledger`: Haskell-based CLI tool with optional TUI/HTTP interface

Example:

```
2025-07-11 * "Coffee Shop"
  expenses:food:coffee    $3.50
  assets:cash
```

These tools favor simplicity, auditability, and integration with editors and scripting.

### Spreadsheet-Based Accounting

For lightweight accounting, spreadsheet software such as `gnumeric` or `libreoffice-calc` can be used to maintain personal ledgers or small business finances.

```sh
# pkg_add gnumeric
```

Templates are available online or can be created manually.
