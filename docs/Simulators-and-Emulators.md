---
title: "Simulators and Emulators"
parent: Writing firmware
nav_order: 5
---

# Simulators and Emulators

Testing on a computer is faster, safer and repeatable. There are two different kinds of tool, and they answer
different questions.

## Host simulators: your app, built for the computer

**[CODE]** The firmware's app (UI, engine, storage logic) is compiled with the computer's compiler against a
simulated platform layer instead of the hardware. Felucca-family trees, fm1-base and others do this. A script
drives it (press keys, turn encoders, wait), and it writes audio, screenshots and logs.

Good for:
- **Regression tests**: render a scenario, compare the audio and the screen byte for byte with the last good run.
- Catching crashes, out-of-bounds accesses (build with ASan/UBSan), logic bugs.
- Screenshots for the manual.

To keep the device and the host giving the same samples: single-precision floats everywhere, the same maths
library on both, and `-ffp-contract=off` on both (no fused multiply-add).

**It does not tell you the device's timing.** A host profile does not predict which blocks are heavy on the FM-1:
code layout, the cache and flash fetches decide that. One firmware's heaviest piece on the device was among the
lightest on the host (DUB-1, October 2026). Measure on the device ([Performance](Performance.md)).

## Browser emulators of the app

**[CODE]** The same host build compiled to WebAssembly (Emscripten), with a picture of the panel. fm1-base, Felucca
and many forks publish one. Good for showing a firmware before anyone installs it.

## The FM-1 as a panel for a computer

**[CODE]** [RemoteFM1](https://github.com/Quixotic7/RemoteFM1) (GPL-3.0) is a firmware that sends every key,
button, encoder and the MASTER pot to the computer as plain MIDI (keys as notes on channel 16, buttons on channel
15, encoders as relative CCs 16–22, MASTER as 14-bit CC 7/39), and shows on its screen and LEDs what the computer
sends back (16×16 tiles, only the changed ones; over USB-MIDI SysEx, a vendor bulk interface at about 830 KB/s, or
Bluetooth MIDI). A firmware built for the emulator then runs on the computer with the real panel: no reflash per
change. ChoralRoot, Felucca and Melodee run this way through [FM1VST](https://github.com/Quixotic7/FM1VST). It
does not model the device's CPU time. Version 0.2.3, tested on an FM-1 on 2026-10-28 (its README).

## Full-system emulators: the chip itself

These emulate the AC791N and run real firmware images unchanged:

| Project | Licence | Notes |
| --- | --- | --- |
| [fm1-emu](https://github.com/joakimeriksson/fm1-emu) | MIT | runs community firmwares unchanged in the browser, natively or as an AU/VST3 plugin; an SMK-37 emulator too |
| [fm1-emulator](https://github.com/simonjohansson/fm1-emulator) | GPL-3.0 | Rust; runs `.fwsc`, `.elf`, `.bin` with the screen, buttons and USB serial |
| [m-wave-fm1-browser-emulator](https://github.com/shcherbakov7/m-wave-fm1-browser-emulator) | GPL-3.0 | Rust/WebAssembly, both cores; Russian README |

Good for: running a package before flashing it, debugging start-up, testing installers and recovery code paths.
How closely they model timing, the cache and the flash is described in each project, not here. **[?]**

## Sources

- Each project's README (October 2026); [awesome-fm-1](https://github.com/cicloid/awesome-fm-1).
- [RemoteFM1 README](https://github.com/Quixotic7/RemoteFM1) and `docs/PROTOCOL.md`.
- Felucca-family `host/` simulators and `web/emu/`; [fm1-base DESIGN.md](https://github.com/charlesvestal/fm1-base/blob/main/DESIGN.md).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
