---
title: "Choosing a Base"
parent: Start
nav_order: 2
---

# Choosing a Base

"Should I start from a Felucca fork, or from M-VAVE's official open firmware?" This question came up in the
r/MVaveFM1 thread and was left open. Here is what each road gives you. The table is a summary; the profiles below
it say more, with a source for each point. The facts were checked in each repository on **2026-11-26**. Check them
again: these projects move fast.

## The roads

| Base | Licence | What it is | Build on | Status, 2026-11-26 | Good for |
| --- | --- | --- | --- | --- | --- |
| [felucca-bsp](https://github.com/hugelton/felucca-bsp) | MPL-2.0 | The chip and board layer only: clock, interrupts, flash, USB, update loader, packaging. No sound, no UI. | JieLi clang; headers `-Ichip -Iboards/fm1` | Beta, active | A firmware that wants a shared, reviewed hardware layer. |
| [fm1-base](https://github.com/charlesvestal/fm1-base) | MPL-2.0 (a build with Ableton Link: GPL-3.0-only) | felucca-bsp plus a platform layer, a host simulator, a browser emulator and installers; optional Bluetooth, WiFi, USB audio. Deep page: [fm1-base as a Base](FM1-Base.md) | Linux x86-64; macOS in a container | Beta, very active | A new firmware that wants everything above the hardware done. Your app may use any licence. |
| [Felucca](https://github.com/hugelton/Felucca) and its forks (X0X, SLOOP, …) | GPL-3.0-only | A complete multi-engine synth; the origin of most community platform code. | Linux x86-64; macOS through Docker | 1.8.3; **not actively maintained** since 2026-11-24 | Forking a working instrument. Your firmware stays GPL-3.0. |
| [Dinghy](https://github.com/zvenson/dinghy) | GPL-3.0-only | Felucca's platform (as SLOOP carries it), documented, with a four-voice sine and a kit of MIDI basics; FM-1 and SMK-37. | Linux x86-64; macOS or Windows with Docker or WSL | 0.6.1, active | Learning the plumbing; a GPL firmware that also runs on the SMK-37. |
| [foma](https://github.com/konsumer/foma) | GPL-3.0-only | Dinghy made smaller: one C file, a Docker toolchain, an emulator and installer on one page. Can also run on M-VAVE's SDK. | Docker only | Last commit 2026-11-09 | A first firmware with nothing installed but Docker. |
| [fm1-nes](https://github.com/Keitark/fm1-nes) | Apache-2.0, with GPL-3.0 parts in its USB builds | A firmware built inside JieLi's SDK, with app-only updates that keep the bootloader. | Windows (the tested host) | Dormant since 2026-10-09 | Reading how to use the SDK's RTOS and drivers. |
| [FM1 Open](https://github.com/m-vave/fm1_open) (M-VAVE), [smk37_open](https://github.com/m-vave/smk37_open) | GPLv3 with an SDK linking exception; the engine is a binary | M-VAVE's open app: 8 tracks, DX7 and other engines, a web workbench. | Windows x64 only | V14 (2026-10-21); no maintainer reply since 2026-10-24 | Extending M-VAVE's own app. |
| [Optimist](https://github.com/w0ts/optimist) | GPL-3.0-only | A modular firmware built in the browser from the parts you pick; on fm1-base since 0.4.1. | macOS arm64, Linux x86-64 (open LLVM) | 0.4 (2026-11-23), active | Adding an engine or effect to a shared firmware (its CONTRIBUTING.md); its hardware notes. |

## The bases, one by one

Each profile has the same parts: status, licence, how it works inside, how to build it, how to test it without a
unit, how it installs, which hardware it supports, what to read, and its limits.

### Felucca

- **Status.** Release 1.8.3 (2026-11-22). Since 2026-11-24 its README says: "Felucca is not actively maintained";
  the author now works on Fune ([README](https://github.com/hugelton/Felucca/blob/ac8cc0e/README.md), commit
  `ac8cc0e`). It targets the FM-1 only. Earlier ports (SDL2 desktop, libretro, PortMaster, M5Stack CoreS3) are kept,
  unsupported, on the `research/platforms` branch. A fork no longer gets fixes from upstream; felucca-bsp and
  fm1-base are where platform fixes now land.
- **Licence.** GPL-3.0-only as a whole. Six engines (ANALOG, LOFI, TRIO, WHEEL, NOISE, PHASE) come from
  [Felucca-Engines](https://github.com/hugelton/Felucca-Engines) under MIT, and the hardware layer is felucca-bsp
  (MPL-2.0); both are git submodules (the hardware layer since 1.7.0, its release notes). Ports keep their licences: PHYS from DaisySP and Rings (MIT), FM6 from msfa
  (Apache-2.0). A fork stays GPL ([LICENSING.md](https://github.com/hugelton/Felucca/blob/ac8cc0e/LICENSING.md)).
- **Inside. [CODE]** Bare metal. The audio renders in the I2S interrupt, one half buffer at a time: **128 frames,
  2.9 ms** at 44.1 kHz (`firmware/src/core.h` `HALF_FRAMES`). It has been 128 since Felucca 1.0
  ([727f272](https://github.com/hugelton/Felucca/commit/727f272), 2026-10-05); forks taken before 1.0 still use 256.
  After two halves in a row above 85 % of the half's time, it fades out one voice (`audio.c`, `SHED_TICKS`). Four
  parts share 8 voices; patterns have 64 steps (`core.h`). The second core is built in since 1.6.1
  (`FELUCCA_CPU2`), off by default (MENU > SYSTEM > 2ND CORE) ([BUILDING.md](https://github.com/hugelton/Felucca/blob/ac8cc0e/BUILDING.md)).
- **Engines.** ANALOG, PHASE, LOFI, TRIO, WHEEL, NOISE, SAMPLE, SLICE, GRAIN, FORMANT, DRUM, PHYS and FM6; an ADPCM
  engine is in the tree but not built by default (one voice costs about three ANALOG voices, BUILDING.md).
- **Build.** `./build.sh`. The JieLi toolchain (clang 4.0.1) runs only on Linux x86-64; on macOS each tool runs in a
  `linux/amd64` Docker container. It also needs three files of the AC79 SDK, and Python with Pillow and fontTools.
  Older notes cite `firmware/hal/fm1_audio.h` and similar: those files now live in the submodule, under
  `firmware/bsp/chip/` and `firmware/bsp/boards/fm1/`.
- **Without a unit.** Host tests (`tests/run_tests.sh`) render every engine and preset and compare hashes with
  `tests/golden.txt`. A browser demo (Emscripten, `web/emu/`) is on the project site; it does not emulate USB, TRS MIDI
  or the user sample slots.
- **Installing.** Release identity `FM-1_918` (1.8.x). Web installer (Chrome, Edge) or `tools/fm1_install.py`. The
  installer offers the way back to stock V15, and to V16 since 2026-11-15 (tested Felucca → V16 → Felucca on an FM-1;
  after V16, Felucca reads its project and sample areas as empty) ([5044cce](https://github.com/hugelton/Felucca/commit/5044cce)).
  **[CODE]** Holding OCT− and OCT+ for 5 s starts the chip's download mode (UBOOT) with RAM intact; a countdown shows
  from 2 s, and letting go cancels (`firmware/src/main.c`). **[HW]** On 1.8.1 this gave `4C4A:8057` "WL80UBOOT1.00",
  and the flash was dumped with jl-uboot-tool (ronbarrantes, [Felucca#320](https://github.com/hugelton/Felucca/issues/320), 2026-11-09).
- **Hardware.** The newer UC25HQ80 flash (`0xB36014`) is recognised from 1.8.2
  ([f4cbb6a](https://github.com/hugelton/Felucca/commit/f4cbb6a), 2026-11-03). On such a unit, 1.8.1 and earlier keep
  saves in RAM only and cannot start an update; the way out is stock V15 first, then 1.8.2 or later (README, 1.8.2
  release notes). No SMK-37 support.
- **Read.** BUILDING.md (every build flag, with why), LICENSING.md, `docs/signed-audio-arithmetic.md`,
  `docs/trs-uart-events.md`, and the comments in `audio.c` and `main.c`.

### felucca-bsp

- **Status.** Beta, active: the last merge was on 2026-11-26 ([#51](https://github.com/hugelton/felucca-bsp/pull/51),
  a USB network adapter). Many contributors send pull requests (Henri David, Charles Vestal, Julius Vitkauskas,
  keremimo and others). Expect file names and layout to change (README).
- **Licence.** MPL-2.0, file by file. Your own code on top may use any licence, and it can be combined with GPL code.
  A Felucca fork cannot move to MPL-2.0 by using it ([README](https://github.com/hugelton/felucca-bsp/blob/fdd709f/README.md),
  [LICENSING.md](https://github.com/hugelton/felucca-bsp/blob/fdd709f/LICENSING.md)).
- **Inside. [CODE]** No sound code and no user interface. `chip/` holds clock, interrupts, guards, flash, USB
  (MIDI, serial, audio in and out, an opt-in MIDI host, mass storage, a network adapter), timers, the update session
  and boot entry. `boards/fm1/` holds the panel, key scan (velocity and pressure, off by default), ADC, checked
  stereo Q23 audio DMA and the headphone switch. The firmware picks its own audio half size.
- **Experimental parts.** Second core, expanded flash, cache RAM, and the **sliced erase** (`FL_ERASE_SLICED`, merged
  2026-11-22 in [#46](https://github.com/hugelton/felucca-bsp/pull/46)). **[HW]** For the sliced erase, Henri David and
  Charles Vestal measured on an FM-1 with the Puya flash: erase windows of about 1 ms with interrupts off, 19 per
  sector, about 24 ms per sector in all, no blank failures and no audio dropouts over about 2,500 erases. The USB
  audio input still gets breaks during these erases (#46 comments, 2026-11-22). The README still calls a higher clock a
  "concept", while `fm1_clock_pll1` exists and fm1-base uses it ([fm1-base as a Base](FM1-Base.md)).
- **Build.** Not built alone: a firmware includes it (`-Ichip -Iboards/fm1`) and builds it with the JieLi toolchain.
  `tools/` has the package builder `fm1pkg.py`, the installer `fm1_install.py` and `fm1_rescue` (stock firmware back
  through download mode).
- **Without a unit.** Host tests of single parts (`tests/`); no simulator of its own.
- **Installing. [HW]** Its update loader has the **fast install**: the installer says hello (command `0x31`), the
  loader compares each 4 KiB sector by CRC and asks only for the ones that differ, and a cut-off install resumes.
  Measured on an FM-1: a whole install in 23.4 s, 149 answers and 675 KB sent, against 2,797 answers and 1.69 MB the old
  way (hdavid, [#26](https://github.com/hugelton/felucca-bsp/pull/26), 2026-11-01). The pull request was closed
  unmerged, but the code is on main (`loader/loader.c`), and `fm1_install.py` uses it since
  [628cbd0](https://github.com/hugelton/felucca-bsp/commit/628cbd0) (2026-11-05). A host that does not say hello gets the
  classic install.
- **Hardware.** FM-1: supported. M-VAVE Galaxia: supported. M-VAVE Pocket AMP: board code checked on a unit. SMK-37
  Pro and Elite: "planned" in the README; an open pull request adds a board layer
  ([#30](https://github.com/hugelton/felucca-bsp/pull/30), keremimo, opened 2026-11-06, not merged on 2026-11-26).
  The flash driver accepts both the Puya (`0x856014`) and the UC25HQ80 (`0xB36014`) (`chip/fm1_flash.h`).
- **Read.** `docs/BOARD_IO.md` (panel, audio sample format and rate, underruns), `docs/EXPERIMENTAL.md`,
  `docs/FLASH.md`, `docs/CPU.md` (the 64-bit multiply erratum), `docs/USB_HOST.md`.

### fm1-base

The deep page is [fm1-base as a Base](FM1-Base.md). In short:

- **Status.** Beta, very active (commits up to 2026-11-26). Expect `plat.h` and `fm1.toml` to change before 1.0.
- **Licence.** MPL-2.0; your app may use any licence. A build with Ableton Link (`link = true`) is GPL-3.0-only as a
  whole, and the build refuses it unless `fm1.toml` says so.
- **Inside. [CODE]** Bare metal on felucca-bsp. The app's `app_render` runs in the audio interrupt, one block of
  `APP_BLOCK` frames (the template: 256, 5.8 ms). Options let a render be copied in only while its half is free
  (`audio_aside`), or run a block ahead (`audio_ahead`). The second core is a worker for part of the render. JieLi's
  WiFi runs in a task of its own; fm1-base's own Bluetooth and WiFi stacks run from the main loop.
- **Build.** `tools/build.py` and the app's `fm1.toml`. Linux x86-64 natively; macOS through a pinned container image
  (Docker or podman). Every download is pinned by hash in `tools/toolchain.lock`.
- **Without a unit.** A host simulator, a browser emulator and `.rfm1` builds for RemoteFM1 on iOS.
- **Installing.** Identity `id_prefix` plus the version (default prefix `FM-1_0`). Its own update loader with the fast
  install; an installer page; `fm1_install.py`.
- **Hardware.** UC25HQ80 since 2026-11-05; the newer screens with `lcd_panel = true`; a microSD card mod with
  `sd = true`. FM-1 only.

### Dinghy

- **Status.** 0.6.1 (2026-11-26), active (zvenson) ([CHANGELOG.md](https://github.com/zvenson/dinghy/blob/14cda7a/CHANGELOG.md)).
- **Licence.** GPL-3.0-only, as Felucca and SLOOP. The screen mirror code is MIT
  ([LICENSING.md](https://github.com/zvenson/dinghy/blob/14cda7a/LICENSING.md)).
- **Inside. [CODE]** Felucca's platform as SLOOP carries it, bare metal. Your `app_audio()` runs in the audio
  interrupt every 5.8 ms with 256 frames at 44.1 kHz (`docs/API.md`; `platform.c` `HALF_FRAMES 256`). `dinghy.c` is
  the app: a four-voice sine and a GLOBAL page. Since 0.6, a kit of headers from OberVave adds MIDI clock sync (a PI
  loop; 5 minutes with no drift in a host test), MIDI basics (channel, pitch bend, sustain, CC 7), FM-1 Mirror, a
  backup over SysEx and VIBE LINK. Storage: settings (4 KB), 8 blobs of 4 KB and 8 slots of 16 KB, each with a CRC.
- **Build.** `./build.sh` with the JieLi toolchain and three SDK files: Linux x86-64 natively, or macOS / Windows with
  Docker or WSL. The build refuses a 64-bit division, a call into the chip ROM, too much RAM, and a register touched
  outside `hal/`. `./build.sh --board smk37` builds for the SMK-37.
- **Without a unit.** Host mode: the same `dinghy.c` on the computer, the sound as a WAV, the screen as PNGs, keys,
  knobs and MIDI from a script. No browser emulator in the repository.
- **Installing.** Identity `FM-1_99Y` (`SMK-37 Pro_99Y`) for release 0.Y. `tools/fm1_install.py` or the
  [FM-1 Firmware Hub](https://thefirmwarehub.com). USB rescue: hold OCT− while switching on; two crashes at boot go
  there by themselves. **[HW]** But on at least two units, OCT− at power-on never opened the rescue (Dinghy 0.3 and 0.4):
  the keys can't be read until the screen's pins are set up, and PA2 also enables the key matrix's driver. The fix is
  in an open pull request ([dinghy#4](https://github.com/zvenson/dinghy/pull/4), sgarrettnelson 2026-10-17, confirmed by
  KevosG 2026-11-06; not merged on 2026-11-26).
- **Hardware.** UC25HQ80 since 0.5 (2026-11-07). **The newer screens** (units with the FM-1 / M-VAVE logo under the
  screen) showed washed-out colours, no reds or a blank screen; 0.6.1 sends M-VAVE's own panel set-up in M-VAVE's
  order and keeps the display clock at 12 MHz. Found and measured by Jake Handy for
  [h1](https://github.com/jakehandy/h1) 0.3.1. [docs/NEWER-FM1-SCREEN.md](https://github.com/zvenson/dinghy/blob/14cda7a/docs/NEWER-FM1-SCREEN.md)
  explains it for every FM-1 firmware. SMK-37 Pro, Elite and MKE-P37: the board layer is M. Geisler's; that code runs in
  OberVave, and Dinghy's own SMK-37 builds are tested on the host (README).
- **Known issue. [HW]** The USB audio input repeats about 1.7 % of its frames: its ring was sized for 128-frame renders,
  and the platform renders 256 ([dinghy#13](https://github.com/zvenson/dinghy/issues/13), kurzhaar168, 2026-11-14, open).
- **Read.** `docs/API.md`, `docs/RULES.md` (the chip's rules and the build checks), `docs/MAKE-IT-YOURS.md`
  (identity, releases, the hub), `docs/SMK37.md`, `docs/NEWER-FM1-SCREEN.md`.

### foma

- **Status.** Last commit 2026-11-09 ([885100f](https://github.com/konsumer/foma/commit/885100f)).
- **Licence.** GPL-3.0-only. `bigscreen/` is MIT. The `mvave` platform keeps M-VAVE's terms (below)
  ([LICENSING.md](https://github.com/konsumer/foma/blob/885100f/LICENSING.md)).
- **Inside. [CODE]** "Dinghy made smaller": Felucca's platform via SLOOP and Dinghy, under a smaller API like an
  Arduino sketch: `setup()`, `loop()`, `audio()`. `audio()` gets 256 frames every 5.8 ms (README; `platform/fm1.c`).
  Optional parts in `firmware.json`: USB audio, TRS MIDI, MIDI clock, Bluetooth MIDI. Bluetooth uses Henri David's
  stack (from Optimist, as fm1-base carries it); it measures the radio's VCO bands at every start, and the first
  `bt_enable` pauses the sound while the radio starts (README). Unlike fm1-base, it does not run the full radio
  calibration, which hung or faulted in most of its builds (2026-10-31): it uses the unit's stored records, else
  constants (`platform/bt.c`).
- **Two platforms.** `"platform": "felucca"` (the default) is all source. `"platform": "mvave"` runs your code on
  M-VAVE's FM1 SDK from FM1 Open (FreeRTOS, USB, Bluetooth and flash in their binary library); it brings that SDK's
  BLE central role and **Bluetooth audio in** (A2DP sink: `bt_audio_in()`). The README lists the cost:
  about **46 KiB of flash** for your code and data, about 160 KiB of RAM, saved settings lost when you switch, identity
  `FM-1_8` plus six digits, and no OCT− rescue.
- **Build.** Docker only: `./fm1 dev` pulls the `konsumer/foma-toolchain` image (`linux/amd64`), builds, and serves a page
  that rebuilds on every save. Windows: `fm1.cmd` with Docker Desktop. `./fm1 flash` runs on the computer (macOS,
  Linux; not Windows yet: use the page).
- **Without a unit.** The page's emulator: your code compiled to WebAssembly with plain `clang --target=wasm32
  -nostdlib`, not Emscripten (`tools/build.py`). It has no USB audio input and no Bluetooth.
- **Installing.** Identity from `firmware.json` (default `FM-1_95`). The page installs, rescues, finishes a cut-off
  install, and puts back stock V15 after checking it by hash.
- **Hardware.** UC25HQ80 since 2026-11-07 ([foma#8](https://github.com/konsumer/foma/pull/8)). No SMK-37.
- **Read.** The README's "rules of the chip" and `platform/fm1.h`.

### fm1-nes

- **Status.** Dormant: no commit since 2026-10-09, and the author's own pull requests #9 and #11 (2026-10-02) are still
  open on 2026-11-26. The README calls it experimental.
- **Licence.** The project's own code is Apache-2.0. The USB audio and serial parts were taken from
  [Keitark/fm1-mdx](https://github.com/Keitark/fm1-mdx) (GPL-3.0), so the serial-only and composite firmware binaries
  carry GPLv3 terms as a whole ([THIRD_PARTY.md](https://github.com/Keitark/fm1-nes/blob/90d41c5/THIRD_PARTY.md)).
- **Inside.** Built inside JieLi's AC79 SDK (pinned commit `e30b1ee`), with the SDK's kernel and drivers. The NES player
  is an example that uses the screen, audio, keys and knobs together. With the composite USB build, a computer plays
  through the FM-1 and a serial console shares the cable (`USB_AUDIO.md`).
- **Build.** The tested host is Windows (Python 3.11, Git, CMake, the JieLi toolchain) (`GETTING_STARTED.md`).
- **Installing. [HW]** App-only updates: a `.fm1app` holds only the app, written through download mode with a patched
  jl-uboot-tool. Only the changed sectors are written, and the bootloader is kept. It accepts only reviewed stock
  layouts (010 and V14), and it checks for the Puya flash ID `0x856014`, so it refuses UC25HQ80 units. Tested on the
  maintainer's V14 unit on 2026-10-03: 51 sectors changed ([APP_UPDATES.md](https://github.com/Keitark/fm1-nes/blob/90d41c5/APP_UPDATES.md), README).
- **Read.** `APP_UPDATES.md`, `VALIDATION.md` (what was and wasn't tested), `PROVENANCE.md`, `USB_AUDIO.md`.

### FM1 Open and smk37_open (M-VAVE)

- **Status.** FM1 Open V14 ([4c7496d](https://github.com/m-vave/fm1_open/commit/4c7496d), 2026-10-21, the last push).
  M-VAVE's staff answered issues and pull requests within days up to
  [#11](https://github.com/m-vave/fm1_open/issues/11) (2026-10-24). Issues and pull requests #12 to #24 have no reply
  from them as of 2026-11-26; among them the Docker build for macOS and Linux
  ([#17](https://github.com/m-vave/fm1_open/pull/17), open since 2026-11-04). smk37_open V01 (2026-10-24): its four
  issues and pull requests have no reply.
- **Licence.** The app is GPLv3 with an extra permission (GPL section 7) to link the binary engine
  `sdk/lib/libfm1_engine.a`. The engine may be copied and redistributed, commercial use included, only for M-VAVE FM1
  devices. The boot and update files and the stock V15 recovery image in the kit are Apache-2.0
  ([LICENSE.md](https://github.com/m-vave/fm1_open/blob/4c7496d/LICENSE.md),
  [THIRD_PARTY.md](https://github.com/m-vave/fm1_open/blob/4c7496d/THIRD_PARTY.md)). This exception came after
  Charles Vestal raised the GPL question; M-VAVE took the code offline for a few days, and V12 "addresses my concerns"
  ([fm1_open#1](https://github.com/m-vave/fm1_open/issues/1), 2026-10-14 to 10-16). THIRD_PARTY.md says the engine
  library contains FreeRTOS and micro-ecc.
- **Inside. [CODE]** An RTOS app on M-VAVE's binary SDK. The SDK renders blocks of **128 frames**, stereo Q23
  (`sdk/include/fm1_block.h`), in the audio interrupt; the app's code runs in a control task, a MIDI routing task and a
  Bluetooth task (`sdk/include/fm1.h`). Audio code goes in RAM (`.volatile_ram_code`). The nominal rate is 44,100 Hz;
  the board file sets the hardware rate to 44,118 Hz (`app/board/fm1_audio.h`). 8 tracks, 64 steps, 8 voices; engines
  DX7, 303, 808, PSG, Pluck and Sample (`app/instrument.h`). MIDI in from DIN, USB and BLE; out to USB and BLE (README).
- **Build.** Windows x64 only: PowerShell and JieLi's Windows toolchain 2.5.3, with 22 toolchain files checked by hash
  (`sdk/manifest.json`). A Linux build pull request was closed (#14); the Docker one (#17) is open.
- **Without a unit.** The [workbench](https://open.m-vave.com/fm1-workbench): update, preview the instrument in the
  browser, manage projects, samples and DX7 voices, and download "Source + SDK". The browser build is made from the same
  C sources with WASI SDK 27 (README).
- **Installing.** Identity `FM-1_80000NN` (V14: `FM-1_8000014`, `app/version.h`). The workbench updates over Web MIDI
  (`sdk/WebUpdate.js`) and can go back to stock V15. M-VAVE's M-UPGRADE 3.11 does not find FM-1s that run open firmware
  ([fm1_open#20](https://github.com/m-vave/fm1_open/issues/20), Charles Vestal, 2026-11-05).
- **Hardware.** The FM-1 only; the SMK-37 Pro and Elite have their own kit, smk37_open, with the same app model and
  separate builds per model. The kit says nothing about the UC25HQ80 flash. **[?]**
- **Known limit.** On smk37_open, the control task runs every 10 ms, so MIDI out from the sequencer has 10–14 ms of
  jitter ([smk37_open#4](https://github.com/m-vave/smk37_open/issues/4), SeattleUser0, 2026-11-10). Whether FM1 Open's SDK
  does the same was not checked. **[?]**
- **Read.** README, LICENSE.md, THIRD_PARTY.md, the headers in `sdk/include/`.

### Optimist

- **Status.** Release 0.4 (2026-11-23); the tree is at 0.4.1. Since 0.4.1 it is "based on fm1-base upstream main
  (2026-11-22)" ([docs/RELEASE-0.4.1.md](https://github.com/w0ts/optimist/blob/b32f7321/docs/RELEASE-0.4.1.md)). Its
  `fm1base` submodule is pinned to Optimist's fork's integration branch: fm1-base's main plus Optimist's topic branches
  that wait as pull requests upstream (hdavid, [optimist#295](https://github.com/w0ts/optimist/issues/295), 2026-11-27).
- **Licence.** GPL-3.0-only. Its Bluetooth stack and WiFi code are `GPL-3.0-only OR MPL-2.0`, which is how fm1-base and
  foma can use them; the Ableton Link port stays GPL-3.0-only (README; confirmed by hdavid, [optimist#295](https://github.com/w0ts/optimist/issues/295)).
- **Inside.** A modular firmware: a browser "builder" picks engines, effects and pages, and builds them. It credits
  SLOOP, Felucca, Melodee, Clipper, X0X and others for its parts (README).
- **Build.** macOS arm64 and Linux x86-64, natively, with the open pi32v2-llvm compiler instead of JieLi's clang
  ([BUILDING.md](https://github.com/w0ts/optimist/blob/b32f7321/BUILDING.md)). Or a cloud build: the site opens a filled-in
  GitHub issue, and a workflow answers with the package (`docs/WEB.md`).
- **Without a unit.** A web simulator (Emscripten) and a full-system emulator (`python tools/optimist.py emu`).
- **Installing.** Release identity `FM-1_709000401` (0.4.1). Its installers keep the bootloader that is there and
  install m-bootloader only when asked ([Open Questions](Open-Questions.md#installing) 23). Which m-bootloader: the
  0.4.1 notes say 3.0, while the main branch on 2026-11-27 still pins m-bootloader v2.4 (`395e6cc`) and M-Flasher
  0.9.2 (`third_party/`, `docs/INSTALL-PATHS.md`). `docs/INSTALL-PATHS.md` has a table of which way each installer
  takes, by running firmware and package.
- **Hardware.** UC25HQ80 since 0.4 ([optimist#261](https://github.com/w0ts/optimist/issues/261)).
- **Read.** `docs/INSTALL-PATHS.md`, `docs/BLE-HW-FACTS.md`, `docs/WIFI-HW-FACTS.md`, `docs/MEMORY-MAP.md`,
  `docs/DUAL-CORE.md`, `docs/SD-CARD-DESIGN.md`.

## Things to weigh

**Licence.** Felucca's code is GPL-3.0-only. A Felucca fork stays GPL and cannot relicense by using felucca-bsp
([felucca-bsp README](https://github.com/hugelton/felucca-bsp)). MPL-2.0 files (felucca-bsp, fm1-base) can be part
of a GPL work and stay MPL file by file, so a GPL firmware may use all of fm1-base, its own Bluetooth and WiFi stacks
included. To link JieLi's closed Bluetooth or WiFi libraries, a GPL firmware needs a GPL-3.0 section 7 permission from
every copyright holder of its GPL code; a Felucca fork can't get one, since Felucca's code is not only its author's
(fm1-base [LICENSING.md](https://github.com/charlesvestal/fm1-base/blob/e4863fa/LICENSING.md), 2026-11-27).
FM1 Open's app links a binary engine under a linking exception, and that engine may only be used on M-VAVE FM1 devices.
Read [License](License.md) and each project's LICENSE before you mix code.

**Shared fixes.** A private copy of Felucca's platform stops getting fixes the day you copy it, and Felucca itself is no
longer actively maintained (2026-11-24). Since early October 2026 the shared base gained, among others: the stock
clock set at start-up, M-VAVE's supply voltages, an "UPDATING / DO NOT UNPLUG" screen in the update loader, installer
handling of FM1 Open, the UC25HQ80 flash, and the fast install. If you keep a copy, compare it with felucca-bsp now and
then. fm1-base ships a Claude skill, `felucca-to-fm1-base`, that moves a Felucca-based firmware onto it and keeps the
firmware's GPL.

**Newer units.** Two changes in newer FM-1s break old copies of the platform: the UC25HQ80 flash (saves stay in RAM,
updates are refused) and a screen that needs M-VAVE's full panel set-up. Check that your base handles both (profiles
above; [Dinghy's note](https://github.com/zvenson/dinghy/blob/14cda7a/docs/NEWER-FM1-SCREEN.md)).

**Simulator.** fm1-base, Felucca-family trees, Dinghy, foma, FM1 Open and others build the same app for the computer,
so you can test without flashing. This matters more than it seems: see [Simulators and Emulators](Simulators-and-Emulators.md).

**RTOS or bare metal.** Felucca, Dinghy, foma (on its default platform) and fm1-base are bare metal: the audio renders in
an interrupt and the UI runs in a main loop. fm1-nes, FM1 Open and foma's `mvave` platform run on JieLi's or M-VAVE's
SDK (a FreeRTOS-derived kernel, per [FM-1-RE](https://github.com/AL-255/FM-1-RE); FM1 Open's THIRD_PARTY.md names
FreeRTOS). Opinions differ on what is better for a synth: in the thread, one developer recommended FreeRTOS on both
cores, another went without it because of its overhead next to heavy DSP. Both have shipped firmware.

**Audio block size.** Felucca renders 128 frames (2.9 ms), Dinghy and foma 256 (5.8 ms), fm1-base lets the app choose,
FM1 Open's SDK uses 128. A smaller block gives less latency and less time per render. Code ported between bases must
follow the new size: Dinghy's USB audio issue (#13) came from exactly this.

**Build system.** The thread's author recommends CMake (with or without Ninja) over a custom `build.py`. The
Felucca family's `build.py` is shared lineage, not something to reinvent. If you start fresh, pick what you and
your contributors can maintain. Host platforms differ a lot: FM1 Open needs Windows, foma needs Docker, the
JieLi-toolchain bases need Linux x86-64 or a container, and Optimist's open compiler runs natively on Apple silicon.

**The SMK-37.** If you want your firmware on the SMK-37 too, see [SMK-37](SMK-37.md) before you design your UI. Of the
bases here, Dinghy builds for it, and M-VAVE has a separate kit (smk37_open). felucca-bsp's SMK-37 board is still a pull
request.

## Sources

All checked on 2026-11-26.

- [Felucca](https://github.com/hugelton/Felucca) at `ac8cc0e`: `README.md`, `BUILDING.md`, `LICENSING.md`,
  `firmware/src/core.h`, `audio.c`, `main.c`; commits [727f272](https://github.com/hugelton/Felucca/commit/727f272),
  [f4cbb6a](https://github.com/hugelton/Felucca/commit/f4cbb6a), [5044cce](https://github.com/hugelton/Felucca/commit/5044cce);
  [release 1.8.3](https://github.com/hugelton/Felucca/releases/tag/v1.8.3); [Felucca#320](https://github.com/hugelton/Felucca/issues/320).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) at `fdd709f`: `README.md`, `docs/EXPERIMENTAL.md`,
  `chip/fm1_flash.h`, `loader/loader.c`; pull requests [#26](https://github.com/hugelton/felucca-bsp/pull/26),
  [#30](https://github.com/hugelton/felucca-bsp/pull/30), [#46](https://github.com/hugelton/felucca-bsp/pull/46),
  [#51](https://github.com/hugelton/felucca-bsp/pull/51); commit [628cbd0](https://github.com/hugelton/felucca-bsp/commit/628cbd0).
- [fm1-base](https://github.com/charlesvestal/fm1-base) at `3039770`: see [fm1-base as a Base](FM1-Base.md).
- [Dinghy](https://github.com/zvenson/dinghy) at `14cda7a`: `README.md`, `CHANGELOG.md`, `LICENSING.md`, `docs/API.md`,
  `docs/NEWER-FM1-SCREEN.md`, `firmware/src/platform.c`; [dinghy#4](https://github.com/zvenson/dinghy/pull/4),
  [dinghy#13](https://github.com/zvenson/dinghy/issues/13).
- [foma](https://github.com/konsumer/foma) at `885100f`: `README.md`, `LICENSING.md`, `tools/build.py`, `platform/fm1.c`;
  [foma#8](https://github.com/konsumer/foma/pull/8).
- [fm1-nes](https://github.com/Keitark/fm1-nes) at `90d41c5`: `README.md`, `THIRD_PARTY.md`, `APP_UPDATES.md`,
  `GETTING_STARTED.md`.
- [FM1 Open](https://github.com/m-vave/fm1_open) at `4c7496d`: `README.md`, `LICENSE.md`, `THIRD_PARTY.md`,
  `sdk/include/fm1.h`, `fm1_block.h`, `app/instrument.h`, `app/board/fm1_audio.h`, `app/version.h`; its issues and pull
  requests #1 to #24. [smk37_open](https://github.com/m-vave/smk37_open) at `b2d6bcb`: `README.md`, issues #1 to #4.
- [Optimist](https://github.com/w0ts/optimist) at `b32f7321`: `README.md`, `BUILDING.md`, `docs/RELEASE-0.4.1.md`,
  `docs/RELEASE-0.4.md`, `docs/WEB.md`.
- [The r/MVaveFM1 thread](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/).
- [fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99): Charles Vestal's answers to this wiki (2026-11-27); fm1-base LICENSING.md, DESIGN.md at [e4863fa](https://github.com/charlesvestal/fm1-base/commit/e4863fa).
- [optimist#295](https://github.com/w0ts/optimist/issues/295): hdavid's answers to this wiki (2026-11-27); Optimist [`docs/INSTALL-PATHS.md`](https://github.com/w0ts/optimist/blob/bfa17d5e/docs/INSTALL-PATHS.md) (2026-11-26).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
