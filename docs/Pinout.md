---
title: "Pinout"
parent: Hardware
nav_order: 4
---

# Pinout

What each pin of the AC791N does on the FM-1, from working drivers. "Port" registers are at
`0x50000 + 0x40 × port` (A=0, B=1, C=2, H=7) with `OUT +0`, `IN +4`, `DIR +8` (1 = input), `DIE +0xC` (digital
input enable), `PU +0x10`, `PD +0x14`, `HD0 +0x18`, `HD +0x1C`. **[SDK] [CODE]**

SMK-37 wiring differs: see [SMK-37](SMK-37.md).

| Pin | Function | Notes | Tag |
| --- | --- | --- | --- |
| PA0 | Matrix row (encoders) | pull-up, low = closed | [CODE] |
| PA1 | 74HC595 latch (RCLK) | | [CODE] [HW] |
| PA2 | LCD backlight, active low | **also the key matrix 74HC595's /OE** (below) | [HW] [CODE] |
| PA3 | 74HC595 clock (SRCLK) | | [CODE] [HW] |
| PA4 | 74HC595 data (SER) | | [CODE] [HW] |
| PA5–PA8 | Matrix rows | pull-ups; low = closed | [CODE] [HW] |
| PA9, PA10 | LED lines | light the LED in the selected column | [CODE] [HW] |
| PB1 | ADC ch 3: battery divider | never drive low (resets the chip after 8 s) | [CODE] |
| PB5 | Headphone jack switch, input | low with headphones in, high with nothing plugged; bounces ~150 ms on a plug (below) | [HW] |
| PB6 | ADC ch 4: MASTER pot | | [CODE] |
| PB7 | Matrix row (encoder 6) | | [CODE] |
| PC0, PC1, PC2 | I2S MCLK, BCLK, LRCK (ALNK0) | felucca-bsp and M-VAVE's FM1 Open agree | [CODE] |
| PC3 | ALNK0 clock / RX | FM1 Open only: "RX supplies the interrupt clock"; no analogue input connected | [CODE] |
| PC6 | I2S data out | felucca-bsp: "not a codec reset or mute pin"; FM1 Open: "DAC" ([Audio Output](Audio-Output.md)) | [CODE] |
| PC7 | LCD CS, active low | SPI1; also the microSD mod's CS, inverted (below) | [CODE] [HW] |
| PC8 | LCD D/C (0 command, 1 data) | the microSD mod's data in (card DO) during a card session | [CODE] [HW] |
| PC9 | LCD SPI clock | | [CODE] [HW] |
| PC10 | LCD SPI data out | | [CODE] [HW] |
| PH6, PH9 | LED lines | | [CODE] [HW] |
| PH8 | TRS MIDI in → UART1 RX | 31250 baud, through input channel 1 | [CODE] |
| USB D+/D− | USB0 full-speed device | also the USB_KEY recovery signal ([Recovery](Recovery.md)) | [HW] |

## The package pins, from the resource hub

The [FM-1 resource hub](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) (§1.2) maps
every pin of the 48-pin package, from Felucca-family drivers, board photos, JieLi's AC7911B datasheet and
reference schematics. Where it goes beyond the drivers, it is **[?]** here until someone checks it on a board:

| Pin(s) | The hub says | Tag |
| --- | --- | --- |
| PC0, PC1, PC2, PC6 | I2S MCLK, BCLK, LRCK, and the codec's power/reset line (active high), to an external codec. **The PC6 part disagrees** with felucca-bsp and FM1 Open, which both make PC6 the data line (table above). | [?] |
| DACL (pin 13) | the chip's own DAC, left channel, as an auxiliary line next to the codec | [?] |
| PD0, PD1, PD4, PD5, PD6 | the internal flash: CS, DI, CLK, DO, and a flash power gate | [?] |
| PH8 / PH7 | share one package pin; PH7 is UART1 TX on the die, but the TRS jack's input circuit only passes MIDI in | [?] |
| 74HC595 outputs | 16 outputs, 11 used as columns: **5 spare** outputs on the second register | [CODE] [?] |
| All 48 pins | in use: no free GPIO pad to solder to | [?] |

