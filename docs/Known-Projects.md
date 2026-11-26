---
title: "Known Projects"
parent: Community
nav_order: 1
---

# Known Projects

Projects that hold **developer knowledge**: bases, references and specifications. For the full list of firmwares,
editors and tools, see [awesome-fm-1](https://github.com/cicloid/awesome-fm-1), the
[FM-1 resource hub](https://github.com/aroum/fm1-custom-fw) and the
[FM-1 Firmware Atlas](https://github.com/wero1414/atlas-fm-1). (The FM-1 Firmware Database,
`stanleygurvich/fm1-firmware-database`, no longer exists: GitHub answers "not found" on 2026-11-26.) Licences and
descriptions are from each project's README and LICENSE file (October 2026; checked again on 2026-11-26). For how the
bases compare, see [Choosing a Base](Choosing-a-Base.md).

## Bases and platforms

| Project | Licence | What |
| --- | --- | --- |
| [Felucca](https://github.com/hugelton/Felucca) (Leo Kuroshita, Hügelton Instruments) | GPL-3.0-only | The first big open firmware; the origin of most community platform code. Not actively maintained since 2026-11-24 (README); release 1.8.3 |
| [felucca-bsp](https://github.com/hugelton/felucca-bsp) | MPL-2.0 | The chip and board layer split out of Felucca, beta; FM-1, Galaxia, Pocket AMP |
| [fm1-base](https://github.com/charlesvestal/fm1-base) (Charles Vestal) | MPL-2.0 | A platform and template on felucca-bsp, with host simulator, browser emulator, BLE MIDI, WiFi, USB audio, velocity and pressure, second core; beta. See [fm1-base as a Base](FM1-Base.md) |
| [Dinghy](https://github.com/zvenson/dinghy) | GPL-3.0-only | The smallest documented firmware to build on; FM-1 and SMK-37 |
| [foma](https://github.com/konsumer/foma) | GPL-3.0-only | One C file, Docker, a WebAssembly preview and installer on one page; can also run on M-VAVE's SDK |
| [fm1-nes](https://github.com/Keitark/fm1-nes) | Apache-2.0; GPL-3.0 parts from fm1-mdx in its USB builds | Source-only integration with JieLi's SDK; app-only updates. No commit since 2026-10-09 |
| [FM1 Open](https://github.com/m-vave/fm1_open) (M-VAVE) | GPLv3 + SDK linking exception | M-VAVE's open app with a binary engine SDK; builds on Windows only. V14 (2026-10-21); no reply from M-VAVE on issues since 2026-10-24 |
| [smk37_open](https://github.com/m-vave/smk37_open) (M-VAVE) | as above | The same for the SMK-37 |
| [Optimist](https://github.com/w0ts/optimist) | GPL-3.0-only | A modular firmware built in the browser from the parts you pick; from SLOOP / Felucca, on fm1-base since 0.4.1 (`docs/RELEASE-0.4.1.md`) |

## Specifications and reverse engineering

| Project | Licence | What |
| --- | --- | --- |
| [m-bootloader](https://github.com/jvitkauskas/m-bootloader) and its [SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) | MIT | An open bootloader with a recovery menu and MIDI mode, in Rust since 3.0 (2026-11-22). Its [PROVENANCE.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/PROVENANCE.md) calls it "an independent reimplementation", written from a specification that was made by studying JieLi's bootloader; it says it contains no JieLi code. It no longer calls itself clean-room (changed on 2026-11-13, [207cc85](https://github.com/jvitkauskas/m-bootloader/commit/207cc85)). The spec is the most complete public description of boot, flash formats, keys and panel |
| [FM-1 resource hub](https://github.com/aroum/fm1-custom-fw) `REVERSE_ENGINEERING.md` | none declared | Pin-by-pin package map from board photos and JieLi's datasheet, the stock updater's internals, protocol notes, case opening; **mixes FM-1 facts with generic AC79 material** (e.g. 4–8 MB flash), so check each point |
| [FM-1-RE](https://github.com/AL-255/FM-1-RE) | WTFPL | Analysis of the stock firmware: architecture, memory map, OTA protocol, Ghidra scripts |
| [pi32v2-isa](https://github.com/jvitkauskas/pi32v2-isa) | MIT-0 | The instruction set as a table, an open assembler/linker, and an LLVM fork that builds FM-1 firmware |
| fm1-bootloader (jvitkauskas) | withdrawn | Was the stock bootloader as readable C. Withdrawn by its author: the repository is gone (2026-11-26). m-bootloader's PROVENANCE.md says why: it was derived from JieLi's bootloader, and whether JieLi's Apache-2.0 SDK licence covers that binary is not certain ([cfa70fa](https://github.com/jvitkauskas/m-bootloader/commit/cfa70fa), 2026-11-13) |
| [m-apps](https://github.com/jvitkauskas/m-apps) | MIT | Rust programs that run from RAM on the FM-1 and SMK-37 (m-bootloader's RAM app format, SPEC §11.8). Early. Its `AGENTS.md` has a "Traps" section of hardware pitfalls found so far |
| [m-card](https://github.com/jvitkauskas/m-card) | CC0-1.0 | A microSD add-on board for the FM-1 that needs no soldering to the device; used by m-bootloader's SD item and fm1-base's `sd` option |
| [jieli-ida](https://github.com/donlon/jieli-ida) | MIT | IDA Pro processor module and ELF loader for pi32v2 (work in progress; tested with IDA Pro 9.4) |
| [skiff-fm1](https://github.com/chancehouston/skiff-fm1) | MIT (its firmware folder: GPL-3.0-only) | `m-vave-fm1-open-source-dossier.md`: a survey of the hardware, flash layout, update and recovery paths and the ecosystem, with confidence tags (November 2026); its own SysEx installer that logs both directions |
| [fw-AC791N_SDK](https://github.com/GizClaw/fw-AC791N_SDK) (GizClaw) | Apache-2.0 (its LICENSE) | A GitHub import of an AC791N SDK from gitcode.com/yunthinker (Chinese README). Not compared with JieLi's AC79 SDK here **[?]** |
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
- [Dinghy `docs/NEWER-FM1-SCREEN.md`](https://github.com/zvenson/dinghy/blob/main/docs/NEWER-FM1-SCREEN.md): why the
  newer FM-1s' screens look washed out with many firmwares, and M-VAVE's full panel set-up (found by Jake Handy for
  [h1](https://github.com/jakehandy/h1)).
- [fm1-mdx](https://github.com/Keitark/fm1-mdx) (GPL-3.0): an MDX/PDX karaoke player on JieLi's SDK; fm1-nes reuses its
  USB audio and serial code. No commit since 2026-10-08.
- [fm1-dock](https://github.com/joakimeriksson/fm1-dock) (GPL-3.0, based on Dinghy): one firmware with synths as modules
  in flash slots, deployed over MIDI; `docs/RULES.md`, `docs/API.md`.
- [Optimist](https://github.com/w0ts/optimist): `docs/INSTALL-PATHS.md` (which installer takes which way),
  `docs/BLE-HW-FACTS.md`, `docs/WIFI-HW-FACTS.md`, `docs/MEMORY-MAP.md`.

Add yours: see [Contributing](Contributing.md).

## Sources

Each project's README and LICENSE file, linked above; repository status and licences checked with GitHub's API on
2026-11-26 (`gh repo view`, `gh api repos/OWNER/REPO/license`). Felucca README at
[ac8cc0e](https://github.com/hugelton/Felucca/commit/ac8cc0e); m-bootloader `docs/PROVENANCE.md` and commits
[207cc85](https://github.com/jvitkauskas/m-bootloader/commit/207cc85), [cfa70fa](https://github.com/jvitkauskas/m-bootloader/commit/cfa70fa);
fm1-nes `THIRD_PARTY.md` at [90d41c5](https://github.com/Keitark/fm1-nes/commit/90d41c5); Optimist `docs/RELEASE-0.4.1.md`.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
