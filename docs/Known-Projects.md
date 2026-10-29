---
title: "Known Projects"
parent: Community
nav_order: 1
---

# Known Projects

Projects that hold **developer knowledge**: bases, references and specifications. For the full list of firmwares,
editors and tools, see [awesome-fm-1](https://github.com/cicloid/awesome-fm-1), the
[FM-1 resource hub](https://github.com/aroum/fm1-custom-fw), the [FM-1 Firmware Database](https://github.com/stanleygurvich/fm1-firmware-database)
and the [FM-1 Firmware Atlas](https://github.com/wero1414/atlas-fm-1). Licences and descriptions are from each
project's README (October 2026).

## Bases and platforms

| Project | Licence | What |
| --- | --- | --- |
| [Felucca](https://github.com/hugelton/Felucca) (Leo Kuroshita, Hügelton Instruments) | GPL-3.0 | The first big open firmware; the origin of most community platform code |
| [felucca-bsp](https://github.com/hugelton/felucca-bsp) | MPL-2.0 | The chip and board layer split out of Felucca, beta |
| [fm1-base](https://github.com/charlesvestal/fm1-base) (Charles Vestal) | MPL-2.0 | A platform and template on felucca-bsp, with host simulator, browser emulator, BLE MIDI, WiFi, USB audio, velocity and pressure, second core; beta. See [fm1-base as a Base](FM1-Base.md) |
| [Dinghy](https://github.com/zvenson/dinghy) | GPL-3.0 | The smallest documented firmware to build on |
| [foma](https://github.com/konsumer/foma) | GPL-3.0 | One C file, Docker, a WebAssembly preview and installer on one page |
| [fm1-nes](https://github.com/Keitark/fm1-nes) | Apache-2.0 | Source-only integration with JieLi's SDK; app-only updates |
| [FM1 Open](https://github.com/m-vave/fm1_open) (M-VAVE) | GPLv3 + SDK linking exception | M-VAVE's open app with a binary engine SDK |
| [smk37_open](https://github.com/m-vave/smk37_open) (M-VAVE) | as above | The same for the SMK-37 |
| [Optimist](https://github.com/w0ts/optimist) | GPL-3.0 | A modular firmware platform from SLOOP / Felucca |

## Specifications and reverse engineering

| Project | Licence | What |
| --- | --- | --- |
| [m-bootloader](https://github.com/jvitkauskas/m-bootloader) and its [SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) | MIT | Clean-room bootloader; the spec is the most complete public description of boot, flash formats, keys and panel |
| [FM-1 resource hub](https://github.com/aroum/fm1-custom-fw) `REVERSE_ENGINEERING.md` | none declared | Pin-by-pin package map from board photos and JieLi's datasheet, the stock updater's internals, protocol notes, case opening; **mixes FM-1 facts with generic AC79 material** (e.g. 4–8 MB flash), so check each point |
| [FM-1-RE](https://github.com/AL-255/FM-1-RE) | WTFPL | Analysis of the stock firmware: architecture, memory map, OTA protocol, Ghidra scripts |
| [pi32v2-isa](https://github.com/jvitkauskas/pi32v2-isa) | MIT-0 | The instruction set as a table, an open assembler/linker, and an LLVM fork that builds FM-1 firmware |
| [fm1-bootloader](https://github.com/jvitkauskas/fm1-bootloader) | Apache-2.0 | The stock bootloader, readable C |
| [jielie](https://github.com/kagaimiq/jielie) (kagaimiq) | | Reference site for JieLi SoCs; USB_KEY notes |
| [ghidra-jieli](https://github.com/kagaimiq/ghidra-jieli) | | Ghidra processor module for pi32v2 |
| [JieLi AC79 SDK](https://gitee.com/Jieli-Tech/fw-AC79_AIoT_SDK) and [docs](https://doc.zh-jieli.com/AC79/zh-cn/master/index.html) | Apache-2.0 | The vendor SDK (Chinese docs) |

## Firmwares with notes worth reading

Many firmwares document hardware lessons in their own docs. Some that have shaped this wiki:

- [fm1-x0x](https://github.com/charlesvestal/fm1-x0x): debug builds for register reads, second-core use; issues #10
  (core 1 misreads) and #21 (voltages).
- [Melodee](https://github.com/keremimo/melodee): the second-core worker, USB MIDI host.
- [SLOOP](https://github.com/isod89/sloop-fm1): a groovebox on both cores.
- [Jangada](https://github.com/zednaked/jangada): core supply raising; measured the core-1 misreads.
- [fm1-radio](https://github.com/charlesvestal/fm1-radio): WiFi, HE-AAC; found the 64-bit multiply erratum.
- [Lunar Modulator](https://github.com/ip2k/lunar-modulator): a USB_KEY dongle write-up.
- [Dinghy](https://github.com/zvenson/dinghy): `docs/RULES.md`, the integer-only rules and the build checks (64-bit
  division, ROM addresses, registers outside `hal/`); the div-by-zero trap left off.
- [TransitFM1](https://github.com/Quixotic7/TransitFM1): `docs/SAFETY.md`, power-cut tests of every flash
  operation of an install; where V15 stages its loader.
- [ORBIT](https://github.com/dspaudio/orbit) (SLOOP lineage): explicit "tested on hardware or not" release notes.
- [kmplngj/Felucca](https://github.com/kmplngj/Felucca): dated investigation notes with exact images (deadline misses, the second core's idle polling, display power), diagnostics and agent workflows ([AI Skills](AI-Skills.md)).
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1): AI skills and a build container ([AI Skills](AI-Skills.md)).
- [DUB-1](https://github.com/sbegueria/fm1-dub): CPU-load notes on XIP and the cache, clock registers on a
  settings page.

Add yours: see [Contributing](Contributing.md).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
