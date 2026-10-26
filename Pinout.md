# Pinout

What each pin of the AC791N does on the FM-1, from working drivers. "Port" registers are at
`0x50000 + 0x40 × port` (A=0, B=1, C=2, H=7) with `OUT +0`, `IN +4`, `DIR +8` (1 = input), `DIE +0xC` (digital
input enable), `PU +0x10`, `PD +0x14`, `HD0 +0x18`, `HD +0x1C`. **[SDK] [CODE]**

SMK-37 wiring differs: see [[SMK-37]].

| Pin | Function | Notes | Tag |
| --- | --- | --- | --- |
| PA0 | Matrix row (encoders) | pull-up, low = closed | [CODE] |
| PA1 | 74HC595 latch (RCLK) | | [CODE] [HW] |
| PA2 | LCD backlight, active low | **also enables the key matrix** (below) | [HW] |
| PA3 | 74HC595 clock (SRCLK) | | [CODE] [HW] |
| PA4 | 74HC595 data (SER) | | [CODE] [HW] |
| PA5–PA8 | Matrix rows | pull-ups; low = closed | [CODE] [HW] |
| PA9, PA10 | LED lines | light the LED in the selected column | [CODE] [HW] |
| PB1 | ADC ch 3: battery divider | never drive low (resets the chip after 8 s) | [CODE] |
| PB6 | ADC ch 4: MASTER pot | | [CODE] |
| PB7 | Matrix row (encoder 6) | | [CODE] |
| PC0, PC1, PC2, PC6 | Audio output-stage control lines | driven during audio bring-up; meaning of each not documented | [CODE] [?] |
| PC7 | LCD CS, active low | SPI1 | [CODE] [HW] |
| PC8 | LCD D/C (0 command, 1 data) | | [CODE] [HW] |
| PC9 | LCD SPI clock | | [CODE] [HW] |
| PC10 | LCD SPI data out | | [CODE] [HW] |
| PH6, PH9 | LED lines | | [CODE] [HW] |
| PH8 | TRS MIDI in → UART1 RX | 31250 baud, through input channel 1 | [CODE] |
| USB D+/D− | USB0 full-speed device | also the USB_KEY recovery signal ([[Recovery]]) | [HW] |

## The key matrix and PA2

**[HW]** "PA2 MUST be driven low while the FM-1 matrix is scanned." On the FM-1, PA2 (the backlight enable) also
enables the key matrix: with PA2 high or left as an input, no key reads closed. The ROM leaves PA2 undriven, so a
key check at start-up sees nothing until PA2 is driven low (m-bootloader SPEC §8.2). A bootloader menu or a
"hold a key at power-on" feature must drive PA2 low first; this also turns the backlight on.

## Matrix layout

**[CODE]** 11 columns selected through the shift registers (a column's bit low = selected; idle word `FFFF`),
6 rows read on PA0, PA5, PA6, PA7, PA8 and PB7. Felucca's key map (`hal/fm1_input.h`, GPL-3.0-only):

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

## Sources

- Felucca's HAL (`hal/fm1_gpio.h`, `fm1_input.h`, `fm1_lcd_hw.h`, `fm1_adc.h`, `fm1_audio.h`, `fm1_uart.h`;
  GPL-3.0-only, © Leo Kuroshita), as in DUB-1.
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §2.2, §8.2, §8.4.
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) `boards/fm1/`.