The hub also suggests ways to get a MIDI *out* (routing UART1 TX to the USB D+ pin when no computer is attached,
or to a freed pin through the output crossbar). **[?]** Since 2026-11-18 the hub's `HARDWARE_MODS.md` lists a
"Hardware DIN MIDI Out Mod (by ehn96)": a 3.3 V DIN-5 MIDI out, "verified with Dinghy v0.3.0 (which outputs hardware
MIDI out on pin 39, recommends 74LVC1G07 buffer)". The linked repository (`ehn96/dinghy`) returned 404 on
2026-11-26, and Dinghy's own source (0.6.1) has no MIDI out on a pin. Unchecked here. **[?]**

## The key matrix and PA2

**[HW]** "PA2 MUST be driven low while the FM-1 matrix is scanned." On the FM-1, PA2 (the backlight enable) also
enables the key matrix: with PA2 high or left as an input, no key reads closed. The ROM leaves PA2 undriven, so a
key check at start-up sees nothing until PA2 is driven low (m-bootloader SPEC §8.2). A bootloader menu or a
"hold a key at power-on" feature must drive PA2 low first; this also turns the backlight on.

**[CODE]** The reason, per a comment in Felucca's `firmware/src/lcd.c`: "PA2 is also the key matrix 595's /OE, so
with it high no button, key or knob reads". **[HW]** Dinghy 0.3 and 0.4 check OCT− for their USB rescue before
anything drives PA2, so on some units the rescue never opens: 0 closed reads of any key in 82 scan frames, and the
rescue opened once `fm1_lcd_hw_init()` ran before the check (sgarrettnelson, 2026-10-17; KevosG, 2026-11-06, on a
second unit; [zvenson/dinghy PR #4](https://github.com/zvenson/dinghy/pull/4), still open on 2026-11-26, and Dinghy
0.6.1's `fm1_cstart()` still checks first).

## Matrix layout

**[CODE]** 11 columns selected through the shift registers (a column's bit low = selected; idle word `FFFF`),
6 rows read on PA0, PA5, PA6, PA7, PA8 and PB7. Felucca's key map (then `hal/fm1_input.h`, GPL-3.0-only; the same
table is now in felucca-bsp's `boards/fm1/fm1_input.h`, MPL-2.0):

```c
/* key id at (physical column, packed row bit), -1 = none;
 * ids 0..13 buttons, 14..40 note keys (note 0 = F3 .. 26 = G5) */
static const int8_t FM1_KEYMAP[6][11] = {
    {-1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1},          /* PA0: encoders */
    { 5, 11,  4, 10,  3,  9,  2,  8, -1, -1, -1},          /* PA5 */
    {34, 35, 36, 37, 38, 40, 39, 13,  7,  6, 12},          /* PA6 */
    {23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33},          /* PA7 */
    { 0,  1, 15, 14, 17, 16, 19, 18, 20, 21, 22},          /* PA8 */
    {-1, -1, -1, -1, -1, -1, -1, -1, -1, -1, -1},          /* PB7: encoder 6 */
};
```

Button 0 is OCT−, button 1 is OCT+. The other button ids map to the panel's labelled buttons in each firmware's
own table. For the bootloader keys, m-bootloader reads OCT− on PA8 with column word `F7FE`, OCT+ on PA8 with
`EFFD`, PLAY on PA6 with `FBFF`, REC on PA6 with `FF7F` **[HW]**.

Allow ~10 µs after selecting a column before reading the rows. **[CODE] [HW]**

## The headphone jack

**[HW]** The headphone jack's switch is on **PB5**, an input as the boot leaves it: **low with headphones in**, high
with nothing plugged. It bounces for ~150 ms on a plug. Read on one FM-1 with a probe of the input registers
(Henri David, 2026-10-29). felucca-bsp's `boards/fm1/fm1_jack.h` reads it and never writes the pin; its commit says
the header itself "has not run in a firmware on a device"
([7bcea16](https://github.com/hugelton/felucca-bsp/commit/7bcea16)). fm1-base offers it as `plat_headphones()` with
`headphones = true` in `fm1.toml`. **[CODE]**

## A microSD card on the screen's bus (a mod)

The FM-1 has no card slot. A mod puts a microSD card, in SPI mode, on the LCD's SPI1, tapping four of the panel's
lines. The [m-card](https://github.com/jvitkauskas/m-card) board does this. **[CODE] [HW]**

| Card pin | FM-1 pin | Notes |
| --- | --- | --- |
| CLK | PC9 (SPI1 clock) | through 470 Ω on m-card |
| DI | PC10 (SPI1 data out) | through 470 Ω, 47 kΩ pull-up |
| DO | PC8, the panel's D/C | through 220 Ω, 47 kΩ pull-up; SPI1's data input while a card session is open |
| CS | **NOT PC7**, an inverter on the panel's CS | the card is selected while the panel's CS is high, the panel while it's low |

There is no card-detect line: a firmware finds a card by asking it. A session first sends the panel a NOP, so
whatever is clocked while the card is deselected is ignored by the panel (m-bootloader SPEC §11.5; felucca-bsp
`boards/fm1/fm1_sd_hw.h`, [db8bbc3](https://github.com/hugelton/felucca-bsp/commit/db8bbc3), Julius Vitkauskas,
2026-11-15). On one FM-1 with an earlier SD board, driving PC9 hard during card sessions raised the fastest clock
that read correctly from 8.57 to 10 MHz; not measured on m-card
([c76eb34](https://github.com/hugelton/felucca-bsp/commit/c76eb34)). **[HW]**

**[HW]** The LCD's 10-pin flat-cable connector **J12**, as probed with a meter (Julius, 2026-10-27; Optimist
`docs/SD-CARD-DESIGN.md`):

| J12 pin | Net | Chip pin |
| --- | --- | --- |
| 1, 10 | GND | |
| 2 | PC8, LCD D/C | 31 |
| 3 | PC7, LCD CS | 32 |
| 4 | PC9, clock | 30 |
| 5 | PC10, data out | 29 |
| 7 | 3.3 V (a working card prototype ran from it, 2026-10-28) | |
| 6, 8, 9 | not probed | |

Pin 1 is the leg next to the board's dot. On the SMK-37 the card's pins differ: see m-bootloader SPEC §11.5.

## Sources

- Felucca's HAL (`hal/fm1_gpio.h`, `fm1_input.h`, `fm1_lcd_hw.h`, `fm1_adc.h`, `fm1_audio.h`, `fm1_uart.h`;
  GPL-3.0-only, © Leo Kuroshita), as in DUB-1. Since Felucca 1.7.0 (2026-10-28) these live in the
  [felucca-bsp](https://github.com/hugelton/felucca-bsp) submodule (MPL-2.0): `chip/fm1_gpio.h`, `chip/fm1_uart.h`,
  `boards/fm1/fm1_input.h`, `fm1_lcd_hw.h`, `fm1_adc.h`, `fm1_audio.h`. Felucca keeps its own `firmware/hal/fm1_uart.h`.
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) at `fdd709f`: `boards/fm1/` (`fm1_audio.h` and
  `docs/BOARD_IO.md`, [e7a22e3](https://github.com/hugelton/felucca-bsp/commit/e7a22e3); `fm1_jack.h`,
  [7bcea16](https://github.com/hugelton/felucca-bsp/commit/7bcea16); `fm1_sd_hw.h`,
  [db8bbc3](https://github.com/hugelton/felucca-bsp/commit/db8bbc3), [c76eb34](https://github.com/hugelton/felucca-bsp/commit/c76eb34)).
- [Felucca](https://github.com/hugelton/Felucca) `firmware/src/lcd.c` at `ac8cc0e` (PA2 as /OE).
- [M-VAVE FM1 Open](https://github.com/m-vave/fm1_open) `app/board/fm1_audio.h` at `4c7496d`.
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §2.2, §8.2, §8.4, §11.5 (at `eb4f2c1`, 2026-11-23).
- [zvenson/dinghy PR #4](https://github.com/zvenson/dinghy/pull/4) (PA2 and the rescue key).
- [Optimist `docs/SD-CARD-DESIGN.md`](https://github.com/w0ts/optimist/blob/b32f7321/docs/SD-CARD-DESIGN.md) §3 (J12).
- [FM-1 resource hub REVERSE_ENGINEERING.md](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md)
  §1.2 (commit `d269391`, 2026-10-28; no licence declared, summarised only); `HARDWARE_MODS.md` item 7
  ([7ad4fe4](https://github.com/aroum/fm1-custom-fw/commit/7ad4fe4), 2026-11-18).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
