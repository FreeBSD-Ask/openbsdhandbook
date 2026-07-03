# Games

## Synopsis

This chapter describes a curated selection of games available on OpenBSD, categorized into terminal-based and graphical titles. While OpenBSD does not prioritize gaming performance, it supports a variety of games through its ports and packages system, including strategy, simulation, roguelike, and emulated titles. This chapter also includes installation examples, system requirements, and performance notes.

## Terminal and Text‑Based Games

The OpenBSD ports tree includes many games that run directly within a terminal or text interface. These are ideal for resource-constrained systems or users who prefer minimalist gameplay.

### Roguelikes and Interactive Fiction

Classic dungeon-crawling games and text adventures are well represented:

- `adom`: Ancient Domains of Mystery, a fantasy roguelike with quests and skills.
- `cataclysm-dda`: Cataclysm: Dark Days Ahead, a post-apocalyptic survival roguelike.
- `crawl`: Dungeon Crawl Stone Soup, known for its tactical depth and community involvement.
- `frotz`: Interpreter for Infocom-style text adventures such as *Zork*.
- `nethack`: A highly intricate and enduring roguelike where players descend into the Mazes of Menace.
- `tome4`: Tales of Maj’Eyal, a modern, story-driven roguelike (also has a graphical version).

### Puzzle and Arcade‑Style Games

Small, self-contained games ideal for quick gameplay:

- `2048-cli`: Terminal version of the number-merging puzzle game.
- `bastet`: “Bastard Tetris”, which gives the worst possible block.
- `bs`: Simple terminal-based battleship game.
- `cgames`: Collection of terminal classics including Tetris, snake, and minesweeper.
- `greed`: A numerical maze challenge where movement consumes tiles.
- `moon-buggy`: Drive a lunar rover while avoiding craters.
- `nsnake`: Snake game clone with wall-collision and score tracking.
- `robotfindskitten`: Surreal text-based game in which the robot must find a kitten among ASCII objects.
- `sudoku`: Console-based version of the popular logic puzzle.
- `tint`: Tetris clone with smooth terminal interface.

## Graphical Games

These games require the X Window System and provide richer graphics and gameplay. Performance may vary depending on hardware acceleration support and available system resources.

### Strategy, RPG, and Simulation

- `endless-sky`: Space exploration and trading game inspired by *Escape Velocity*.
- `freeciv`: Civilization-style empire-building and technology game.
- `opencity`: City-building simulation with 3D graphics.
- `openra`: Reimplementation of Command & Conquer titles with mod support.
- `openttd`: Open-source version of Transport Tycoon Deluxe.
- `simutrans`: Transport simulation with logistics and multiplayer.
- `tome4`: Also available as a graphical roguelike.
- `wesnoth`: Battle for Wesnoth, a polished turn-based fantasy strategy game.

### Action, Arcade, and Platform Games

- `abe`: Platform game inspired by *Oddworld: Abe’s Oddysee*.
- `openarena`: Quake III-style arena shooter.
- `supertux`: Side-scrolling platform game similar to *Super Mario Bros*.
- `supertuxkart`: Cartoon-themed 3D racing game.
- `teeworlds`: Multiplayer 2D platform shooter with fast-paced action.
- `xonotic`: Fast-paced arena-style first-person shooter with advanced physics.

### Emulators and Retro Gaming

OpenBSD provides access to a wide range of historical systems via emulators:

- `dosbox`: Emulates MS-DOS; compatible with many DOS-based games.
- `fuse`: ZX Spectrum emulator.
- `mame`: Multi-system arcade machine emulator.
- `mednafen`: Versatile emulator for NES, SNES, Game Boy, and others.
- `scummvm`: Supports classic point-and-click adventure games from LucasArts and Sierra.
- `vice`: Emulates the Commodore 64 and related systems.

ROMs must be legally acquired and are not included with emulator packages.

## Installing and Launching Games

Games can be installed using `pkg_add`. For example:

```sh
# pkg_add tome4 wesnoth crawl
$ tome4
```

Games will then be available in the user’s `PATH` and can be started by name. Terminal-based games can be played in a text console or terminal emulator, while graphical games require an X session.

## System Considerations

### Graphics Hardware

OpenBSD supports hardware-accelerated graphics via `drm(4)` for select Intel, AMD, and newer NVIDIA devices. SDL and OpenGL games generally run well on supported GPUs, but performance-intensive or Vulkan-based games are not feasible due to the lack of Vulkan support and proprietary drivers.

### Audio Configuration

Graphical games typically require `sndiod(8)` for sound output. Ensure that the sound daemon is running:

```sh
# rcctl enable sndiod
# rcctl start sndiod
```

Volume and device parameters can be tuned using `sndioctl(1)` and `mixerctl(1)`.

### Input Devices

Keyboard and mouse input is universally supported. USB gamepads may be recognized by `uhid(4)`, although button remapping may be required manually. There is no native joystick mapping layer akin to `evdev` or `xpad`.

### Performance Notes

OpenBSD prioritizes correctness and security over raw performance. Lightweight 2D and strategic games typically run well. However, high-end 3D titles, especially those requiring Vulkan, may exhibit performance limitations or fail to run. Running modern games inside a virtual machine or on a dedicated gaming system may be preferable for those requirements.

## Browsing Available Games

Use the following to search the package repository:

```sh
$ pkg_info -Q games
$ pkg_info -Q emulators
```

To inspect individual packages:

```sh
$ pkg_info openttd
```
