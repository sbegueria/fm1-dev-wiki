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

**What the bases offer (checked 2026-11-26). [CODE]**

- **fm1-base**: a scripted host simulator (`host/plat_host.c`). Besides keys, knobs and MIDI, a script can dump the
  whole flash to a 1 MiB image or load one (`flash_dump`, `flash_load`), type on the USB serial console and check the
  reply (`console`, `expect_console`), and drive an SD card. Its own network (`host/wifi_air.c`) runs the WiFi and
  Ableton Link tests with no radio.
- **Felucca**: host unit and regression tests (`tests/run_tests.sh`) that render every engine and preset and compare
  hashes with `tests/golden.txt`; they also compare a cost figure with baseline files (`tests/cpu_baseline.txt`,
  `tests/target_budget.txt`); how those figures relate to the device's timing is not described here.
- **Dinghy**: host mode (`tests/run_host.sh`): the same app, the sound as a WAV, the screen as PNGs, from a script.

## Browser emulators of the app

**[CODE]** The same host build compiled to WebAssembly, with a picture of the panel. fm1-base, Felucca and many forks
publish one. Good for showing a firmware before anyone installs it. The compilers differ (checked 2026-11-26):

| Project | Compiler | Notes |
| --- | --- | --- |
| fm1-base | Emscripten (`tools/build.py --web`) | Optional parts (`emu = [...]` in `fm1.toml`, since 2026-10-27): `midi`, `port`, `data`, `image`, `backup`, `wasm`, `reboot`; `resample` (44.1 kHz played at the browser's rate) is on unless left out. Since 2026-11-10, **Copy session** turns what you played into a host-simulator script, so a problem seen in the browser can be replayed and tested ([91798ae](https://github.com/charlesvestal/fm1-base/commit/91798ae)). The same build makes `.rfm1` files for RemoteFM1 on iOS. |
| Felucca | Emscripten (`web/emu/`) | A demo: it keeps the flash's storage sectors in the browser; USB, TRS MIDI and the user sample slots are not emulated (BUILDING.md). |
| foma | plain `clang --target=wasm32 -nostdlib`, **not Emscripten** (`tools/build.py`) | `./fm1 dev` rebuilds on every save, and the page reloads when `build.json` changes. No USB audio input and no Bluetooth in the emulator (README). |
| FM1 Open (M-VAVE) | WASI SDK 27 | The [workbench](https://open.m-vave.com/fm1-workbench) previews the instrument in the browser beside the updater and the project, sample and DX7 managers; the browser build comes from the same C sources (README). It previews MIDI settings without sending performance MIDI out. |
| Optimist | Emscripten | A web simulator; `python tools/optimist.py emu` also runs a full-system emulator (BUILDING.md). |

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
| [FM1.Pocket](https://github.com/utopick/fm1-pocket/blob/pocket/pocket/README.md) (UtopickDesigns) | MIT | a fork of fm1-emu as an offline web app ([utopick.github.io/fm1-pocket/pocket](https://utopick.github.io/fm1-pocket/pocket/)) and an Android app; a firmware list merged from fm1-emu's and the Firmware Hub's catalogues; MIDI keyboards mapped to the keys, knobs and buttons. The Android build runs the emulator as native code and the second core on a second phone core: on a Galaxy S23 it plays 386DX at 240 MHz, per its README (2026-11). With a real FM-1 (Chrome, Edge or Android) it backs up, installs m-bootloader, then installs firmware through m-bootloader's MIDI mode; with the **Tether** firmware on the unit, the emulated firmware's screen, sound and LEDs are sent to the FM-1 and its controls come back ([MIDI and USB](MIDI-and-USB.md)) |
| [fm1-emulator](https://github.com/simonjohansson/fm1-emulator) | GPL-3.0 | Rust; runs `.fwsc`, `.elf`, `.bin` with the screen, buttons and USB serial |
| [m-wave-fm1-browser-emulator](https://github.com/shcherbakov7/m-wave-fm1-browser-emulator) | GPL-3.0 | Rust/WebAssembly, both cores; Russian README |

Good for: running a package before flashing it, debugging start-up, testing installers and recovery code paths.
How closely they model timing, the cache and the flash is described in each project, not here. **[?]**

## Sources

- Each project's README (October 2026); [awesome-fm-1](https://github.com/cicloid/awesome-fm-1).
- Checked again on 2026-11-26: [fm1-base](https://github.com/charlesvestal/fm1-base) `tools/build.py`, `host/plat_host.c`,
  `web/emu/parts/` (commits [b01ecd5](https://github.com/charlesvestal/fm1-base/commit/b01ecd5),
  [91798ae](https://github.com/charlesvestal/fm1-base/commit/91798ae)); [Felucca](https://github.com/hugelton/Felucca)
  `BUILDING.md` (`ac8cc0e`); [foma](https://github.com/konsumer/foma) `README.md`, `tools/build.py` (`885100f`);
  [FM1 Open](https://github.com/m-vave/fm1_open) `README.md` (`4c7496d`); [Dinghy](https://github.com/zvenson/dinghy)
  `README.md` (`14cda7a`); [Optimist](https://github.com/w0ts/optimist) `BUILDING.md` (`b32f7321`).
- [RemoteFM1 README](https://github.com/Quixotic7/RemoteFM1) and `docs/PROTOCOL.md`.
- Felucca-family `host/` simulators and `web/emu/`; [fm1-base DESIGN.md](https://github.com/charlesvestal/fm1-base/blob/main/DESIGN.md).
- [FM1.Pocket](https://github.com/utopick/fm1-pocket) `pocket/README.md` (branch `pocket`, 2026-11-27); r/MVaveFM1, UtopickDesigns, "FM1.Pocket and Tether CFW" (November 2026).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
