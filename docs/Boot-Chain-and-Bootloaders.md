---
title: "Boot Chain and Bootloaders"
parent: Boot and install
nav_order: 1
---

# Boot Chain and Bootloaders

## From power-on to your code

```
mask ROM ──> bootloader ("uboot.boot", flash 0xA0, runs from RAM at 0x01C02000)
               │  sets clocks, reads the flash directory, sets up XIP decryption and the cache
               ├──> your app at 0x02000120 (XIP)                 normal start
               └──> staged update loader at 0x01C0A800 (RAM)     if an update record is pending
```

1. **The mask ROM** reads the flash header at offset 0, finds the boot container at `0xA0` (decoded with the fixed
   key `0xFFFF`), loads its payload to `0x01C02000` and jumps to it. **[RE] [HW]**
2. **The bootloader** (JieLi calls it the SPL or `uboot.boot`) sets the clock, reads the app directory, programs
   the SFC and the XIP key, invalidates the cache and starts the app at `0x02000120`. If an update record is
   pending, it starts the staged update loader instead. **[RE] [CODE]**
3. **Your app** starts with the clock, cache and XIP as the bootloader left them. Everything else (stacks, `.data`,
   `.bss`, vectors, peripherals) is yours to set up. **[CODE]**

The full contract, with every field, is in
[m-bootloader's SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §4–§6. It is the most
complete public description of the FM-1's boot path. m-bootloader no longer calls itself "clean room" (since
2026-11-13, commit [207cc85](https://github.com/jvitkauskas/m-bootloader/commit/207cc85)): it calls itself an
"independent reimplementation", written from a specification that was made by studying JieLi's bootloader. How the
two sides were kept apart, and where that was not strict, is in its
[docs/PROVENANCE.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/PROVENANCE.md).

## The bootloaders you will meet

| Bootloader | Licence | Notes |
| --- | --- | --- |
| M-VAVE's stock V15 (`FM-1_015`): JieLi's AC79 SDK 1.1.9 `uboot.boot` | see below | Hands the app 360 MHz (clock select 6, from the second PLL). No keys, no menu. Checks only the file table, not the firmware. The SMK-37's stock 016 carries this same file. **[HW]** Most community packages, including Felucca's, carry an SDK `uboot.boot` in their head ([Firmware Package Format](Firmware-Package-Format.md)). |
| M-VAVE's stock V16 (`FM-1_016`): JieLi's AC79 SDK 1.2.13 `uboot.boot` | see below | Its `isd_config` asks for **320 MHz**: clock select 2, from the 480 MHz main PLL, with the second PLL left off, so HSB 160 MHz and LSB 53.3 MHz. It also sets the I/O supply to 3.2 V, which lasts only until the firmware sets its own. It is the same file as FM1 Open's `sdk/tools/pack/uboot.boot`. **[HW]** |
| [m-bootloader](https://github.com/jvitkauskas/m-bootloader) | MIT | Version 3.0 (2026-11-22), rewritten in Rust; it does what 2.4 does. **REC** at power-on: download mode. **PLAY** at power-on: a menu with SD card (with a card mod), Boot app or Boot anyway (once, past a bad checksum), Download mode and **MIDI mode** (install from a browser over USB MIDI, below). The menu opens by itself in MIDI mode when the firmware fails its check (`APP BAD CRC`, `APP INCOMPLETE`, `APP NOT FOUND`). Checks every firmware file's checksum before starting it. Tested on FM-1 and SMK-37 Pro. Details below. |
| fm1-bootloader | (withdrawn) | A C restructuring of the SDK's stock bootloader, from a byte-identical reconstruction. **Withdrawn by its author** (the repository returns "not found", checked 2026-11-26): it is derived from JieLi's code, and whether the SDK's Apache-2.0 licence covers JieLi's bootloader binary is "not certain" (m-bootloader `docs/PROVENANCE.md`, [cfa70fa](https://github.com/jvitkauskas/m-bootloader/commit/cfa70fa), 2026-11-13). |

The two stock rows come from m-bootloader's
[docs/STOCK-V15-V16.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md)
([121cfbe](https://github.com/jvitkauskas/m-bootloader/commit/121cfbe)): boot probes on one FM-1 (Puya flash)
on 2026-11-14 and 2026-11-21, and the packages' boot areas. The firmware's own clock, set after the bootloader, is
not measured there ([Clocks and Power](Clocks-and-Power.md)).

**The stock bootloader's licence: two readings.** M-VAVE's FM1 Open lists `uboot.boot`, `ota.bin`,
`cfg_tool.bin`, `boot-config.bin` and the V15 package as "distributed by M-VAVE under the Apache License, Version 2.0"
([THIRD_PARTY.md](https://github.com/m-vave/fm1_open/blob/4c7496d/THIRD_PARTY.md) at 4c7496d). m-bootloader's
provenance notes say JieLi ships the bootloader only as a binary in an Apache-2.0 SDK, "without source or terms of
its own", and that whether that licence covers the binary "is not certain". Both are statements by the projects,
not legal advice.

**Installing an official M-VAVE update, or a firmware whose package carries the stock bootloader, puts the stock
bootloader back** (m-bootloader README; Baud Girl's installer does it too). **[HW]** M-VAVE's own V15 → V16 update
through M-UPGRADE did this on an FM-1 (m-bootloader
[v3.0 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0), 2026-11-22), and the probes above
read V16's bootloader on a unit running V16. So **an official V16 install brings the V16 bootloader and its
320 MHz set-up**; going back to V15 brings V15's. Felucca-family update loaders never write the boot area. If you rely on
m-bootloader's menu, check after each install. SLOOP's installer was seen to keep m-bootloader (SPEC §13).

### m-bootloader in more detail

- **Two stages.** Stage 0 is the ROM entry and all that download mode needs: power and clock, the key scan, REC. It
  unpacks stage 1 (Upkr-packed) through `0x01C0A800` (SPEC §4.4). Holding REC at power-on "bypasses almost all the
  bootloader code" (its author, [m-bootloader#37](https://github.com/jvitkauskas/m-bootloader/issues/37),
  2026-11-21). **[CODE]**
- **The clock it hands over.** Version 1.2 left the clock at 320 MHz. Since 2.3 it hands the app 360 MHz, as stock
  V15 does: clock select 6, the same PLL, bus and flash-controller values (measured with 2.3 on one FM-1,
  2026-11-14, STOCK-V15-V16.md). The specification now asks for 360 MHz always (SPEC §3.2). **[HW]**
- **The I/O supply.** 2.4 lowered the I/O supply from the chip's 3.5 V at power-on to 3.2 V until the firmware set its
  own; later versions leave it at 3.5 V (README). **[HW]**
- **1.2 over a V16 boot area** stops at its menu with `BAD CONFIG` on every start, as it rejects V16's 320 MHz
  configuration; Boot app still starts the firmware, and 1.3 or later fixes it (README; fm1-base
  [4cc3781](https://github.com/charlesvestal/fm1-base/commit/4cc3781)). **[HW]**
- **No more "wipe user data".** Version 2.4 (2026-11-13) removed the menu's Wipe user data item
  ([2e7a96b](https://github.com/jvitkauskas/m-bootloader/commit/2e7a96b)): firmwares keep their settings in
  different places, so erasing the declared areas could make things worse. Use the firmware's own reset, or restore
  a backup.
- **MIDI mode** answers as `ota-FM-1_900` (`ota-SMK-37 Pro_900`) on USB `1209:0002` "FM-1 Update", writes only
  `[0x4000, size)`, erases `0x4000` first and writes it last, and also answers read-only host commands (backup over
  MIDI, RAM apps). See [Update Protocol and Installers](Update-Protocol-and-Installers.md) and [Recovery](Recovery.md).
- **The SD card item** loads and runs `/MBOOT/MENU.BIN` from a microSD card on the panel's SPI1, with a card mod
  (SPEC §11.5); it touches the card only while the menu is open.
- **The newer flash (UC25HQ80, `0xB36014`).** The sources disagree; see [Open Questions](Open-Questions.md). The README's
  warning was removed on 2026-11-23 ([eb4f2c1](https://github.com/jvitkauskas/m-bootloader/commit/eb4f2c1)); asked
  whether 2.4 had been tested on that chip, the author answered "Yes" (m-bootloader#37, 2026-11-21); the
  [v3.0 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0) list "the UC25HQ80" under
  "Not tested"; M-Flasher's install dialog still says such units "have had little testing with m-bootloader"
  (`web/app.js` on main, 2026-11-26). **[?]**

## What your app should do at start

From Felucca-family firmware and felucca-bsp. **[CODE]**

1. Mask interrupts, set the stack pointers, copy `.data` and any RAM code from flash, clear `.bss`.
2. Read and keep the **reset reason** first thing (P33's reset source: power-on, watchdog, VDDIO low, long press,
   soft reset).
3. Arm the **watchdog** early, and feed it from the main loop.
4. Put the **clock** and the **supply voltages** back to stock ([Clocks and Power](Clocks-and-Power.md)).
5. Set up the interrupt table (all 128 vectors to a fatal stub first) and the hardware guards ([Debugging](Debugging.md)).
6. Count boots in a `NOINIT` area: if the firmware crashes twice during start-up, come up in a **safe mode** with
   no audio and USB on, so the user can install something else. Felucca-family firmwares do this.
   **Make that record `volatile`** (or otherwise keep the compiler from shrinking it). In felucca-bsp's `hwtest`
   example, a `static` record in `.noinit` that was only ever given its magic word was shrunk by the compiler to
   1-bit flags. Whatever RAM V15 or V16 left behind then passed for a valid record, and the first boot after an
   install landed in download mode, 3 of 3 times from V15 and 2 of 2 from V16
   ([felucca-bsp#17](https://github.com/hugelton/felucca-bsp/issues/17), hellsend 2026-10-28, charlesvestal
   2026-10-31; fixed in [14610bd](https://github.com/hugelton/felucca-bsp/commit/14610bd)). fm1-base's and
   Felucca-based firmwares' guards are globals and were checked to be kept whole. **[HW]**
7. Only then: the screen, the inputs, the audio, USB.

## Leaving your app

| To | How | Tag |
| --- | --- | --- |
| Reboot | chip reset through P33 (`P3_PR_PWR` bit 4) | [CODE] |
| ROM download mode | write `usb_update_mode\0` (16 bytes) at `0x01C7FD80`, then a software reset (power control `0x10000` bit 4). Don't jump into the ROM directly. | [HW] |
| The update loader | write the 112-byte `UPDATA_PARM` record at `0x01C7FD88` (and its copy in flash), then reset: the bootloader starts the staged loader | [CODE] |

Stop the audio DMA before any of these ([Audio Output](Audio-Output.md)).

**Put the clock back first** if your firmware raised it. A soft reset keeps the PLL, so the bootloader, the update
loader or the ROM would start above stock. felucca-bsp's `fm1_clock_restore()` does this, and fm1-base calls it
before every reset and update ([d61bfcb](https://github.com/hugelton/felucca-bsp/commit/d61bfcb), `chip/fm1_clock.h`;
fm1-base `device/main_fm1.c` `enter_uboot`). **[CODE]**

## Sources

- [m-bootloader](https://github.com/jvitkauskas/m-bootloader) README and SPEC.md §3.2, §4, §6, §7, §11.5, §11.6, §13
  (at [eb4f2c1](https://github.com/jvitkauskas/m-bootloader/commit/eb4f2c1), 2026-11-23);
  [docs/STOCK-V15-V16.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md) (121cfbe,
  2026-11-21); [docs/PROVENANCE.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/PROVENANCE.md) (207cc85,
  cfa70fa); release notes [v2.4](https://github.com/jvitkauskas/m-bootloader/releases/tag/v2.4) and
  [v3.0](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0);
  [m-bootloader#37](https://github.com/jvitkauskas/m-bootloader/issues/37) (2026-11-21).
- [M-Flasher](https://github.com/jvitkauskas/m-flasher) `web/app.js` (main, 2026-11-26).
- M-VAVE's [FM1 Open](https://github.com/m-vave/fm1_open) `THIRD_PARTY.md` (4c7496d).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) issue #17, commits 14610bd and d61bfcb (MPL-2.0);
  [fm1-base](https://github.com/charlesvestal/fm1-base) `device/main_fm1.c` (3039770), commit 4cc3781.
- [FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §3.
- Felucca's start-up and its derivatives' (GPL-3.0-only). Since Felucca 1.7.0 the hardware layer is the
  felucca-bsp submodule: `chip/fm1_sys.h`, `chip/crt0.S` there (MPL-2.0); older forks keep `hal/fm1_sys.h`.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
