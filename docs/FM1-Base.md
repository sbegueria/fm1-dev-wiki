---
title: "fm1-base as a Base"
parent: Start
nav_order: 4
---

# fm1-base as a Base

[fm1-base](https://github.com/charlesvestal/fm1-base) (Charles Vestal) is a base for new FM-1 firmware: the
platform layer above the hardware, on top of [felucca-bsp](https://github.com/hugelton/felucca-bsp). In the
r/MVaveFM1 "Claude skill" thread a commenter recommended it as "much better than using Felucca as base", with
"BLE, WIFI, Velocity, USB Audio, Dual Core, USB Midi, Aftertouch", and warned that "since it's not GPL, you won't
be able to easily bring in engines from other firmwares that are GPL". This page checks both points against the
repository (commit `d8ff8f4`, 2026-10-29). Its status is **beta**, and its interface (`include/plat.h`,
`fm1.toml`) may change before 1.0 (its README). See [Choosing a Base](Choosing-a-Base.md) for the other roads.

## How it is built

```
your app (any licence)          app_init, app_frame (UI, ~60/s), app_render (audio), app_tick (1 kHz)
  |  include/plat.h             the platform interface: 68 functions in groups, only the ones you turn on
fm1-base (MPL-2.0)              device/ (FM-1), host/ (simulator), web/ (emulator, installer), tools/build.py
felucca-bsp (MPL-2.0, submodule)  chip, board, update loader, packaging
```

**[CODE]** One `tools/build.py` driven by the firmware's `fm1.toml` (name, package identity, USB names, sources,
optional parts). It checks the rules (no `double`, no hardware access outside the BSP and fm1-base's device
files) and packs the `.fwsc` with felucca-bsp's `fm1pkg.py`. The same app runs on the FM-1, in the host simulator
and in the browser (DESIGN.md). Proof firmwares: CTL-1 (Bluetooth MIDI) and Radio (WiFi); RemoteFM1 0.2 and
LOOM-8 also build on it.

## The features the thread named

All **[CODE]** from fm1-base's README, DESIGN.md and `include/plat.h`, unless tagged otherwise.

| Feature | How you turn it on | How it is done |
| --- | --- | --- |
| **USB MIDI** | always | 4-byte USB-MIDI packets; ports `PORT_USB`, `PORT_BLE`, `PORT_TRS`. Out to USB and Bluetooth only: **the TRS jack is in only**. Per-port counters, queue room for a sequencer's note-offs, `plat_midi_in_from()` for MIDI thru. USB **host** (a USB-MIDI keyboard into the FM-1) with `usb_host = true`, from felucca-bsp; the FM-1 gives the port no power. |
| **USB audio** | on by default (`usb_audio_in`, `usb_audio_out`) | The master output recorded by a computer, and a computer playing through the FM-1 (mixed after the app, at MASTER). Off in a WiFi build (WiFi needs the RAM). |
| **Dual core** | `cpu2 = true` | felucca-bsp's `chip/fm1_cpu1.h` (from Melodee): a worker for part of the render, one job at a time (`plat_cpu2_run`, `plat_cpu2_wait`), started in `app_init` at power-on before audio and timers. A misread job, a fault, a job not back in **8 ms** or no answer at start **restarts the FM-1** with the reason in the crash record; two failed starts are safe mode. With `cpu2_fallback = true` it is given up instead and the render does the work on one core until power-off. See [Second Core](Second-Core.md). |
| **Velocity and aftertouch** | `#define APP_TOUCH` in app.h | No hardware change (lib/touch.h, from CTL-1 and MONK-1, the Velo work). The **20 keys on the analog rows (F3 to C5)** are sensed through the scan's ADC readings: velocity at each press and a **pressure** (0–127) while held, i.e. per-key aftertouch. The **7 top-row keys (C#5 to G5)** have no converter: a velocity from a probe through the LED line, **no pressure**. Velocity curve and pressure sensitivity −5..+5. Off by default. See [Controls](Controls.md). |
| **BLE MIDI** | `ble = true` (+ `ble_central` for the FM-1 to connect to a keyboard) | Two stacks. **JieLi's** binary libraries through a shim (needs the AC79 SDK). Or **fm1-base's own stack** (`ble_stack = "own"`, Henri David's from Optimist, MPL-2.0): nothing of JieLi's is linked (the build checks), the radio calibrates itself at each start, and it adds pairing and **A2DP audio** to headphones (150–300 ms late: for listening, not playing). Own stack: one BLE MIDI link at a time. Tried on one FM-1 with a Mac, an iPhone, headphones and a speaker **[HW]**. |
| **WiFi** | `wifi = true` (not with `ble`) | JieLi's WiFi libraries and lwIP from the SDK; network code in its own task (`plat_net_task`). It costs about 300 KB of flash, 100 KB of RAM and 276 KB of pool, leaving an app about 170 KB of flash. |

Measured costs on the template (October 2026, its README): no radio 77 KB flash / 27 KB RAM / 68 KB pool;
BLE on JieLi's stack +124 / +45 / +24; BLE on the own stack +41–46 / +10 / +1. The slot is 581 KB of flash,
96 KB of RAM (128 KB with WiFi) and a 336 KB pool (304 KB with WiFi). **Bluetooth or WiFi, not both**: together
they don't fit. **[CODE]**

Also in it: crash record and log readable over USB and from the installer page, `tools/fm1_remote.py` to drive
the panel and read the screen, screen mirroring, backup and restore of saved objects, the screen reader,
profiling calls (`plat_cycles*`, `plat_stalls*`, `plat_cpu_peak_pct`), power readings, `.rfm1` builds for
RemoteFM1 on iOS.

## How it differs from Felucca as a base

| | Felucca (and its forks) | fm1-base |
| --- | --- | --- |
| What it is | a complete instrument (engines, sequencer, UI) you fork and cut down | a platform and a template; the instrument is yours |
| Licence | GPL-3.0-only | MPL-2.0, file by file |
| Hardware layer | its own `hal/`, copied into each fork | felucca-bsp as a pinned submodule: shared fixes |
| Interface to the app | the platform's functions, used directly | `plat.h`; optional parts compiled in only when asked |
| Radios | not in Felucca's platform as far as we know [?] | BLE (two stacks), WiFi, A2DP |
| Engines to start from | many, in the GPL family | none of Felucca's (see below) |

## The licence, in both directions

**fm1-base is MPL-2.0, file by file** (`LICENSING.md`): changes to its files are published under MPL-2.0; an
app's own files may use any licence, closed ones included. Its files do not carry MPL's "Incompatible With
Secondary Licenses" notice, so they can be combined with GPL code. How it got there: Leo Kuroshita gave
permission for his Felucca code in felucca-bsp and fm1-base to be MPL-2.0
([felucca-bsp#4](https://github.com/hugelton/felucca-bsp/issues/4)); keremimo agreed for his part
([felucca-bsp#3](https://github.com/hugelton/felucca-bsp/pull/3)); no code by other Felucca contributors is in it.
That permission does **not** relicense Felucca itself.

| Direction | Can you? | Why |
| --- | --- | --- |
| **fm1-base into a GPL firmware** (a Felucca fork moving onto it) | **Yes** | MPL-2.0 files may be part of a GPL work and stay MPL-2.0 file by file (fm1-base README). Its skill `felucca-to-fm1-base` does this and keeps the firmware's GPL. The firmware as a whole is distributed under the GPL. |
| **GPL code into an fm1-base app that is not GPL** (Felucca's engines, sequencer, UI; code from X0X, Melodee, Jangada and other GPL forks) | **No**, unless the firmware becomes GPL | GPL code makes the whole firmware GPL. To stay MPL or closed, take third-party parts from their own upstream (MIT, BSD, Apache-2.0…), or rewrite from behaviour, not by translating Felucca's source (`skill/felucca-to-fm1-base/RULES.md`). This is the thread's point. |
| **A GPL firmware linking JieLi's binary BT or WiFi libraries** | **No**, by fm1-base's reading | "A GPL-3.0 firmware that links them can't be distributed without an extra permission from the GPL code's copyright holders" (RULES.md). An MPL-2.0 firmware can. So WiFi, and BLE on JieLi's stack, need a firmware with no GPL file in it. |
| **A GPL firmware with BLE on fm1-base's own stack** | **Probably yes [?]** | The own stack links nothing of JieLi's; its parts are MPL-2.0 (offered as `GPL-3.0-only OR MPL-2.0` at source), Apache-2.0 (SBC encoder, radio tables) and BSD-2-Clause (micro-ecc), all usable in a GPL-3.0 work. fm1-base does not state this case outright; this is the wiki's reading, not legal advice. |

Relicensing a Felucca-based firmware to MPL-2.0 (to link JieLi's radio libraries) needs every line to be the
author's own or from a permissive source; `tools/provenance.py` finds lines copied from Felucca, including those
that came through contributors' pull requests (RULES.md). **[CODE]**

## When to choose it

- A **new** firmware, or one that wants WiFi or Bluetooth with JieLi's libraries: fm1-base, kept free of GPL code.
- A **Felucca fork** that wants shared platform fixes, velocity/pressure, the radios' own BLE stack, or the cpu2
  wrapper: move onto fm1-base and stay GPL ([AI Skills](AI-Skills.md): `felucca-to-fm1-base`).
- A firmware that wants **Felucca's engines**: stay in the GPL family (Felucca, or fm1-base under a GPL app).

## Sources

- [fm1-base](https://github.com/charlesvestal/fm1-base) `README.md`, `DESIGN.md`, `LICENSING.md`,
  `include/plat.h`, `lib/touch.h`, `tools/build.py`, `skill/felucca-to-fm1-base/RULES.md` (MPL-2.0; commit
  `d8ff8f4`, 2026-10-29).
- [felucca-bsp#4](https://github.com/hugelton/felucca-bsp/issues/4), [felucca-bsp#3](https://github.com/hugelton/felucca-bsp/pull/3).
- [Mozilla Public License 2.0](https://www.mozilla.org/en-US/MPL/2.0/) §3.3 (distribution of a Larger Work with
  Secondary Licenses) and its [FAQ](https://www.mozilla.org/en-US/MPL/2.0/FAQ/).
- The r/MVaveFM1 thread ["Claude skill"](https://www.reddit.com/r/MVaveFM1/comments/1xh6o7m/claude_skill/), a
  commenter's reply (the commenter's name is not recorded here).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
