---
title: "Controls"
parent: Hardware
nav_order: 7
---

# Controls

The FM-1's panel: **27 note keys** (F3 to G5), **14 buttons**, **7 encoders**, the **MASTER** pot, and LEDs under
keys and buttons. Wiring: [Pinout](Pinout.md).

## Scanning

**[CODE] [HW]** All keys, buttons and encoders share one matrix: 11 columns selected by two 74HC595 shift
registers (bit-banged on PA4/PA3/PA1), 6 rows read with pull-ups (closed = low).

Two ways to scan, both used by Felucca-family firmware:
- **Polled:** one full frame (11 columns, ~0.6 ms) from the main loop.
- **From a timer interrupt:** one column per tick at ~10 kHz, a full frame every 11 ticks. Rows are sampled one
  tick after the column was latched (it needs ~10 µs to settle). This keeps a steady rhythm even when the main loop
  is busy, and it lets the LEDs stay lit between ticks.

If your audio interrupt is long, let the scan timer interrupt nest into it (give it a higher priority), or the
scan stutters and encoders miss steps. X0X and the firmwares derived from it run TIMER5 at priority 4, above ALNK0 at 3, for this reason.
M-VAVE's FM1 Open does the same: TIMER5 at 10 kHz (IRQ 63) at priority 5, above ALNK0 (IRQ 11) at 4
(`app/board/device.c`, `fm1_audio.h`). **[CODE]**

