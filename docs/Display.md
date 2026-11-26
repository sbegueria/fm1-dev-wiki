---
title: "Display"
parent: Hardware
nav_order: 6
---

# Display

| | | Tag |
| --- | --- | --- |
| Panel | 240×240, ST7789-class controller, RGB565, high byte first, origin (0,0), no row offset | [HW] |
| Bus | SPI1 (registers at `0x11D00`), write only; CLK PC9, MOSI PC10, CS PC7 (active low), D/C PC8 | [HW] |
| Backlight | PA2, active low. PA2 is also the key matrix 74HC595's /OE ([Pinout](Pinout.md)). | [HW] [CODE] |
| Shared bus | A microSD mod can share SPI1 with the panel ([Pinout](Pinout.md)). | [CODE] [HW] |
| Tearing line (TE) | **None** on the FM-1 | [CODE] |

## Commands

The usual ST7789 set: `2A` column window and `2B` row window (two big-endian 16-bit inclusive endpoints each),
`2C` memory write, then the pixels. m-bootloader's FM-1 start-up sequence: `01` reset (wait 150 ms), `11` sleep
out (wait 120 ms), `3A 55` (16-bit pixels), `36 00`, `21` (inversion on), `13`, clear, `29` display on, backlight
on (m-bootloader SPEC §8.4). **[HW]**

### The newer FM-1s' panel needs the full set-up

