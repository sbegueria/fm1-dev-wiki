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

"The same bootloader" means the FM-1's **V15** one. The SMK-37's stock 016 packages carry JieLi's AC79 SDK 1.1.9
`uboot.boot`, the same file as FM-1 V15's, and their boot configuration is byte-identical to FM-1 V15's after the
chip-key record (360 MHz). FM-1 V16 brought a newer bootloader with a 320 MHz set-up; the SMK-37 has nothing like it
(m-bootloader [docs/STOCK-V15-V16.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md),
121cfbe, 2026-11-21). **[RE]**

In the r/MVaveFM1 thread, several people asked that the SMK-37 be treated as a first-class target from the start,
so the community doesn't split in two.

## What differs from the FM-1

| | FM-1 | SMK-37 Pro / Elite | Tag |
| --- | --- | --- | --- |
| Key matrix | 2 × 74HC595, 16-bit word, MSB first | 4 shift registers, 32-bit word sent **least significant byte first**, MSB first within each byte; output enable PA0 active low | [HW] |
| Matrix rows | PA0, PA5–PA8, PB7 | PH8 for the menu keys; others PA2, PA5, PA6, PA7, PA9, PH6, PH9 | [HW] |
| LEDs | PA9, PA10, PH6, PH9 lines | key LEDs share the high-speed USB PHY's D+/D− pads (`0x16A0C`), two commons | [HW] |
| Screen SPI1 | group B: CLK PC9, MOSI PC10, CS PC7, D/C PC8; backlight PA2 | group A: CLK PB6, MOSI PB7, CS PC8, D/C PB5; no controllable backlight | [HW] |
| Panel init | m-bootloader sends a short ST7789 sequence; M-VAVE's firmware and FM1 Open send the full one too (FM1 Open `app/board/lcd.inc`), and newer FM-1 panels need it ([Display](Display.md)) | full ST7789 sequence with porch, gate, VCOM, voltages and gamma | [HW] [CODE] |
| Flash layout (stock) | firmware `[0x4000, 0x93000)` | firmware `[0x4000, 0x9D000)`, VM `[0x9D000, 0xC0000)`, USRTRIM and USRFLASH | [HW] |
| Calibration | none known | **factory calibration** at `0xC1000` (`trimdat` marker) and wheel values in the presets header: **unit-specific, never copy between units** | [HW] |
| Flash chip (JEDEC ID) | `0x856014` (Puya), or `0xB36014` (UC25HQ80) on newer units | `0xEB6014` on two units reported so far ([fm1_open#11](https://github.com/m-vave/fm1_open/issues/11), dsgnloopz 2026-10-21; the SMK-37 Pro in m-bootloader's [v3.0 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0)). Firmware that checks the ID must accept it | [HW] |
| DIN MIDI in | TRS in | Pro and Elite have **no DIN MIDI input**; DIN out exists | [CODE] |
| Bootloader keys (m-bootloader) | REC / PLAY | SEQ REC / SEQ PLAY | [HW] |
| Soft key | first USB-MIDI port "FM-1 MIDI 1" | first of three ports, "… Private" | [HW] |

(m-bootloader SPEC §5.5, §8.2–§8.4; M-VAVE's [smk37_open](https://github.com/m-vave/smk37_open) README.)

## Where to look

- **M-VAVE's [smk37_open](https://github.com/m-vave/smk37_open)**: SMK37 OPEN V01, the same app as FM1 Open, with a
  binary SDK; board drivers in `app/board/`. Licence: GPLv3 with an SDK linking exception. A documented reference
  for the SMK's pins, pads and faders. What its code does (at b2d6bcb, 2026-10-24): **[CODE]**
  - Identities `SMK-37 Pro_8000001` and `SMK-37 Elite_8000001`; update families 8 (Pro) and 9 (Elite). The kit pins
    M-VAVE's stock V16 packages (dated 2026-08-12): Pro `2d98ce72…`, Elite `b2ee38ab…`. The stock Pro V16 package
    uses 36 identity marker slots, not 20 ([Firmware Package Format](Firmware-Package-Format.md)).
  - It **does not read the factory calibration** at `0xC1000`. It reads its own records 51–54, or falls back to
    default curves (`app/board/scan.c`).
  - Its package uses the FM-1's layout, with a `USER` region `[0x7C000, 0xFF000)` (`sdk/Package.cs`), which spans
    the calibration and Bluetooth sectors `0xC0000`–`0xC1FFF`. Whether its firmware or updater ever writes there is
    not known. **[?]**
  - Its return to stock writes only the stock `USR` resource, `0xF4000`–`0xFDFFF` (37,382 bytes, checked by
    SHA-256), with the comment "Never touch calibration, user patterns, VM or Bluetooth addresses"
    (`sdk/WebUpdate.js`).
  - M-VAVE has not answered its issues #1, #2, #4 or pull request #3 (checked 2026-11-26).
- **[fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37)**: a kit to move an FM-1 firmware to the SMK-37 with
  a coding agent, keeping the FM-1 build byte-identical; a list of ports already done (SLOOP, X0X, FoMni,
  Felucca and others) with their upstream pull requests; pitfalls and mapping patterns.
- **[m-bootloader](https://github.com/jvitkauskas/m-bootloader)**: tested on the SMK-37 Pro; its SPEC has the
  SMK's scan, LED and panel facts. It also guards the SMK's factory calibration (keeps a copy and puts it back if it
  is erased). Its MIDI mode answers as `ota-SMK-37 Pro_900` and never writes `0xC0000`–`0xC1FFF`; it can't tell a
  Pro from an Elite, so an Elite installed through it reports itself as a Pro (SPEC §11.6). The 3.0 code was tested
  on an SMK-37 Pro under its 2.4 version strings; the exact 3.0 build was not (v3.0 release notes). **[HW]**
- **[M-Unbricker](https://github.com/jvitkauskas/m-unbricker)**: a working SMK-37 can revive a dead FM-1, and the
  other way round; it never writes the SMK's `0xC0000`–`0xC1FFF` ([Recovery](Recovery.md)). **[HW]**
- **felucca-bsp**: its README still lists the SMK-37 Pro and Elite as "Planned" (fdd709f, 2026-11-26). A board layer
  is **in progress** in pull request [felucca-bsp#30](https://github.com/hugelton/felucca-bsp/pull/30) (keremimo,
  open since 2026-11-06, updated 2026-11-26): `boards/smk37/` for the SMK-37 Pro, Elite and MKE-P37, with the app area
  `0x4000`–`0x9D000`, MIDI out on UART1, and the boot head, BTIF, USRTRIM (calibration), USR and key_mac never written.
  Its board facts come from fm1-to-smk37, marked *verified* or *analysis*; its audio set-up is not yet measured on an
  SMK. **[CODE]**
- fm1-base "needs a PR for SMK-37 support".

## Advice for a firmware that wants both

- Keep the board's facts (pins, matrix, LEDs, panel) behind one board layer, chosen at build time
  (e.g. `BOARD_SMK37`), so the FM-1 build stays byte-identical.
- Decide early what the pads, faders and extra keys do in your UI: it's easier than retrofitting.
- **Never write the SMK's calibration area**, and never ship an image that contains one unit's calibration.
- Test with M-VAVE's stock firmware back on the unit afterwards: stock firmware has been seen to reset presets and
  erase USRTRIM/BTIF when USRFLASH is blank (m-bootloader SPEC §5.5).
- Back up before installing Baud Girl: it overwrites the SMK-37's stock presets, so stock firmware installed
  afterwards has none until a backup is restored, and its installer rewrites the Bluetooth sector at `0xC0000`, which
  may lose pairings; calibration stays (m-bootloader README). **[HW]**
- Not every SMK firmware takes the same update path. ChanceTheMaker's SMK firmware (`SMK-37_9NN`) answers the upgrade
  message by starting its own updater, `ota-SMK-37_930`, which takes only its own kind of package (m-bootloader SPEC
  §11.7). **[HW]**

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §5.5, §8, §11.6, §11.7,
  README and [docs/STOCK-V15-V16.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md)
  (eb4f2c1); [v3.0 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0).
- [smk37_open](https://github.com/m-vave/smk37_open) `app/version.h`, `app/board/scan.c`, `sdk/Package.cs`,
  `sdk/WebUpdate.js` (b2d6bcb); [FM1 Open](https://github.com/m-vave/fm1_open) `app/board/lcd.inc` (4c7496d),
  [fm1_open#11](https://github.com/m-vave/fm1_open/issues/11).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) README (fdd709f), [PR #30](https://github.com/hugelton/felucca-bsp/pull/30).
- [M-Unbricker](https://github.com/jvitkauskas/m-unbricker) README (204f88d).
- [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37), [smk37_open](https://github.com/m-vave/smk37_open) READMEs.
- [r/MVaveFM1 thread](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
