---
title: "Programming Guide"
parent: Writing firmware
nav_order: 1
---

# Programming Guide

How a bare-metal FM-1 firmware is usually put together, and the rules that keep it from crashing or dropping out.
The shape here is the Felucca family's (Felucca, X0X, AMB-1, DUB-1, fm1-base and many more). RTOS-based firmwares
(FM1 Open, fm1-nes) differ; see [Choosing a Base](Choosing-a-Base.md).

## The shape of a firmware

```
start-up ── clock, voltages, guards, vectors, storage, screen, inputs, audio, USB
   │
   ├── ALNK0 interrupt (prio 3)      every 2.9 or 5.8 ms: render one half buffer       ← the deadline
   ├── TIMER5 interrupt (prio 4)     10 kHz: key matrix column, USB poll at 2 kHz, ms counter
   └── main loop                     UI at ~60 frames/s, storage, everything slow
```

- **Audio in the interrupt.** The render runs inside the ALNK0 handler. It must finish before the DMA reaches the
  other half ([Audio Output](Audio-Output.md)).
- **The half's size differs between bases.** **[CODE]** Felucca has used **128-frame halves (2.9 ms)** since 1.0
  (`HALF_FRAMES` 256 → 128 in [727f272](https://github.com/hugelton/Felucca/commit/727f272), 2026-10-05).
  Forks taken from Felucca before 1.0 keep 256. fm1-base (`APP_BLOCK`, `include/fm1b_config.h`) and Dinghy
  (`firmware/src/platform.c`) use **256 frames (5.8 ms)**. M-VAVE's FM1 Open uses 128 (`FM1_BLOCK_FRAMES`). A
  smaller half halves the latency, and also the time any one block may take.
- **Input and USB in a faster timer** that may nest into the audio render, so key scans and USB keep their rhythm.
- **Everything else in the main loop**, which only gets the time the interrupts leave.
- **Hand-offs through lock-free rings or single-writer flags.** The main loop never waits for the audio, and the
  audio never waits for anything. A ring with one writer and one reader, and a compiler barrier on publish, is
  enough on one core.

## Rules for the audio code

1. **Measure the worst block**, not the average. A meter that shows the peak of the last ~2 s, and a dropout
   counter, are worth more than any optimisation ([Debugging](Debugging.md), [Performance](Performance.md)).
2. **No floating-point doubles.** The FPU is single precision. Write `1.0f`, use `sinf`, not `sin`. Check the
   link map for soft-double routines (`__adddf3`, `__muldf3`, …), and make the build fail if it finds one.
   *Two traditions:* Felucca, DUB-1 and FM1 Open compile with the FPU on (`-mfprev1`) and use `float`. The SLOOP /
   Dinghy line builds **integers only** (Q15 levels, Q12 gains, 32-bit phase accumulators, no 64-bit division
   since `__udivdi3` is not linked), and its docs say the chip has "no FPU". Whose statement holds for the chip is
   in [Open Questions](Open-Questions.md); for your build, what matters is which flags and libraries it links.
3. **No flash access from the audio path while the flash may be busy.** Saves erase the flash with interrupts off;
   the audio code must not run then, and the DMA buffer should be zeroed ([Flash Storage](Flash-Storage.md)).
4. **Spread bursts of work.** Changing a whole preset, regrowing patterns or loading a sample in one block makes
   a spike. Spread it over a few blocks; 30–40 ms is not heard.
5. **Compute slow things slowly.** Envelopes, LFOs and smoothing every 32, 64 or 128 frames (a "control rate").
6. **Multiply, don't divide.** Precompute `1/x` when `x` changes.
7. **Keep 64-bit integer multiplies out of line** in hot fixed-point code: see the CPU erratum in [Performance](Performance.md).
8. **Flush tiny floats.** Feedback states decaying towards zero can go subnormal. Whether this FPU slows down on
   subnormals is not established **[?]**; flushing state below ~1e-20 to 0 costs little and removes the doubt.

## Rules for the UI code

- Draw into RAM canvases and send them by DMA in strips ([Display](Display.md)).
- Expect the UI to steal cache from the audio. Measure the audio load with the busiest screen up.
- Keep the UI frame rate steady; don't redraw what didn't change.
- Encoders are relative: show a value when it changes.

## Rules for the whole firmware

- **Registers only in one place.** Felucca keeps every register access in its hardware layer: since 1.7.0 that is
  the felucca-bsp submodule (`firmware/bsp`, `chip/` and `boards/fm1/`), plus what is left in `hal/`. Its build
  refuses register addresses, volatile casts or inline assembly in the app's sources (`tools/build.py`). This keeps
  hardware facts in one reviewable place, and makes the code testable on a computer.
- **A platform interface.** If the app only talks to the hardware through a `plat.h`-style interface, the same app
  can run in a host simulator and a browser emulator ([Simulators and Emulators](Simulators-and-Emulators.md)).
- **Survive bad starts.** Boot counters in a `NOINIT` area, a safe mode after two failed starts, a crash screen
  with the PC, a watchdog ([Debugging](Debugging.md)).
- **Start the clock and voltages as M-VAVE's firmware has them** ([Clocks and Power](Clocks-and-Power.md)).
- **Be installable**: answer the update protocol and the soft key ([Update Protocol and Installers](Update-Protocol-and-Installers.md)).
  A firmware without an update path locks the unit to it until someone uses download mode. Keep a key-held
  rescue at power-on as well (OCT− in SLOOP, ORBIT and Dinghy).
- **Check divisors**: in the Felucca family a division by zero gives a wrong value, not a crash ([Debugging](Debugging.md)).
- **Don't trust the flash's contents** ([Flash Storage](Flash-Storage.md)).

## The RTOS shape: FM1 Open

**[CODE]** M-VAVE's FM1 Open (at [4c7496d](https://github.com/m-vave/fm1_open/tree/4c7496d)) runs on JieLi's RTOS,
with fixed contexts (`sdk/include/fm1*.h`): `fm1_engine_render()` runs in the I2S/DMA interrupt, once per 128-frame
block of stereo Q23; most other calls belong to a control task, and some work runs on a Bluetooth task. Audio code is
marked `FM1_AUDIO_CODE` (section `.volatile_ram_code`, in RAM) because "Audio code and every callee must be safe
during Flash programming". Its tables go in `.volatile_ram.fm1`. Flash calls come only from the control task, with
interrupts on.

## A minimal render callback (sketch)

```c
/* a sketch, not from any one project: the shape of a block render */
#define FRAMES 128u                                 /* your base's half: 128 (Felucca) or 256 (fm1-base, Dinghy) */
static void render_block(int32_t *out)              /* called from the ALNK0 handler */
{
    uint32_t t0 = fm1_ticks();                      /* TIMER4, 24 MHz */
    for (uint32_t i = 0; i < FRAMES; i++) {
        float l, r;
        synth_tick(&l, &r);                         /* your DSP, single-precision floats */
        out[2 * i]     = to_q23(l);                 /* signed Q23 in the low 24 bits, sign-extended */
        out[2 * i + 1] = to_q23(r);
    }
    load_update(fm1_ticks() - t0, FRAMES);          /* peak and mean, for the screen */
}
```

The sample format is felucca-bsp's (`boards/fm1/fm1_audio.h`, `docs/BOARD_IO.md`): "signed Q23 in int32_t
(sign-extended low 24 bits, NOT shifted left by 8)", full scale `0x007FFFFF`. **[CODE]**

## Targeting both FM-1 and SMK-37

The SMK-37 has the FM-1's chip with other panel wiring ([SMK-37](SMK-37.md)). genkigenki's
[fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) kit (CC BY-NC 4.0, at `54252ff`, 2026-10-17) describes how
its ports keep one source for both; its
[07-develop-for-both.md](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/07-develop-for-both.md) and
[08-pitfalls.md](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/08-pitfalls.md) have the detail. In
short, as practice from those ports: **[CODE]**

- **Board code in board files.** Input, screen, pad LEDs and flash map live in the board's own files. Shared code
  has few, short `#if BOARD_SMK37` blocks (identities, UI mapping, storage layout).
- **The FM-1 build stays byte-identical** with the SMK flag off: that is the test that nothing changed for FM-1
  users. Compare per area. `__DATE__` makes builds differ from day to day, so compare on the same day as the baseline.
- **Write features against what the board has**, not which board it is: a few capability constants (pads, faders,
  wheels, RGB) with an FM-1 fallback for each feature (pad colours on the key LEDs, faders as a menu page). With the
  constants at zero the compiler drops the SMK paths.
- **Build both, in order:** the SMK package, copied out; then the FM-1 build last, since both write the same
  intermediates and the tests expect FM-1 binaries; then the tests. Build the emulator or simulator too, if the
  project has one.
- **No literals for panel sizes.** Size arrays and loops by the board's button and LED counts (the SMK has 18 buttons
  to the FM-1's 14), and count time in milliseconds, not scan frames (625 a second on the SMK port, 909 on the FM-1).
- **Fix the fork's own hardware code**: power-on key combos, flash ID checks that accept only `0x856014`, boot stubs.
  On the SMK, never drive PB1, and treat PH8 and PA2 as matrix rows, not outputs.
- **Each firmware owns its data**: an owner tag or a data family, so two firmwares never read each other's records
  as their own.
- **Check the model on every install path**, fresh installs and resumes alike: never an FM-1 package on an SMK, or the
  other way.
- **Labels move**: an FM-1 firmware's on-screen "K1"–"K4" (its KNOB 1–4) are K5–K8 on the SMK.
- **Never write the SMK's calibration**, and read [SMK-37](SMK-37.md) on what stock firmware does with a blank
  USRFLASH before you put data there.

## Where next

[Performance](Performance.md) · [Debugging](Debugging.md) · [Second Core](Second-Core.md) · [Simulators and Emulators](Simulators-and-Emulators.md)

## Sources

- The main loops and HALs of Felucca-family firmwares (GPL-3.0-only): [Felucca](https://github.com/hugelton/Felucca)
  `firmware/src/core.h`, `firmware/src/audio.c`, `firmware/src/main.c`, `tools/build.py` (at ac8cc0e);
  [fm1-base DESIGN.md](https://github.com/charlesvestal/fm1-base/blob/main/DESIGN.md), `include/fm1b_config.h`.
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) `boards/fm1/fm1_audio.h`, `docs/BOARD_IO.md` (MPL-2.0).
- [FM1 Open](https://github.com/m-vave/fm1_open) `sdk/include/fm1.h`, `fm1_block.h`, `fm1_engine_control.h`.
- [Dinghy docs/RULES.md](https://github.com/zvenson/dinghy/blob/main/docs/RULES.md) (integer-only rules); [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) development skill §5, §7.
- [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) `docs/07-develop-for-both.md`, `docs/08-pitfalls.md`
  (CC BY-NC 4.0; 54252ff, 2026-10-17).
- The tips of the [r/MVaveFM1 thread](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/) by u/ZenLegume and its comments.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
