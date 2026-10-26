# Display

| | | Tag |
| --- | --- | --- |
| Panel | 240×240, ST7789-class controller, RGB565, high byte first, origin (0,0), no row offset | [HW] |
| Bus | SPI1 (registers at `0x11D00`), write only; CLK PC9, MOSI PC10, CS PC7 (active low), D/C PC8 | [HW] |
| Backlight | PA2, active low. PA2 also enables the key matrix ([[Pinout]]). | [HW] |
| Tearing line (TE) | **None** on the FM-1 | [CODE] |

## Commands

The usual ST7789 set: `2A` column window and `2B` row window (two big-endian 16-bit inclusive endpoints each),
`2C` memory write, then the pixels. m-bootloader's FM-1 start-up sequence: `01` reset (wait 150 ms), `11` sleep
out (wait 120 ms), `3A 55` (16-bit pixels), `36 00`, `21` (inversion on), `13`, clear, `29` display on, backlight
on (m-bootloader SPEC §8.4). **[HW]**

## SPI speed

**[CODE] [HW]** The SPI clock is the peripheral clock (LSB, 60 MHz at stock) divided by `baud + 1`:

| `baud` | Clock | Seen |
| --- | --- | --- |
| 4 | 12 MHz | Felucca's original. A 240×72 strip took 23 ms, longer than the panel's 16.7 ms refresh: visible tearing. |
| 0 | 60 MHz nominal (~44 MHz effective, measured) | Used by several firmwares. ST7789 parts are rated to about 62 MHz. |

Under m-bootloader's 320 MHz clock the LSB drops to 53 MHz, so every SPI speed drops by 11% unless you reset the
clock ([[Clocks and Power]]).

## Drawing without tearing (as far as anyone got)

With no TE line, you can't sync to the panel's refresh in hardware. Notes from Felucca-family firmware:
- Draw into a RAM canvas in horizontal strips (e.g. 240×72), and send each strip by DMA while you draw the next.
  DMA source buffers must be in RAM, never in flash.
- Leave the last transfer running and wait for it only when you next touch the LCD or that buffer.
- Reading the panel's scanline (`RDSCL`) over the data wire is possible, but timing writes to it did **not** stop
  the tearing on hardware (a comment in Felucca-derived `lcd.c`). **[CODE]**
- Never start a DMA with a count of zero: it never completes. **[HW] [CODE]**

## The screen and the audio share the CPU

On a single-core firmware, drawing runs between audio interrupts. Big clears, fonts read from flash and
full-screen redraws can push the audio code out of the instruction cache, so a heavy screen can cause audio
peaks. Measure the audio load with your busiest screen showing. See [[Performance]].

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §2.2, §8.4.
- Felucca's `hal/fm1_lcd_hw.h` and `src/lcd.c` (GPL-3.0-only), with the SPI speed notes from DUB-1's copy.
