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
repository. It was first written at commit `d8ff8f4` (2026-10-29) and checked again at `0fa7f55` / `3039770`
(2026-11-26). Its status is **beta**, and its interface (`include/plat.h`, `fm1.toml`) may change before 1.0 (its
README). The project is very active: read its README for anything newer. See [Choosing a Base](Choosing-a-Base.md)
for the other roads.

## How it is built

```
your app (any licence)          app_init, app_frame (UI, ~60/s), app_render (audio), app_tick (1 kHz)
  |  include/plat.h             the platform interface: about 230 plat_* functions in groups, only the ones you turn on
fm1-base (MPL-2.0)              device/ (FM-1), host/ (simulator), web/ (emulator, installer), tools/build.py
felucca-bsp (MPL-2.0, submodule)  chip, board, update loader, packaging
```

**[CODE]** One `tools/build.py` driven by the firmware's `fm1.toml` (name, package identity, USB names, sources,
optional parts). It checks the rules (no `double`, no hardware access outside the BSP and fm1-base's device
files, no call into the chip ROM; a note, not an error, on 64-bit multiplies next to loads) and packs the `.fwsc` with felucca-bsp's `fm1pkg.py`.
The same app runs on the FM-1, in the host simulator and in the browser (DESIGN.md). Proof firmwares: CTL-1
(Bluetooth MIDI) and Radio (WiFi); RemoteFM1 0.2, LOOM-8, AMB-1, Polly, MONK-1, Boardwalk and Optimist (since 0.4.1)
also build on it.

**The size of `plat.h`.** DESIGN.md still says `plat.h` is "the union of what the 13 firmwares use today (68 functions)".
That was the start. On 2026-11-26 `include/plat.h` declares about 230 distinct `plat_*` names (counted with a search
for `plat_` followed by a bracket). Read `plat.h` itself, not the count.

**The audio block. [CODE]** The app chooses it: `APP_BLOCK` in `app.h` (the template: 256 frames, 5.8 ms at 44.1 kHz;
`device/unity.c`). `app_render` runs in the audio interrupt. Two options help with late renders: `audio_aside`
renders into a buffer of its own and copies it into the DMA half only while that half is still free (Henri David's;
2 KiB of RAM), and `audio_ahead` renders one block ahead, so a render may take up to two halves without a dropout, for
one block more of latency (`tools/build.py` help text).

**Build hosts. [CODE]** Linux x86-64 runs JieLi's toolchain natively. On macOS it runs in a container (Docker or podman),
with the image pinned. `tools/setup_toolchain.sh` fetches the toolchain, the SDK's three files and the other tools,
each checked against `tools/toolchain.lock` (README).

## The features the thread named

All **[CODE]** from fm1-base's README, DESIGN.md, `include/plat.h` and `tools/build.py`, unless tagged otherwise.

