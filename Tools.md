# Tools

Tools for building, installing, recovering and inspecting. Each links to its source; check its README for the
current state.

## Toolchains

| Tool | Notes |
| --- | --- |
| JieLi Linux toolchain (clang 4.0.1, pi32v2) | `https://pkgman.jieliapp.com/s/linux-toolchain`; m-bootloader pins `20250324.1` by SHA-256 |
| JieLi Windows toolchain 2.5.3 | used by FM1 Open; `C:\JL\pi32` |
| [pi32v2-llvm](https://github.com/jvitkauskas/pi32v2-llvm) / [pi32v2-isa](https://github.com/jvitkauskas/pi32v2-isa) | open assembler, linker, objdump and LLVM; builds FM-1 firmwares that run on hardware |

## Packaging and installing

| Tool | Notes |
| --- | --- |
| `fm1pkg.py` / `fm1pkg_make.py` (felucca-bsp, Felucca) | builds a `.fwsc` from an app and a loader ([[Firmware Package Format]]) |
| `fm1_install.py` (felucca-bsp, Felucca, fm1-base) | USB-MIDI installer; `--info` reads the identity |
| Web installers (Felucca family, fm1-base) | Chrome/Edge, Web MIDI with SysEx |
| M-UPGRADE (M-VAVE) | official updater, Windows/macOS |
| [fm1-linux-update](https://github.com/fuleo/fm1-linux-update) | stock updates from Linux |
| [Slipway](https://github.com/Discodave-Glitch/slipway) | Windows: firmware list and installer |
| [FM1Kit](https://github.com/ocean/FM1Kit) | Swift: install from iOS/macOS |
| [fm-static](https://seajaysec.github.io/fm-static/) | web installer for units with odd MIDI port names |

## Recovery and flash access

| Tool | Notes |
| --- | --- |
| [M-Flasher](https://github.com/jvitkauskas/m-flasher) | backup, restore, install, bootloader install; Windows, macOS, Linux |
| [FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter) | RP2040 dongle: USB_KEY plus flash read/write |
| [jl-uboot-tool](https://github.com/kagaimiq/jl-uboot-tool) | JieLi flasher and dumper; source of `wl82loader.bin` |
| [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker) | soft-brick recovery over a plain cable |
| [flipper-fm1-recovery](https://github.com/merthsoft/flipper-fm1-recovery) | Flipper Zero as the USB key |
| `fm1_rescue` (Felucca family, felucca-bsp) | macOS: stock V15 back through download mode |

## Inspecting

| Tool | Notes |
| --- | --- |
| `objdump -d` (JieLi or pi32v2-isa) | look up a crash PC |
| `fm1_debug.py` (fm1-x0x debug builds) | register peek/poke, clock loop |
| `fm1_remote.py` (fm1-base) | drive the panel and read the screen over USB |
| [jl-misctools](https://github.com/kagaimiq/jl-misctools) | JieLi containers and key files |
| Ghidra + [ghidra-jieli](https://github.com/kagaimiq/ghidra-jieli) | disassemble stock firmware |
| Emulators | see [[Simulators and Emulators]] |
