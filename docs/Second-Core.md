---
title: "Second Core"
parent: Writing firmware
nav_order: 3
---

# Second Core

The AC791N has **two cores** (JieLi's SDK: `CPU_CORE_NUM 2`; "dual-core floating-point DSP"). Many open firmwares
use only one. Using the second can roughly double the DSP headroom, but it brings a hardware problem some units
have, and new kinds of bugs.

## What is established

From felucca-bsp's [EXPERIMENTAL.md](https://github.com/hugelton/felucca-bsp/blob/main/docs/EXPERIMENTAL.md)
(October 2026):

- **[HW]** Melodee's second-core code (keremimo; MPL-2.0 by agreement) runs on the device in Melodee and X0X, and
  is used by NuSaw, SPH-16 and LOOM-8. Its model: **a worker that runs one job at a time for core 0**
  (`fm1_cpu1_run`, `fm1_cpu1_wait`).
- **[HW]** Core 1 is started at power-on, **before the audio and the timers**, with the PC limits open while it
  boots through the ROM. It has its own guards (a stack limit in its own bank). It can be held at a start, safely
  (`fm1_cpu1_stop`: the delayed bus flag cleared).
- **[HW]** **Some units' core 1 misreads shared RAM** (keremimo/melodee#17,
  [charlesvestal/fm1-x0x#10](https://github.com/charlesvestal/fm1-x0x/issues/10), measured by Jangada's
  maintainer). The defence used: a **check word on each job**; a misread, a fault or a late job **retires** the
  second core, and the firmware runs on one core until power-off (`fm1_cpu1_bar`). Core 1's faults are handled
  without a crash.
- **[HW]** The core supply matters: "at the boot loader's [voltages], under load (both cores, or WiFi …), some
  units read flash and SRAM wrong" (felucca-bsp `chip/fm1_sys.h`). Raise VDC14 and SYSVDD to M-VAVE's levels
  before starting core 1 ([Clocks and Power](Clocks-and-Power.md)).
- felucca-bsp's own copy of this code is **beta**: "this version not yet on a device". fm1-base turns it on with
  `cpu2 = true`.

Other firmwares that describe themselves as dual-core include SLOOP (both cores, per pi32v2-isa's table) and
GammaWave. See [Known Projects](Known-Projects.md).

## The worker's idle loop: one unit's finding

**[HW]** In kmplngj's Felucca fork, a unit repeatedly reported invalid second-core jobs (bad sequence, function
or checksum), even with an **empty** worker job and all real DSP on core 0, and with different RAM layouts. Making
the worker's idle branch execute **32 NOPs in RAM before polling the mailbox again** stopped it (41,419 dispatched
sub-blocks with no bad job, against failures after 432 to 14,064 without the pacing). One unit, an empirical fix:
the mechanism (silicon, cache, bus, compiler) is not known (kmplngj/Felucca `docs/cpu2-idle-polling.md`, issue
#554, 2026-10-22). It may be the same misread the check word catches above
([Open Questions](Open-Questions.md)). That fork's worker times a job out after 4 ms; fm1-base's after 8 ms.

**[CODE]** fm1-base wraps the worker as `plat_cpu2_*` (`cpu2 = true`): by default a failure **restarts** the FM-1
with the reason recorded and two failed starts give safe mode; with `cpu2_fallback = true` the core is given up
and the render runs on one core until power-off ([fm1-base as a Base](FM1-Base.md)).

## Design advice

- **Your firmware must sound right on one core.** Some units can't use the second one reliably, and the fallback
  will happen on someone's unit. Design what core 1 does so that losing it sheds something (a voice, an effect),
  not the whole sound.
- **One core should own the audio, through to filling the DMA buffer** (u/ZenLegume's suggestion, offered as
  speculation): it hands jobs to the other core, rather than the UI core treating the second as optional extra
  power. Jobs a block behind (e.g. the effects bus a block late) are easy to pipeline.
- **Keep the hand-off small and explicit**: a job descriptor, input and output buffers, a check word. Shared state
  touched by both cores is where bugs hide.
- **A UI on core 1** is another option: less DSP freed, but the audio core's cache no longer has the UI's code
  pushed through it between blocks.
- **Flash contention:** both cores fetching from XIP share the flash. Code that core 0 needs at a deadline should
  be in SRAM ([Performance](Performance.md)).

## How to test stability

From the r/MVaveFM1 discussion (u/Nukkebeer's question, u/ZenLegume's reply):
- Watch a **DSP-core load meter**: steady load with a little jitter is good; erratic jumps point to code fetched
  from flash.
- Long soak tests: an hour per patch or piece, the busiest screen showing, dropouts and job errors counted.
- Count the check-word failures, faults and late jobs, and show them in a debug page. On a unit that misreads,
  you want to see the fallback happen, not a crash.

## FreeRTOS or not

The thread's author suggests FreeRTOS on both cores, or FreeRTOS on one and bare-metal DSP on the other.
keremimo (Melodee) disagreed: its "overhead is too expensive to rock next to heavy synths that split their work in
two cores … I went without it. It paid off." Both approaches have shipped.

## Sources

- [felucca-bsp](https://github.com/hugelton/felucca-bsp) `docs/EXPERIMENTAL.md`, `chip/fm1_sys.h`.
- [kmplngj/Felucca `docs/cpu2-idle-polling.md`](https://github.com/kmplngj/Felucca/blob/main/docs/cpu2-idle-polling.md) (GPL-3.0-only).
- [fm1-base](https://github.com/charlesvestal/fm1-base) `include/plat.h` (the `plat_cpu2_*` group).
- [r/MVaveFM1 thread](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/).
- JieLi AC79 SDK `README.md`, `include_lib/driver/cpu/wl82/asm/cpu.h`.
- [fm1-base DESIGN.md](https://github.com/charlesvestal/fm1-base/blob/main/DESIGN.md) (which firmwares use `plat_cpu2_*`).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
