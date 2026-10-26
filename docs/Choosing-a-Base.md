# Choosing a Base

"Should I start from a Felucca fork, or from M-VAVE's official open firmware?" This question came up in the
r/MVaveFM1 thread and was left open. Here is what each road gives you. The facts are from each project's README
as of October 2026; check them again, these projects move fast.

## The roads

| Base | Licence | What it is | Build | Good for |
| --- | --- | --- | --- | --- |
| [felucca-bsp](https://github.com/hugelton/felucca-bsp) | MPL-2.0 | The chip layer only: clock, interrupts, flash, USB, UART, timers, boot entry, board code, update loader, packaging. No sound, no UI. Beta. | JieLi clang, `-Ichip -Iboards/fm1` | A firmware that wants a shared, reviewed hardware layer. |
| [fm1-base](https://github.com/charlesvestal/fm1-base) | MPL-2.0 | felucca-bsp plus a platform layer (screen, fonts, panel, LEDs, audio, MIDI, storage, a screen reader), a host simulator, a browser emulator, installers; optional Bluetooth MIDI, WiFi, USB audio. Beta. | its `tools/build.py` and `fm1.toml` | A new firmware that wants everything above the hardware done too. Your app may use any licence. |
| [Felucca](https://github.com/hugelton/Felucca) and its forks (X0X, SLOOP, …) | GPL-3.0-only | A complete multi-engine synth; the origin of most community platform code. | `build.sh` / `tools/build.py` | Forking a working instrument. Your firmware stays GPL-3.0. |
| [Dinghy](https://github.com/zvenson/dinghy) | GPL-3.0 | The smallest firmware to build on: Felucca's platform, documented, with a four-voice sine. | per its README | Learning the plumbing. |
| [fm1-nes](https://github.com/Keitark/fm1-nes) | Apache-2.0 | Firmware built inside JieLi's SDK (source-only integration), with notes on app-only updates that keep the bootloader. | JieLi SDK | Using the SDK's RTOS and drivers. |
| [FM1 Open](https://github.com/m-vave/fm1_open) (M-VAVE) | "GPLv3 with the FM1 SDK linking exception"; the engine is a binary SDK | M-VAVE's open app: 8 tracks, DX7 and other engines, a web workbench. | Windows only, JieLi Windows toolchain 2.5.3, PowerShell | Extending M-VAVE's own app. |

## Things to weigh

**Licence.** Felucca's code is GPL-3.0-only. A Felucca fork stays GPL and cannot relicense by using felucca-bsp
([felucca-bsp README](https://github.com/hugelton/felucca-bsp)). MPL-2.0 files (felucca-bsp, fm1-base) can be part
of a GPL work and stay MPL file by file. To link JieLi's closed Bluetooth or WiFi libraries, fm1-base's notes say
the firmware has to have no GPL file in it at all ([fm1-base DESIGN.md](https://github.com/charlesvestal/fm1-base/blob/main/DESIGN.md)).
FM1 Open's app links a binary engine under a linking exception. Read [License](License.md) and each project's LICENSE
before you mix code.

**Shared fixes.** A private copy of Felucca's platform stops getting fixes the day you copy it. Since early
October 2026 the shared base gained, among others: the stock clock set at start-up, M-VAVE's supply voltages, an
"UPDATING / DO NOT UNPLUG" screen in the update loader, and installer handling of FM1 Open. If you keep a copy,
compare it with felucca-bsp now and then. fm1-base ships a Claude skill, `felucca-to-fm1-base`, that moves a
Felucca-based firmware onto it and keeps the firmware's GPL.

**Simulator.** fm1-base, Felucca-family trees and others build the same app for the computer, so you can test
without flashing. This matters more than it seems: see [Simulators and Emulators](Simulators-and-Emulators.md).

**RTOS or bare metal.** Felucca-family firmwares are bare metal: the audio renders in an interrupt and the UI runs
in a main loop. fm1-nes and FM1 Open run on JieLi's SDK (a FreeRTOS-derived kernel, per
[FM-1-RE](https://github.com/AL-255/FM-1-RE)). Opinions differ on what is better for a synth: in the thread, one
developer recommended FreeRTOS on both cores, another went without it because of its overhead next to heavy DSP.
Both have shipped firmware.

**Build system.** The thread's author recommends CMake (with or without Ninja) over a custom `build.py`. The
Felucca family's `build.py` is shared lineage, not something to reinvent. If you start fresh, pick what you and
your contributors can maintain.

**The SMK-37.** If you want your firmware on the SMK-37 too, see [SMK-37](SMK-37.md) before you design your UI.

## Sources

READMEs of each project linked above (October 2026);
[the r/MVaveFM1 thread](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Contents](../README.md)