**The scan rate.** **[CODE]** In felucca-bsp the tick rate is a build option, `FM1_INPUT_HZ`: 10,000 (the default),
5,000, 2,000 or 1,000 ticks a second (fm1-base: `input_hz` in `fm1.toml`). TIMER5's period is 6 MHz /
`FM1_INPUT_HZ`, and every time in the scan is derived from it
([e36cb7c](https://github.com/hugelton/felucca-bsp/commit/e36cb7c), Charles Vestal, 2026-11-10). The scan's GPIO
work is ~7.5% of core 0 at 10 kHz. A slower tick frees CPU but costs latency and LED refresh (from
`boards/fm1/fm1_input.h`; the knob figures come from a host test):

| Rate | Frame (LED refresh) | Press counted after | Release after | Knob, fastest kept |
| --- | --- | --- | --- | --- |
| 10,000 | 1.1 ms (909 Hz) | 1.1–2.2 ms | 8.8 ms | ~225 clicks/s |
| 5,000 | 2.2 ms (455 Hz) | 2.2–4.4 ms | 8.8 ms | ~113 |
| 2,000 | 5.5 ms (182 Hz) | 0–5.5 ms | 11 ms | ~45 |
| 1,000 | 11 ms (91 Hz) | 0–11 ms | 22 ms | ~23 |

At 2,000 and 1,000 some LED effects flicker, per the same file. With velocity on, the battery is read in the
MASTER pot's slot of the scan, every `FM1_BATT_EVERY`-th time (~0.2 s), because the main loop can't use the ADC
while the scan owns it. **[CODE]**

**How the stock firmware scans.** **[RE]** M-VAVE's **V13** (FM-1-RE
[`io/07-input.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/07-input.md) §2, §3) scans 41 key positions
(27 keys and 14 buttons make 41) on a 20 ms timer, and keeps 7 encoder slots. A press or release counts after 3 scans
in a row; a long press after 41 scans, about 0.8 s. Pressing scanner positions 0 and 1 together only resets the octave
and semitone shift, in V13 and V14; it is not a factory or update mode (FM-1-RE marks this high).

## Debounce

Felucca uses an integrating debounce of 8 frames (~5 ms). **[CODE]**

felucca-bsp's `boards/fm1/fm1_input.h` (at `fdd709f`) describes an **asymmetric** debounce: a press counts after
2 closed frames in a row (1.1–2.2 ms at 10 kHz), a release after ~9 ms open (8 frames at 10 kHz). Its reasoning: the
matrix has diodes and no ghosting, so a closed sample is a closed key; the long release stops a bouncing contact
from ending a note early or playing it twice. **[CODE]**

## Encoders

**[CODE]** The encoders are plain quadrature, read through the same matrix. Felucca's decoder learns each
encoder's rest (detent) states while it sits still (~25 ms), and emits a step only on reaching a rest state after
at least two net transitions. That gives one click = one step at any speed, whether a detent is a half or a full
quadrature cycle, which differs between units. (felucca-bsp's current `fm1_input.h` says instead that "an FM-1
detent is one full quadrature cycle (4 transitions)" and keeps one rest state per knob. Both statements are kept
here: if your unit differs, please report it.)

**The encoders are relative.** A firmware can't read a knob's position after a page change, only the next steps.

## MASTER

The MASTER pot is on ADC channel 4 (PB6), 10-bit, polled from the main loop. **[CODE]**

FM-1-RE's input notes call ADC channels 3 and 4 the "pitch and mod wheels". The FM-1 has no wheels: channel 4 is
MASTER and channel 3 the battery's divider ([Pinout](Pinout.md)). Its own reading of V13's code fits that: channel 3
feeds the battery level and channel 4 an analog-change event (`io/07-input.md` §4.2). The "Pitch Up" and "Pitch Dn"
strings it found are not explained.

## LEDs

**[CODE] [HW]** LEDs share the matrix columns: four LED lines (PH6, PH9, PA9, PA10) light the LED on the selected
column. Brightness depends on how long a column stays selected. Felucca keeps an LED lit ~40 µs per column and
dims by lighting it in fewer frames.

**LEDs during a flash write. [CODE]** With interrupts off for a flash operation, the scan stops, and lit LEDs went dark
or dimmed by a third during a save. [OG Synth](https://github.com/mikejohnsonbeats/OG-Synth-source) keeps them going: the flash driver's busy loops call a step that runs
from RAM and goes on lighting the columns, without reading keys (`fm1_led_ram_step`, `FL_BUSY_STEP` in its
`patches/felucca-bsp.patch`, MPL-2.0, 2026-11).

## Velocity and pressure from the keys

**[CODE]** The keys have no velocity sensor, but [Velo](https://github.com/charlesvestal/fm1-velocity) reads
velocity and aftertouch "from the M-VAVE FM-1's keys, no hardware mods", and felucca-bsp and fm1-base include
it (off by default). How it works is documented in those repositories.

**[CODE]** In fm1-base (`lib/touch.h`, `#define APP_TOUCH`): the **20 keys on the analog rows (F3 to C5)** are read
through the scan's ADC readings, giving a velocity per press and a **pressure** (0–127) while held, which works as
per-key aftertouch. The **7 top-row keys (C#5 to G5)** have no converter: a velocity from a probe through the LED
line, and no pressure. Curve and sensitivity are adjustable (−5..+5). Confirmed by fm1-base's author (Charles Vestal,
[fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99), 2026-11-27). See [fm1-base as a Base](FM1-Base.md).

## Hold-at-power-on keys

Different bootloaders and firmwares claim different keys at power-on. Avoid clashing with them:

| Keys at power-on / held | Who | Does |
| --- | --- | --- |
| REC held at power-on | m-bootloader | ROM download mode |
| PLAY held at power-on | m-bootloader | recovery menu |
| OCT− + OCT+ | Felucca, SLOOP | calibration (at start) |
| OCT− + OCT+ held 5 s (while running) | Felucca and many forks | the ROM's download mode (UBOOT), RAM kept |
| OCT− alone | SLOOP, Snoop37, Dinghy | USB rescue (Dinghy: see below) |
| REC held at power-on | fm1-base | the ROM's download mode (UBOOT; `enter_uboot`), though the screen says "UPDATE MODE (REC)" |

(m-bootloader SPEC §8.2; project READMEs.) Note the two uses of REC: under m-bootloader, the bootloader acts first.

**OCT− + OCT+ held 5 s in Felucca** enters the chip's ROM download mode, **UBOOT** (USB `4C4A:8057`), not an update
loader. A countdown shows from 2 s; letting go cancels it. Felucca stops the audio, detaches USB and jumps to UBOOT
with RAM intact (`firmware/src/main.c` at [`ac8cc0e`](https://github.com/hugelton/Felucca/blob/ac8cc0e/firmware/src/main.c),
in Felucca since 0.8-beta). **[CODE]** On a UC25HQ80 unit stuck on Felucca 1.8.1, this was the way out: the unit
showed up as `4c4a:8057` "WL80UBOOT1.00", and the whole flash was dumped twice with jl-uboot-tool, the dumps
matching. On Linux the UBOOT disk reports no medium; open it with `O_NONBLOCK`
(ronbarrantes, [hugelton/Felucca#320](https://github.com/hugelton/Felucca/issues/320), 2026-11-09). **[HW]**

**Dinghy's OCT− rescue** checks the key before anything drives PA2, so on some units it never opens (Dinghy 0.3
and 0.4 on two units; the check is unchanged in 0.6.1). Drive PA2 low before any power-on key check
([Pinout](Pinout.md)). **[HW]**

## Sources

- Felucca's `hal/fm1_input.h` (GPL-3.0-only) and its derivatives' `main_fm1.c`. Since Felucca 1.7.0 the input driver
  is felucca-bsp's [`boards/fm1/fm1_input.h`](https://github.com/hugelton/felucca-bsp/blob/fdd709f/boards/fm1/fm1_input.h)
  (MPL-2.0; rate options [e36cb7c](https://github.com/hugelton/felucca-bsp/commit/e36cb7c)) and `chip/fm1_timer.h`.
- [Felucca](https://github.com/hugelton/Felucca) `firmware/src/main.c` at `ac8cc0e`; issue [#320](https://github.com/hugelton/Felucca/issues/320).
- [M-VAVE FM1 Open](https://github.com/m-vave/fm1_open) `app/board/device.c`, `app/board/fm1_audio.h` at `4c7496d`.
- [zvenson/dinghy PR #4](https://github.com/zvenson/dinghy/pull/4); Dinghy `firmware/src/platform.c` at `14cda7a`.
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §8.2.
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) `docs/io/07-input.md` §2, §3, §4.2 (WTFPL; stock V13, V14).
  Its code addresses need `0x120` added ([Memory Map](Memory-Map.md#the-stock-app-runs-at-0x02000120)).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp), [fm1-base](https://github.com/charlesvestal/fm1-base) READMEs; fm1-base `lib/touch.h` (MPL-2.0).
- [fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99): Charles Vestal's answers to this wiki (2026-11-27); fm1-base LICENSING.md, DESIGN.md at [e4863fa](https://github.com/charlesvestal/fm1-base/commit/e4863fa).
- [OG Synth](https://github.com/mikejohnsonbeats/OG-Synth-source) `patches/felucca-bsp.patch` (`chip/fm1_flash.h`, `boards/fm1/fm1_input.h`), 2.1.3 (MPL-2.0).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
