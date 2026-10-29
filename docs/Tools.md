---
title: "Tools"
parent: Community
nav_order: 2
---

# Tools

Tools for building, installing, recovering and inspecting. Each links to its source; check its README for the
current state.

## Toolchains

| Tool | Notes |
| --- | --- |
| JieLi Linux toolchain (clang 4.0.1, pi32v2) | `https://pkgman.jieliapp.com/s/linux-toolchain`; m-bootloader pins `20250324.1` by SHA-256 |
| JieLi Windows toolchain 2.5.3 | used by FM1 Open; `C:\JL\pi32` |
| [pi32v2-llvm](https://github.com/jvitkauskas/pi32v2-llvm) / [pi32v2-isa](https://github.com/jvitkauskas/pi32v2-isa) | open assembler, linker, objdump and LLVM; builds FM-1 firmwares that run on hardware |
| [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) `mvave-build.sh` | podman image with the JieLi toolchain and SDK files, hash-pinned; builds Felucca-family firmwares unchanged ([Getting Started](Getting-Started.md)) |

## Packaging and installing

| Tool | Notes |
| --- | --- |
| `fm1pkg.py` / `fm1pkg_make.py` (felucca-bsp, Felucca) | builds a `.fwsc` from an app and a loader ([Firmware Package Format](Firmware-Package-Format.md)) |
| `fm1_install.py` (felucca-bsp, Felucca, fm1-base) | USB-MIDI installer; `--info` reads the identity |
| Web installers (Felucca family, fm1-base) | Chrome/Edge, Web MIDI with SysEx |
| M-UPGRADE (M-VAVE) | official updater, Windows/macOS |
| [fm1-linux-update](https://github.com/fuleo/fm1-linux-update) | stock updates from Linux |
| [Slipway](https://github.com/Discodave-Glitch/slipway) | Windows: firmware list and installer |
| [FM1Kit](https://github.com/ocean/FM1Kit) | Swift: install from iOS/macOS |
| [fm-static](https://seajaysec.github.io/fm-static/) | web installer for units with odd MIDI port names |
| [TransitFM1](https://github.com/Quixotic7/TransitFM1) | silent stopover firmware between two custom firmwares; answers as V15, writes the next firmware first sector last ([Update Protocol and Installers](Update-Protocol-and-Installers.md)) |
| [FM-1-RE](https://github.com/AL-255/FM-1-RE) `tools/fm1_ota.py` | the stock update protocol from Python |
| [fm1-firmware-patcher](https://github.com/czietz/fm1-firmware-patcher) (Unlicense) | binary patches of stock V15 and V16 (detune, aftertouch, colours); the patched file may not be shared |
| [FM-1 Macropad Patch](https://github.com/smememe/fm1-macropad-patch) (MIT) | patch of stock V15: every input sends a USB-MIDI note |

## Recovery and flash access

| Tool | Notes |
| --- | --- |
| [M-Flasher](https://github.com/jvitkauskas/m-flasher) | backup, restore, install, bootloader install; Windows, macOS, Linux |
| [FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter) | RP2040 dongle: USB_KEY plus flash read/write |
| [jl-uboot-tool](https://github.com/kagaimiq/jl-uboot-tool) | JieLi flasher and dumper; source of `wl82loader.bin` |
| [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker) | soft-brick recovery over a plain cable; double-read full backup; Linux and Windows for flash work (no macOS SCSI back end), `softkey` on macOS too ([Recovery](Recovery.md)) |
| [flipper-fm1-recovery](https://github.com/merthsoft/flipper-fm1-recovery) | Flipper Zero as the USB key |
| `fm1_rescue` (Felucca family, felucca-bsp) | macOS: stock V15 back through download mode |

## Inspecting

| Tool | Notes |
| --- | --- |
| `objdump -d` (JieLi or pi32v2-isa) | look up a crash PC |
| `fm1_debug.py` (fm1-x0x debug builds) | register peek/poke, clock loop |
| `fm1_remote.py` (fm1-base) | drive the panel and read the screen over USB |
| [RemoteFM1](https://github.com/Quixotic7/RemoteFM1) | the FM-1 as a USB/Bluetooth panel for a firmware running on the computer ([Simulators and Emulators](Simulators-and-Emulators.md)) |
| [jl-misctools](https://github.com/kagaimiq/jl-misctools) | JieLi containers and key files |
| Ghidra + [ghidra-jieli](https://github.com/kagaimiq/ghidra-jieli) | disassemble stock firmware |
| Emulators | see [Simulators and Emulators](Simulators-and-Emulators.md) |
| `tools/fm1_bootarea.py` ([kmplngj/Felucca](https://github.com/kmplngj/Felucca)) | offline, read-only: names a dump's or package's boot area (V15, V16, m-bootloader), checks CRCs, reads the chip key and the declared clock |
| `tests/mul64_hload_check.py` (kmplngj/Felucca) | scans a build's disassembly for 64-bit multiplies next to 16-bit loads, the erratum's pattern ([Performance](Performance.md)) |
| `tools/fm1_diagnostics.py` (kmplngj/Felucca) | read-only collector of a device's health, input, crash and boot-clock data over the USB serial console |
| `tools/provenance.py` ([fm1-base](https://github.com/charlesvestal/fm1-base)) | finds lines copied from Felucca in a project, for relicensing ([fm1-base as a Base](FM1-Base.md)) |
| AI assistant skills | see [AI Skills](AI-Skills.md) |

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