**[HW]** Newer FM-1s (the FM-1 / M-VAVE logo under the screen, most with the UC25HQ80 flash, `B36014`) show
**washed-out colours, no reds, or a blank screen** with firmwares that send the ST7789V only a reset, the pixel
format (`3A`), the scan direction (`36`) and inversion (`21`). Everything else then stays at the panel's power-on
defaults, which suit the first FM-1s' panels but not the newer ones. The fix is M-VAVE's own register values, in
M-VAVE's order, with the SPI clock kept at **12 MHz**. The first FM-1s look the same with it. Found and measured by
Jake Handy for h1 0.3.1; written up for every firmware in Dinghy's
[`docs/NEWER-FM1-SCREEN.md`](https://github.com/zvenson/dinghy/blob/14cda7a/docs/NEWER-FM1-SCREEN.md)
(2026-11-26; in Dinghy since 0.6.1).

| Command | Data | Meaning |
| --- | --- | --- |
| `01` | wait 150 ms | software reset |
| `28` | wait 100 ms | display off |
| `11` | wait 120 ms | sleep out |
| `B2` | `0C 0C 0C 00 33` | porch |
| `B7` | `56` | gate control |
| `BB` | `18` | VCOM |
| `C0` | `2C` | LCM control |
| `C2` | `01` | VDV/VRH enable |
| `C3` | `1F` | VRH |
| `C4` | `20` | VDV |
| `C6` | `0F` | frame rate, 60 Hz |
| `D0` | `A6 A1` | power control 1 |
| `E0`, `E1` | `D0 0D 14 0B 0B 07 3A 44 50 08 13 13 2D 32` (both) | positive and negative gamma |
| `36` | `00` | top-left origin, RGB |
| `3A` | `55` | RGB565 |
| `E7` | `00` | one data line |
| `51` | `FF` | brightness, full |
| `21` | | inversion on (IPS) |
| `13` | | normal display |
| clear, then `29` | | display on |

**[CODE]** These are the values in M-VAVE's own FM1 Open source (`app/board/lcd.inc`, `display_setup[]`), which runs
the panel at `baud` 4 (12 MHz). M-VAVE's list also sets the window (`2A`, `2B`) and sends `20` (inversion off) early,
before the final `21`; it has no `01` or `13`. fm1-base sends M-VAVE's set-up with `lcd_panel = true` in `fm1.toml`
(opt-in, [1677a14](https://github.com/charlesvestal/fm1-base/commit/1677a14), 2026-10-28).

## SPI speed

**[CODE] [HW]** The SPI clock is the peripheral clock (LSB, 60 MHz at stock) divided by `baud + 1`:

| `baud` | Clock | Seen |
| --- | --- | --- |
| 4 | 12 MHz | Felucca's original, felucca-bsp's `fm1_lcd.h` default, M-VAVE's FM1 Open, m-bootloader. A 240×72 strip took 23 ms, longer than the panel's 16.7 ms refresh: visible tearing. |
| 0 | 60 MHz nominal (~44 MHz effective, measured) | Used by several firmwares (fm1-base among them). ST7789 parts are rated to about 62 MHz. **Not for the newer panels:** Dinghy's note says they need 12 MHz, "at least on the `0xB36014` units" (above). |

The LSB changes with the system clock, and every SPI speed with it **[HW]**: 60 MHz at 360 MHz; **53.3 MHz** at the
320 MHz that M-VAVE's 016 (V16) bootloader hands over, 11% slower (m-bootloader
[`docs/STOCK-V15-V16.md`](https://github.com/jvitkauskas/m-bootloader/blob/121cfbe/docs/STOCK-V15-V16.md),
2026-11-21); 66 MHz at 396 MHz (felucca-bsp `fm1_sd_hw.h`). This page used to give 320 MHz as m-bootloader's
clock; m-bootloader's SPEC (§3.2) sets 360 MHz, and a boot probe under m-bootloader 2.3 read 360 MHz (same file).
See [Clocks and Power](Clocks-and-Power.md).

## Drawing without tearing (as far as anyone got)

With no TE line, you can't sync to the panel's refresh in hardware. Notes from Felucca-family firmware:
- Draw into a RAM canvas in horizontal strips (e.g. 240×72), and send each strip by DMA while you draw the next.
  DMA source buffers must be in RAM, never in flash.
- Leave the last transfer running and wait for it only when you next touch the LCD or that buffer.
- Reading the panel's scanline (`RDSCL`) over the data wire is possible, but timing writes to it did **not** stop
  the tearing on hardware (a comment in Felucca-derived `lcd.c`). **[CODE]**
- Never start a DMA with a count of zero: it never completes. **[HW] [CODE]**

## Turning the backlight off

**[HW] [?]** PA2 is both the backlight and the key matrix's enable, so switching PA2 off to darken the screen also
stops the keys. kmplngj's Felucca fork reviewed the known firmwares and found none that turns the backlight off
physically while keeping the keys working (2026-10-22). An experiment sending the ST7789 brightness commands
`0x51`/`0x53` was reported with **wrong colours and unreadable text** right after install, and was rolled back;
M-VAVE's own source sets `0x51 = 255` with PA2 on, which is a lead, not proof that `0x51 = 0` darkens this
panel's backlight (`docs/display-power-research-2026-10-22.md`, `docs/lcd-backlight-probe.md`). That fork's
"all dark" mode blanks the image and the LEDs; the backlight stays powered.

## The screen and the audio share the CPU

On a single-core firmware, drawing runs between audio interrupts. Big clears, fonts read from flash and
full-screen redraws can push the audio code out of the instruction cache, so a heavy screen can cause audio
peaks. Measure the audio load with your busiest screen showing. See [Performance](Performance.md).

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §2.2, §3.2, §8.4;
  [`docs/STOCK-V15-V16.md`](https://github.com/jvitkauskas/m-bootloader/blob/121cfbe/docs/STOCK-V15-V16.md).
- Felucca's `hal/fm1_lcd_hw.h` and `src/lcd.c` (GPL-3.0-only), with the SPI speed notes from DUB-1's copy. Since
  Felucca 1.7.0, `fm1_lcd_hw.h` is in the [felucca-bsp](https://github.com/hugelton/felucca-bsp) submodule
  (`boards/fm1/fm1_lcd_hw.h`, MPL-2.0), beside the newer panel driver `boards/fm1/fm1_lcd.h` (`docs/BOARD_IO.md`);
  `lcd.c` is now `firmware/src/lcd.c`.
- [Dinghy `docs/NEWER-FM1-SCREEN.md`](https://github.com/zvenson/dinghy/blob/14cda7a/docs/NEWER-FM1-SCREEN.md) and
  `CHANGELOG.md` 0.6.1 (GPL-3.0); credit to Jake Handy ([jakehandy/h1](https://github.com/jakehandy/h1)).
- [M-VAVE FM1 Open](https://github.com/m-vave/fm1_open) `app/board/lcd.inc` at `4c7496d` (GPL-3.0 with a linking exception).
- [fm1-base](https://github.com/charlesvestal/fm1-base) `lcd_panel` ([1677a14](https://github.com/charlesvestal/fm1-base/commit/1677a14)).
- [kmplngj/Felucca](https://github.com/kmplngj/Felucca) `docs/display-power-research-2026-10-22.md`, `docs/lcd-backlight-probe.md`, `docs/lcd-colour-ab-2026-10-22.md` (GPL-3.0-only).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
