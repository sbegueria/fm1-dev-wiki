---
title: "SMK-37"
parent: Hardware
nav_order: 9
---

# SMK-37

The M-VAVE SMK-37 Pro and Elite are 37-key controllers with 16 pads, 8 encoders, 4 faders, pitch and mod wheels,
a colour screen and a built-in synth. Inside: **the same JieLi AC791N (WL82) chip as the FM-1, the same
second-stage bootloader, the same update loader and the same chip key. Only the panel wiring is different**
(genkigenki's [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37)). **[HW]**

In the r/MVaveFM1 thread, several people asked that the SMK-37 be treated as a first-class target from the start,
so the community doesn't split in two.

## What differs from the FM-1

| | FM-1 | SMK-37 Pro / Elite | Tag |
| --- | --- | --- | --- |
| Key matrix | 2 × 74HC595, 16-bit word, MSB first | 4 shift registers, 32-bit word sent **least significant byte first**, MSB first within each byte; output enable PA0 active low | [HW] |
| Matrix rows | PA0, PA5–PA8, PB7 | PH8 for the menu keys; others PA2, PA5, PA6, PA7, PA9, PH6, PH9 | [HW] |
| LEDs | PA9, PA10, PH6, PH9 lines | key LEDs share the high-speed USB PHY's D+/D− pads (`0x16A0C`), two commons | [HW] |
| Screen SPI1 | group B: CLK PC9, MOSI PC10, CS PC7, D/C PC8; backlight PA2 | group A: CLK PB6, MOSI PB7, CS PC8, D/C PB5; no controllable backlight | [HW] |
| Panel init | short ST7789 sequence | full ST7789 sequence with porch, gate, VCOM, voltages and gamma | [HW] |
| Flash layout (stock) | firmware `[0x4000, 0x93000)` | firmware `[0x4000, 0x9D000)`, VM `[0x9D000, 0xC0000)`, USRTRIM and USRFLASH | [HW] |
| Calibration | none known | **factory calibration** at `0xC1000` (`trimdat` marker) and wheel values in the presets header: **unit-specific, never copy between units** | [HW] |
| DIN MIDI in | TRS in | Pro and Elite have **no DIN MIDI input**; DIN out exists | [CODE] |
| Bootloader keys (m-bootloader) | REC / PLAY | SEQ REC / SEQ PLAY | [HW] |
| Soft key | first USB-MIDI port "FM-1 MIDI 1" | first of three ports, "… Private" | [HW] |

(m-bootloader SPEC §5.5, §8.2–§8.4; M-VAVE's [smk37_open](https://github.com/m-vave/smk37_open) README.)

## Where to look

- **M-VAVE's [smk37_open](https://github.com/m-vave/smk37_open)**: SMK37 OPEN V01, the same app as FM1 Open, with a
  binary SDK; board drivers in `app/board/`. Licence: GPLv3 with an SDK linking exception. A documented reference
  for the SMK's pins, pads and faders.
- **[fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37)**: a kit to move an FM-1 firmware to the SMK-37 with
  a coding agent, keeping the FM-1 build byte-identical; a list of ports already done (SLOOP, X0X, FoMni,
  Felucca and others) with their upstream pull requests; pitfalls and mapping patterns.
- **[m-bootloader](https://github.com/jvitkauskas/m-bootloader)**: tested on the SMK-37 Pro; its SPEC has the
  SMK's scan, LED and panel facts. It also guards the SMK's factory calibration.
- felucca-bsp lists the SMK-37 Pro and Elite as "planned"; fm1-base "needs a PR for SMK-37 support".

## Advice for a firmware that wants both

- Keep the board's facts (pins, matrix, LEDs, panel) behind one board layer, chosen at build time
  (e.g. `BOARD_SMK37`), so the FM-1 build stays byte-identical.
- Decide early what the pads, faders and extra keys do in your UI: it's easier than retrofitting.
- **Never write the SMK's calibration area**, and never ship an image that contains one unit's calibration.
- Test with M-VAVE's stock firmware back on the unit afterwards: stock firmware has been seen to reset presets and
  erase USRTRIM/BTIF when USRFLASH is blank (m-bootloader SPEC §5.5).

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §5.5, §8.
- [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37), [smk37_open](https://github.com/m-vave/smk37_open) READMEs.
- [r/MVaveFM1 thread](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
