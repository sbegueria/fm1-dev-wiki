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
| [pi32v2-llvm](https://github.com/jvitkauskas/pi32v2-llvm) / [pi32v2-isa](https://github.com/jvitkauskas/pi32v2-isa) | open assembler, linker, objdump and LLVM; builds FM-1 firmwares that run on hardware. Optimist builds with it natively on macOS arm64 and Linux x86-64; m-bootloader 3.0 pins release `pi32v2-23.1.6-2` |
| [pi32v2-rust](https://github.com/jvitkauskas/pi32v2-rust) | Rust 1.100.0 with a port to pi32v2, built on pi32v2-llvm (its description: "proof-of-concept"). Release `pi32v2-1.100.0-1` builds m-bootloader 3.0, m-apps and M-Unbricker. Licence file: Apache-2.0 |
| [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) `mvave-build.sh` | podman image with the JieLi toolchain and SDK files, hash-pinned; builds Felucca-family firmwares unchanged ([Getting Started](Getting-Started.md)) |

## Packaging and installing

| Tool | Notes |
| --- | --- |
| `fm1pkg.py` / `fm1pkg_make.py` (felucca-bsp, Felucca) | builds a `.fwsc` from an app and a loader ([Firmware Package Format](Firmware-Package-Format.md)) |
| `fm1_install.py` (felucca-bsp, Felucca, fm1-base) | USB-MIDI installer; `--info` reads the identity. Since 2026-11-05 it tries the **fast install** first (only the changed sectors, resumable) and falls back to the classic install with loaders that don't answer ([felucca-bsp 628cbd0](https://github.com/hugelton/felucca-bsp/commit/628cbd0); [Choosing a Base](Choosing-a-Base.md#felucca-bsp)) |
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
| [fm1buddy](https://github.com/konsumer/fm1buddy) (GPL-3.0) | an ESP32-S3 touchscreen box (Freenove 2.8", no soldering) that runs an FM-1 or SMK-37 without a computer: installs firmware from SD or over Wi-Fi, backs up and recovers, routes USB and Bluetooth MIDI gear, records the FM-1's USB audio to WAV (README, 2026-11-26) |
| [awesome-fm-1](https://github.com/cicloid/awesome-fm-1) `mooring/catalog.json` | a signed list of firmware packages ("Mooring firmware list", revision 11, issued 2026-11-16): 235 entries, each with its SHA-256, size, download address and loader family, plus the SHA-256 of each known update loader and where it comes from. `catalog.signed.json` is the signed envelope, `catalog.json.intoto.jsonl` an in-toto attestation. Which installer reads it is not said in the repository **[?]** |

## Recovery and flash access

| Tool | Notes |
| --- | --- |
| [M-Flasher](https://github.com/jvitkauskas/m-flasher) | backup, restore, install, bootloader install; Windows, macOS, Linux |
| [FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter) | RP2040 dongle: USB_KEY plus flash read/write |
| [jl-uboot-tool](https://github.com/kagaimiq/jl-uboot-tool) | JieLi flasher and dumper; source of `wl82loader.bin` |
| [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker) | soft-brick recovery over a plain cable; double-read full backup; Linux and Windows for flash work (no macOS SCSI back end), `softkey` on macOS too ([Recovery](Recovery.md)) |
| [flipper-fm1-recovery](https://github.com/merthsoft/flipper-fm1-recovery) | Flipper Zero as the USB key |
| `fm1_rescue` (Felucca family, felucca-bsp) | macOS: stock V15 back through download mode; `fm1_rescue.sh` / `fm1_rescue.py` in felucca-bsp take the UC25HQ80 flash too (fm1-base#36, 2026-11-21) |
| [m-bootloader](https://github.com/jvitkauskas/m-bootloader) (MIT) | replaces the stock bootloader: REC at power-on for download mode, PLAY for a recovery menu, a MIDI mode that installs a package from a browser installer with no firmware running, an SD card item. Rust since 3.0 (2026-11-22). Official M-VAVE updates put the stock bootloader back (3.0 release notes) ([Boot Chain and Bootloaders](Boot-Chain-and-Bootloaders.md)) |
| [M-Unbricker](https://github.com/jvitkauskas/m-unbricker) (MIT) | one working FM-1 or SMK-37 revives a broken one over a USB-C cable, from a web page: the working unit runs it from RAM, sends USB_KEY to the broken unit, writes a fresh m-bootloader and erases the rest up to `0xFEFFF`; it never writes `0xFF000` and up, nor the SMK-37's `0xC0000`–`0xC1FFF`. Tested FM-1 ↔ SMK-37 Pro both ways, by its author (README, 2026-11-26) |

## Inspecting

| Tool | Notes |
| --- | --- |
| `objdump -d` (JieLi or pi32v2-isa) | look up a crash PC |
| `fm1_debug.py` (fm1-x0x debug builds) | register peek/poke, clock loop |
| `fm1_remote.py` (fm1-base) | drive the panel and read the screen over USB |
| [RemoteFM1](https://github.com/Quixotic7/RemoteFM1) | the FM-1 as a USB/Bluetooth panel for a firmware running on the computer ([Simulators and Emulators](Simulators-and-Emulators.md)) |
| [jl-misctools](https://github.com/kagaimiq/jl-misctools) | JieLi containers and key files |
| [jieli-ida](https://github.com/donlon/jieli-ida) (MIT) | IDA Pro processor module and ELF loader for pi32v2, work in progress (tested with IDA Pro 9.4) |
| [m-apps](https://github.com/jvitkauskas/m-apps) (MIT) | Rust programs run from RAM without touching the flash (`hello`, `usbdisk`, `slots`, `mp3`, a launcher); `make run` starts one through M-Flasher's `run-ram` (0.9.2 or newer) and prints its log. Early. `AGENTS.md` lists traps found on the hardware |
| Ghidra + [ghidra-jieli](https://github.com/kagaimiq/ghidra-jieli) | disassemble stock firmware |
| Emulators | see [Simulators and Emulators](Simulators-and-Emulators.md) |
| `tools/fm1_bootarea.py` ([kmplngj/Felucca](https://github.com/kmplngj/Felucca)) | offline, read-only: names a dump's or package's boot area (V15, V16, m-bootloader), checks CRCs, reads the chip key and the declared clock |
| `tests/mul64_hload_check.py` (kmplngj/Felucca) | scans a build's disassembly for 64-bit multiplies next to 16-bit loads, the erratum's pattern ([Performance](Performance.md)) |
| `tools/fm1_diagnostics.py` (kmplngj/Felucca) | read-only collector of a device's health, input, crash and boot-clock data over the USB serial console |
| `tools/provenance.py` ([fm1-base](https://github.com/charlesvestal/fm1-base)) | finds lines copied from Felucca in a project, for relicensing ([fm1-base as a Base](FM1-Base.md)) |
| AI assistant skills | see [AI Skills](AI-Skills.md) |

## Sources

Each tool's README, linked above. Checked on 2026-11-26 with GitHub's API (repository, description, licence file):
pi32v2-rust, fm1buddy, M-Unbricker, m-apps, jieli-ida; [m-bootloader](https://github.com/jvitkauskas/m-bootloader)
README and [release 3.0](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0);
[awesome-fm-1](https://github.com/cicloid/awesome-fm-1) `mooring/` at commit `671ce25` (2026-11-16);
[felucca-bsp](https://github.com/hugelton/felucca-bsp) `README.md` at `fdd709f`;
[fm1-base#36](https://github.com/charlesvestal/fm1-base/issues/36).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
