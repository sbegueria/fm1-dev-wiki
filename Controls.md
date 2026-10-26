# Controls

The FM-1's panel: **27 note keys** (F3 to G5), **14 buttons**, **7 encoders**, the **MASTER** pot, and LEDs under
keys and buttons. Wiring: [[Pinout]].

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

## Debounce

Felucca uses an integrating debounce of 8 frames (~5 ms). **[CODE]**

## Encoders

**[CODE]** The encoders are plain quadrature, read through the same matrix. Felucca's decoder learns each
encoder's rest (detent) states while it sits still (~25 ms), and emits a step only on reaching a rest state after
at least two net transitions. That gives one click = one step at any speed, whether a detent is a half or a full
quadrature cycle, which differs between units.

**The encoders are relative.** A firmware can't read a knob's position after a page change, only the next steps.

## MASTER

The MASTER pot is on ADC channel 4 (PB6), 10-bit, polled from the main loop. **[CODE]**

## LEDs

**[CODE] [HW]** LEDs share the matrix columns: four LED lines (PH6, PH9, PA9, PA10) light the LED on the selected
column. Brightness depends on how long a column stays selected. Felucca keeps an LED lit ~40 µs per column and
dims by lighting it in fewer frames.

## Velocity and pressure from the keys

**[CODE]** The keys have no velocity sensor, but [Velo](https://github.com/charlesvestal/fm1-velocity) reads
velocity and aftertouch "from the M-VAVE FM-1's keys, no hardware mods", and felucca-bsp and fm1-base include
it (off by default). How it works is documented in those repositories.

## Hold-at-power-on keys

Different bootloaders and firmwares claim different keys at power-on. Avoid clashing with them:

| Keys at power-on / held | Who | Does |
| --- | --- | --- |
| REC held at power-on | m-bootloader | ROM download mode |
| PLAY held at power-on | m-bootloader | recovery menu |
| OCT− + OCT+ | Felucca, SLOOP | calibration (at start) |
| OCT− + OCT+ held 5 s | Felucca and many forks | update mode |
| OCT− alone | SLOOP, Snoop37 | USB rescue |
| REC held at power-on | fm1-base | update mode |

(m-bootloader SPEC §8.2; project READMEs.) Note the two uses of REC: under m-bootloader, the bootloader acts first.

## Sources

- Felucca's `hal/fm1_input.h` (GPL-3.0-only) and its derivatives' `main_fm1.c`.
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §8.2.
- [felucca-bsp](https://github.com/hugelton/felucca-bsp), [fm1-base](https://github.com/charlesvestal/fm1-base) READMEs.