| Feature | How you turn it on | How it is done |
| --- | --- | --- |
| **USB MIDI** | always | 4-byte USB-MIDI packets; ports `PORT_USB`, `PORT_BLE`, `PORT_TRS`. Out to USB and Bluetooth only: **the TRS jack is in only**. Per-port counters, queue room for a sequencer's note-offs, `plat_midi_in_from()` for MIDI thru, `plat_midi_rt` for a clock tick sent at once from the fast tick (`APP_TICK_FAST`). USB **host** (a USB-MIDI keyboard into the FM-1) with `usb_host = true`, from felucca-bsp; the FM-1 gives the port no power. `usb_serial = true` gives a serial number from the chip's ID, so macOS tells several FM-1s apart. |
| **USB audio** | on by default (`usb_audio_in`, `usb_audio_out`) | The master output recorded by a computer, and a computer playing through the FM-1 (mixed after the app, at MASTER). `usb_audio_channels = 4…10` adds the app's own channels after the master (44.1 kHz only); then the input scan's timer stays at 10 kHz, so the stream is serviced every 0.1 ms. `usb_midi_last` (on by default) lists MIDI after the audio interfaces, which macOS 13 to 15 need to show the audio device. Off in a build with JieLi's WiFi (it needs the RAM). |
| **Dual core** | `cpu2 = true` | felucca-bsp's `chip/fm1_cpu1.h` (from Melodee): a worker for part of the render, one job at a time (`plat_cpu2_run`, `plat_cpu2_wait`), started in `app_init` at power-on before audio and timers. A misread job, a fault, a job not back in **8 ms** or no answer at start **restarts the FM-1** with the reason in the crash record; two failed starts are safe mode. With `cpu2_fallback = true` it is given up instead and the render does the work on one core until power-off. See [Second Core](Second-Core.md). |
| **Velocity and aftertouch** | `#define APP_TOUCH` in app.h | No hardware change (lib/touch.h, from CTL-1 and MONK-1, the Velo work). The **20 keys on the analog rows (F3 to C5)** are sensed through the scan's ADC readings: velocity at each press and a **pressure** (0–127) while held, i.e. per-key aftertouch. The **7 top-row keys (C#5 to G5)** have no converter: a velocity from a probe through the LED line, **no pressure**. Velocity curve and pressure sensitivity −5..+5. Off by default. The readings were measured at the default input rate (10 kHz) only. See [Controls](Controls.md). |
| **BLE MIDI** | `ble = true` (+ `ble_central` for the FM-1 to connect to a keyboard) | Two stacks. **JieLi's** binary libraries through a shim (needs the AC79 SDK). Or **fm1-base's own stack** (`ble_stack = "own"`, Henri David's from Optimist, MPL-2.0): nothing of JieLi's is linked (the build checks), the radio calibrates itself at each start, and it adds pairing and **A2DP audio** to headphones (150–300 ms late: for listening, not playing). Own stack: one BLE MIDI link at a time. Tried on one FM-1 with a Mac, an iPhone, headphones and a speaker **[HW]**. |
| **WiFi** | `wifi = true`; with `wifi_stack = "own"` beside the own Bluetooth stack | Two stacks, as for Bluetooth. **JieLi's** WiFi libraries and lwIP from the SDK, network code in its own task (`plat_net_task`), not with `ble`: about 300 KB of flash, 100 KB of RAM and 276 KB of pool, leaving an app about 170 KB of flash. **fm1-base's own stack** (merged 2026-11-25, [c734409](https://github.com/charlesvestal/fm1-base/commit/c734409), from Optimist): a WPA2-PSK station with a small IP layer (ARP, DHCP, IGMP and UDP; **no TCP and no DNS**), run from the main loop, beside Bluetooth. Options: MODE AP (`wifi_ap`: the FM-1 as an access point, 192.168.4.1, up to four clients), MODE JANK (`wifi_jank`: FM-1s in an open ad-hoc group, no keys), a USB network adapter on the cable (`usb_net`, merged 2026-11-26, [0fa7f55](https://github.com/charlesvestal/fm1-base/commit/0fa7f55)), and **Ableton Link** (`link`, GPL-3.0-only: below). |

**Where the own WiFi stack was tried. [HW]** On an FM-1 in Optimist's firmware, with the same code before it moved:
a WPA2 join, a DHCP lease, Link discovery and Live in sync (2026-10-27); MODE AP with an iPhone running Ableton Note,
which joined and showed one Link peer (2026-10-28). Not tried on an FM-1: this fm1-base build itself, WiFi beside
Bluetooth audio, a long soak ([docs/wifi/README.md](https://github.com/charlesvestal/fm1-base/blob/3039770/docs/wifi/README.md)).
In fm1-base it is tested on the host simulator's own network.

**Bluetooth fixes and open problems. [HW]** BLE MIDI links to some devices (a CME WIDI Master) dropped within seconds,
and a dead link stayed CONNECTED; fixed in the own stack's radio code
([fm1-base#34](https://github.com/charlesvestal/fm1-base/issues/34), Jeremy-MW, 2026-11-03). A Roland FP-30X refused
notes on an unencrypted link: the own stack now pairs and asks again
([#44](https://github.com/charlesvestal/fm1-base/pull/44), [#45](https://github.com/charlesvestal/fm1-base/pull/45),
2026-11-08). Still open: with `ble_central`, a device can't connect while a computer is already connected; the other
order works ([#10](https://github.com/charlesvestal/fm1-base/issues/10), 2026-10-23).

### What each costs

Measured on the template (October and November 2026, its README), in KB:

| | Flash | RAM | Pool |
| --- | --- | --- | --- |
| The template without a radio | 77 | 27 | 68 |
| BLE MIDI on JieLi's stack | + 124 | + 45 | + 24 |
| BLE MIDI on the own stack | + 41 to 46 | + 10 | + 1 |
| Own stack with pairing, devices and Bluetooth audio | + 57 to 62 | + 8 | + 30 |
| WiFi on JieLi's stack (from fm1-radio) | + about 300 | + about 100 | + about 276 |
| WiFi on the own stack (on the own Bluetooth stack) | + 18.6 | + 3.8 | 0 |
| Ableton Link too | + 8.8 | + 1.8 | 0 |
| MODE AP too | + 6.4 | 0 | + 1.7 |

The default slot is 581 KB of flash, 96 KB of RAM (128 KB with JieLi's WiFi) and a 336 KB pool (304 KB with JieLi's
WiFi). **On JieLi's libraries, Bluetooth or WiFi, not both**: together they don't fit. **On fm1-base's own stacks, both**
(README, "Bluetooth and WiFi"). When this page was first written (`d8ff8f4`, 2026-10-29) only JieLi's WiFi existed,
so "not both" was true for every build then. **[CODE]**

### Other parts added since October

All **[CODE]** from `tools/build.py`'s help text and the README, unless tagged otherwise; each is off by default.

- **More room.** `app_end` gives the app area a later end, up to `0xE0000` (at `0xCE000`, app.bin up to 823,228 bytes).
  fm1-base's update loaders erase an older, bigger app's code above their own end; Felucca's and M-VAVE's loaders
  don't. m-bootloader's MIDI mode writes the whole area, but not the resource region (README, "A bigger app slot").
  `ram_layout = "packed"` puts RAM code, `.data`/`.bss` and the pool in one SRAM budget of 464 KB; the radio's DMA
  must stay below `0x01C20000`, and the build checks it. `cache_ram_ways = 2, 4 or 7` turns data-cache ways into RAM at
  `0x01F28000` (with 7, flash data reads over 16 KiB are 57× slower on an FM-1). `boot_pack` stores RAM code and data
  LZMA-packed (from Melodee). `PLAT_SPARE` (2026-11-25): about 13.8 KB of NOINIT RAM, zeroed at boot.
- **CPU and input.** `cpu_clock = 396` runs the CPU at 396 MHz instead of 360, JieLi's top clock "not for mass
  production". fm1-base puts 360 back before every reset and update. **[HW]** Tried on one FM-1: no dropouts on a heavy
  synth, installs both ways clean ([a6630fe](https://github.com/charlesvestal/fm1-base/commit/a6630fe), 2026-11-10).
  `input_hz` (10000 by default, or 5000, 2000, 1000) slows the key scan; at 10 kHz its GPIO work is about 7.5 % of core 0.
  The build's table gives latency and LED flicker per rate; the freed CPU is estimated, not measured.
- **Storage and data.** A microSD card through the [m-card](https://github.com/jvitkauskas/m-card) board (`sd`, FatFs,
  `docs/SD.md`), the card or the app's blocks as a USB drive (`msc`), a resource region written after the app (`resources`),
  up to four extra flash regions (`extras = ["FILE@ADDR"]`), crash records kept in flash (`crash_history`).
- **Tools for hosts.** The companion services over USB (`companion`, SysEx, `docs/COMPANION.md`); `firmware.json`
  written by every build, for hubs and installers (`docs/FIRMWARE-JSON.md`); backup and restore (`backup`); screen
  mirroring (`mirror`); the headphone switch (`headphones`).
- **The newer screens.** `lcd_panel = true` sends M-VAVE's own ST7789V set-up after the reset (porch, gate, VCOM, supplies,
  60 Hz, gammas) ([1677a14](https://github.com/charlesvestal/fm1-base/commit/1677a14), 2026-10-28). The display clock
  stays fm1-base's default, which `device/lcd.c` gives as 60 MHz (about 44 MHz measured): the set-up is sent at the
  start-up clock, and the clock is raised to 60 MHz only for the pixels. Dinghy's note on the newer panels says they
  need M-VAVE's 12 MHz as well
  ([NEWER-FM1-SCREEN.md](https://github.com/zvenson/dinghy/blob/14cda7a/docs/NEWER-FM1-SCREEN.md)). `lcd_panel` has
  not been tried on a newer unit; the commit that added it was not run on an FM-1 at all (Charles Vestal,
  [fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99), 2026-11-27). **[?]** ([Open Questions](Open-Questions.md#display) 42)
- **The newer flash. [HW]** The store, data area, backup and update loader accept the UC25HQ80 (`0xB36014`) since
  2026-11-05. A unit that has an older fm1-base release on such a chip can't install anything itself; felucca-bsp's
  `fm1_rescue.sh` puts stock firmware back once. Still open: units whose protected first 16 KB match neither V15 nor V16
  ([fm1-base#36](https://github.com/charlesvestal/fm1-base/issues/36), Charles Vestal, 2026-11-21).
- **Installing.** The fast install (only changed sectors, resumable) from felucca-bsp, used by `fm1_install.py` and the
  web pages since [5ab0a19](https://github.com/charlesvestal/fm1-base/commit/5ab0a19) (2026-11-05). The identity is
  `id_prefix` plus the version as XXYYZZ (default prefix `FM-1_0`). The plain handshake answers only the name and the
  first three digits (`FM-1_11000100` → `FM-1_110`), because M-UPGRADE 3.11 takes only `NAME_` and one to three digits;
  a handshake with the body `01` gets the whole identity (DESIGN.md, "Build"). The site ships m-bootloader 2.4.
- **The browser emulator's parts.** See [Simulators and Emulators](Simulators-and-Emulators.md).

Also in it since the start: crash record and log readable over USB and from the installer page, `tools/fm1_remote.py`
to drive the panel and read the screen, the screen reader, profiling calls (`plat_cycles*`, `plat_stalls*`,
`plat_cpu_peak_pct`), power readings, `.rfm1` builds for RemoteFM1 on iOS.

## How it differs from Felucca as a base

| | Felucca (and its forks) | fm1-base |
| --- | --- | --- |
| What it is | a complete instrument (engines, sequencer, UI) you fork and cut down | a platform and a template; the instrument is yours |
| Status (2026-11-26) | not actively maintained since 2026-11-24 | beta, very active |
| Licence | GPL-3.0-only | MPL-2.0, file by file |
| Hardware layer | felucca-bsp as a submodule since 1.7; older forks carry their own `hal/` copy | felucca-bsp as a pinned submodule: shared fixes |
| Interface to the app | the platform's functions, used directly | `plat.h`; optional parts compiled in only when asked |
| Audio block | 128 frames (2.9 ms) | the app's choice (the template: 256) |
| Radios | not in Felucca's firmware as far as we know [?] | BLE and WiFi (two stacks each), A2DP, Ableton Link |
| Engines to start from | many, in the GPL family | none of Felucca's (see below) |

## The licence, in both directions

**fm1-base is MPL-2.0, file by file** (`LICENSING.md`): changes to its files are published under MPL-2.0; an
app's own files may use any licence, closed ones included. Its files do not carry MPL's "Incompatible With
Secondary Licenses" notice, so they can be combined with GPL code. How it got there: Leo Kuroshita gave
permission for his Felucca code in felucca-bsp and fm1-base to be MPL-2.0
([felucca-bsp#4](https://github.com/hugelton/felucca-bsp/issues/4)); keremimo agreed for his part
([felucca-bsp#3](https://github.com/hugelton/felucca-bsp/pull/3)); no code by other Felucca contributors is in it.
That permission does **not** relicense Felucca itself. The own Bluetooth and WiFi stacks are Henri David's, offered
as `GPL-3.0-only OR MPL-2.0` and taken under MPL-2.0 (LICENSING.md).

**One opt-in part is GPL-3.0-only: Ableton Link** (`link/`, a C port of Link's tempo, beat and start/stop, by Henri
David). A firmware built with `link = true` is distributed under GPL-3.0-only as a whole, and `tools/build.py` refuses it
unless `fm1.toml` says `licence = "GPL-3.0-only"` (or `-or-later`) (LICENSING.md, README).

| Direction | Can you? | Why |
| --- | --- | --- |
| **fm1-base into a GPL firmware** (a Felucca fork moving onto it) | **Yes** | MPL-2.0 files may be part of a GPL work and stay MPL-2.0 file by file (fm1-base README). Its skill `felucca-to-fm1-base` does this and keeps the firmware's GPL. The firmware as a whole is distributed under the GPL. |
| **GPL code into an fm1-base app that is not GPL** (Felucca's engines, sequencer, UI; code from X0X, Melodee, Jangada and other GPL forks) | **No**, unless the firmware becomes GPL | GPL code makes the whole firmware GPL. To stay MPL or closed, take third-party parts from their own upstream (MIT, BSD, Apache-2.0…), or rewrite from behaviour, not by translating Felucca's source (`skill/felucca-to-fm1-base/RULES.md`). This is the thread's point. |
| **A GPL firmware linking JieLi's binary BT or WiFi libraries** | **Only with an extra permission** | It needs a GPL-3.0 section 7 permission from every copyright holder of its GPL code. An author who owns all of a firmware's GPL code can grant it, as FM1 Open does; a Felucca fork can't, because its code is not only its author's. An MPL-2.0 firmware needs none (fm1-base LICENSING.md, DESIGN.md at [e4863fa](https://github.com/charlesvestal/fm1-base/commit/e4863fa); earlier DESIGN.md said such a firmware could have "no GPL file at all"). |
| **A GPL firmware with BLE (and WiFi) on fm1-base's own stacks** | **Yes [CODE]** | The own stacks link nothing of JieLi's. Ableton Link needs `wifi_stack = "own"`, which needs `ble_stack = "own"`, and makes the firmware GPL-3.0-only; fm1-base ships such a firmware itself (`examples/link`, GPL-3.0-only). Their parts are MPL-2.0 (offered as `GPL-3.0-only OR MPL-2.0` at source), Apache-2.0 (SBC encoder, radio tables) and BSD-2-Clause (micro-ecc). Since 2026-11-27 fm1-base says so in words: "A GPL firmware may use all of fm1-base, the own Bluetooth and Wi-Fi stacks included", with or without Link, since none of its MPL-2.0 files carries the "Incompatible With Secondary Licenses" notice (LICENSING.md, [e4863fa](https://github.com/charlesvestal/fm1-base/commit/e4863fa); Charles Vestal, [fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99)). Not legal advice. |

Relicensing a Felucca-based firmware to MPL-2.0 (to link JieLi's radio libraries) needs every line to be the
author's own or from a permissive source; `tools/provenance.py` finds lines copied from Felucca, including those
that came through contributors' pull requests (RULES.md). **[CODE]**

## When to choose it

- A **new** firmware, or one that wants WiFi or Bluetooth with JieLi's libraries: fm1-base, kept free of GPL code
  (or GPL code you own all of, with a section 7 permission). With fm1-base's own stacks, a GPL firmware is fine.
- A **Felucca fork** that wants shared platform fixes, velocity/pressure, the own BLE and WiFi stacks, or the cpu2
  wrapper: move onto fm1-base and stay GPL ([AI Skills](AI-Skills.md): `felucca-to-fm1-base`). Optimist did this in
  0.4.1.
- A firmware that wants **Felucca's engines**: stay in the GPL family (Felucca, or fm1-base under a GPL app).

## Sources

- [fm1-base](https://github.com/charlesvestal/fm1-base) `README.md`, `DESIGN.md`, `LICENSING.md`,
  `include/plat.h`, `lib/touch.h`, `tools/build.py`, `device/unity.c`, `device/lcd.c`, `docs/wifi/README.md`,
  `skill/felucca-to-fm1-base/RULES.md` (MPL-2.0; first at commit `d8ff8f4`, 2026-10-29; checked again at
  `3039770` and `0fa7f55`, 2026-11-26). Commits [c734409](https://github.com/charlesvestal/fm1-base/commit/c734409),
  [0fa7f55](https://github.com/charlesvestal/fm1-base/commit/0fa7f55), [a6630fe](https://github.com/charlesvestal/fm1-base/commit/a6630fe),
  [1677a14](https://github.com/charlesvestal/fm1-base/commit/1677a14), [5ab0a19](https://github.com/charlesvestal/fm1-base/commit/5ab0a19).
- fm1-base issues and pull requests [#10](https://github.com/charlesvestal/fm1-base/issues/10),
  [#34](https://github.com/charlesvestal/fm1-base/issues/34), [#36](https://github.com/charlesvestal/fm1-base/issues/36),
  [#44](https://github.com/charlesvestal/fm1-base/pull/44), [#45](https://github.com/charlesvestal/fm1-base/pull/45).
- [felucca-bsp#4](https://github.com/hugelton/felucca-bsp/issues/4), [felucca-bsp#3](https://github.com/hugelton/felucca-bsp/pull/3).
- [Dinghy `docs/NEWER-FM1-SCREEN.md`](https://github.com/zvenson/dinghy/blob/14cda7a/docs/NEWER-FM1-SCREEN.md) (2026-11-26).
- [Optimist `docs/RELEASE-0.4.1.md`](https://github.com/w0ts/optimist/blob/b32f7321/docs/RELEASE-0.4.1.md).
- [Mozilla Public License 2.0](https://www.mozilla.org/en-US/MPL/2.0/) §3.3 (distribution of a Larger Work with
  Secondary Licenses) and its [FAQ](https://www.mozilla.org/en-US/MPL/2.0/FAQ/).
- The r/MVaveFM1 thread ["Claude skill"](https://www.reddit.com/r/MVaveFM1/comments/1xh6o7m/claude_skill/), a
  commenter's reply (the commenter's name is not recorded here).
- [fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99): Charles Vestal's answers to this wiki (2026-11-27); fm1-base LICENSING.md, DESIGN.md at [e4863fa](https://github.com/charlesvestal/fm1-base/commit/e4863fa).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
