# Getting Started

From nothing to your own firmware running on an FM-1. This page follows the Felucca-family toolchain, which most
open firmwares use. M-VAVE's own open kit ([FM1 Open](https://github.com/m-vave/fm1_open)) uses a different,
Windows-only build; see [[Choosing a Base]].

> Before your first install, read [[Recovery]] and make a full flash backup with
> [M-Flasher](https://github.com/jvitkauskas/m-flasher). It takes a minute and it is your way back.

## 1. What you need

| Thing | Why | Where |
| --- | --- | --- |
| JieLi's toolchain (clang 4.0.1 for the pi32v2 CPU) | compiles and links for the FM-1's CPU | JieLi's package server (below). Linux x86-64 and Windows builds exist. |
| Three files from JieLi's AC79 SDK (Apache-2.0) | the package needs JieLi's first-stage loader and two config blobs | [fw-AC79_AIoT_SDK on Gitee](https://gitee.com/Jieli-Tech/fw-AC79_AIoT_SDK) |
| Python 3 | the build scripts, packaging, installing | your system |
| On macOS or Windows: Docker, or a Linux VM | the Linux toolchain runs only on x86-64 Linux | Docker Desktop, podman, WSL2 |
| A USB data cable | installing over USB-MIDI | |

**[CODE]** The JieLi Linux toolchain is fetched like this in Felucca-derived trees (`tools/get_toolchain.sh`):

```sh
curl -fL https://pkgman.jieliapp.com/s/linux-toolchain | tar -xJ -C ~/.jieli
```

m-bootloader pins an exact toolchain release, `jieli-linux-toolchains-20250324.1`, and checks its SHA-256
([m-bootloader README](https://github.com/jvitkauskas/m-bootloader)). Pinning a version and a hash is a good habit.

**[CODE]** The SDK files a Felucca-style package uses are `cpu/wl82/tools/uboot.boot`, `cfg_tool.bin` and
`cfg/eq_cfg_hw.bin`. Felucca-family trees build against the tag `AC79NN_SDK_V1.2.1_2023-12-13`:

```sh
git clone --depth 1 --branch AC79NN_SDK_V1.2.1_2023-12-13 \
    https://gitee.com/Jieli-Tech/fw-AC79_AIoT_SDK.git ~/fw-AC79_AIoT_SDK
```

Some build scripts check these three files by SHA-256, so a package made from another SDK version is noticed.

**An open toolchain is coming.** [pi32v2-isa](https://github.com/jvitkauskas/pi32v2-isa) and its LLVM fork build
m-bootloader, fm1-base, Felucca and SLOOP with nothing of JieLi's, and the results run on hardware (its README,
October 2026). Watch that project if you need a toolchain you can audit.

## 2. Pick a starting point

Don't start from an empty file. The hard parts (start-up, the audio interrupt, USB-MIDI, the update path, a crash
screen) are done and tested in several bases. [[Choosing a Base]] compares them.

The smallest documented starting points:
- [Dinghy](https://github.com/zvenson/dinghy) (GPL-3.0): "the plumbing done and documented", with a four-voice sine.
- [fm1-base](https://github.com/charlesvestal/fm1-base) (MPL-2.0, beta): a template app on
  [felucca-bsp](https://github.com/hugelton/felucca-bsp), with a host simulator and a browser emulator.

## 3. First build

Each base has its own build command; read its BUILDING or README. What a good build does, whichever base:

- compiles with the FPU on: `-mcpu=r3 -mfprev1` (see [[Performance]]);
- links at the address the boot chain jumps to, `0x02000120` **[CODE]**;
- checks the image: its size against the app slot, that no `double` crept in (the FPU is single precision),
  that code which runs with the flash off calls nothing in flash;
- wraps the app and an update loader into a `.fwsc` package ([[Firmware Package Format]]).

If your build prints a check as failed, **do not install the package it leaves behind.** Some scripts leave the
previous package in place when a check fails. Compare the package's hash or a build stamp before installing.

## 4. First install

Most open firmwares install over USB-MIDI with the same protocol M-VAVE's updater uses
([[Update Protocol and Installers]]):

- from a browser (Chrome or Edge, Web MIDI with SysEx), on the project's install page; or
- from the command line, e.g. `python3 tools/fm1_install.py build/app.fwsc` in Felucca-family trees.

**If the unit runs M-VAVE's FM1 Open firmware**, many installers refuse or fail: put M-VAVE's stock firmware (V15)
back first ([[Update Protocol and Installers]]).

## 5. When something goes wrong

- The unit shows a crash screen: read the PC address on it, then [[Debugging]].
- The unit doesn't start: [[Recovery]].
- The sound clicks or drops out: [[Performance]].

## Next

[[Hardware Overview]] · [[Programming Guide]] · [[Simulators and Emulators]]

## Sources

- Felucca-family build: [Felucca BUILDING.md](https://github.com/hugelton/Felucca/blob/main/BUILDING.md); DUB-1's
  `BUILDING.md`, `tools/get_toolchain.sh`, `tools/build.py` (GPL-3.0).
- [m-bootloader](https://github.com/jvitkauskas/m-bootloader) (toolchain pin), [pi32v2-isa](https://github.com/jvitkauskas/pi32v2-isa).
- [JieLi AC79 SDK](https://gitee.com/Jieli-Tech/fw-AC79_AIoT_SDK) and its
  [documentation](https://doc.zh-jieli.com/AC79/zh-cn/master/index.html) (Chinese).
