---
title: "Flash Storage"
parent: Boot and install
nav_order: 4
---

# Flash Storage

The FM-1 has one 1 MiB SPI NOR flash. It holds the boot area, your code (run in place) and any data you save.

## The stock layout

**[RE] [HW]** As declared by M-VAVE's FM-1 images (m-bootloader SPEC §5.5; FM-1-RE):

| Range | Stock use |
| --- | --- |
| `[0x00000, 0x04000)` | Boot area: flash header, directory, bootloader at `0xA0`, `isd_config` (chip key) |
| `[0x04000, 0x93000)` | Firmware (the app area; `app.bin` at `0x4120`) |
| `[0x93000, 0xE9000)` | VM: JieLi's key/value config store |
| `[0xE9000, 0xEA000)` | BTIF: Bluetooth pairing |
| `[0xEA000, 0xFC000)` | USR: user patches and resources |
| `[0xFF000, 0x100000)` | Unit reservation (`key_mac`) |

**Reading a region table.** **[RE]** In SMK-37 stock 016, the region table (in the decrypted flash from `0x4060`)
gives VM, BTIF, USRTRIM, USRFLASH and USR as **absolute** flash offsets, not relative to the app area at `0x4000`.
Some unpackers add `0x4000` and report these regions 16 KiB too high. The FM-1 ranges above are given as absolute
offsets too (m-bootloader SPEC §5.5). Each entry also carries flags; a
package for a device should carry that device's table with the stock flags, so the official loader treats it the same
way (genkigenki's [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) 01 §2, 02 §2.3, 05 §2; read from SMK-37
stock 016). The SMK-37's own layout, with its calibration and the risk of a blank USRFLASH, is on [SMK-37](SMK-37.md).

**Where the stock data really is.** **[HW]** On one FM-1 that had run stock V15 (read 2026-10-08), the SDK's VM
was at **`0xE8000`** (at most 4 KiB, since BTIF follows), not at the start of the declared range: magic `55 AA AA 55`, then
records 106, 107, 187, 108, 113 and 109. 106, 107, 108 and 187 are the radio's calibration (only 187 measured per unit; the others hold the SDK demo board's constants); 109 and 113 are BLE keys ([Bluetooth and WiFi](Bluetooth-and-WiFi.md#calibration-and-where-it-lives)). BTIF at **`0xE9000`** holds the
Bluetooth address (record 102). `0x93000`–`0x96FFF` and `0xE7000` read `FF` (fm1-base's
`docs/bt/BLE-HW-FACTS.md` §1, §14.1, a document that came from Optimist; one unit).

**[CODE]** These two sectors sit inside ranges that some firmwares use for big data. fm1-base's `PLAT_XDATA1`
window (`0xE5000`–`0xFBFFF`) covers both, and LOOM-8's sample bank and Polly's QY70 bank fill that range. "On a unit
where either bank was loaded large enough, the VM at 0xE8000 and BTIF at 0xE9000 are probably gone already"
(charlesvestal, [fm1-base#14](https://github.com/charlesvestal/fm1-base/issues/14), 2026-10-31, still open). Since
2026-11-21 neither of fm1-base's Bluetooth stacks reads the stock VM: the own stack calibrates at every start and
takes its address from the flash's unique ID. What stock firmware does with an empty VM and BTIF is not known.

**V16 moved things.** **[HW]** One user compared full flash dumps (ocean,
[fm1_open#16](https://github.com/m-vave/fm1_open/issues/16), 2026-11-02). V16's own regions end at `0xECFFF`
(VM `0xBA000`, SEQ64 `0xDA000`, BTIF `0xEC000`). Yet after V16 had run, V15's USR region at `0xF2000`–`0xFBFFF` had
changed too. V15's settings record at `0xFBD80`–`0xFBE09` then read `FF` bytes, and V15 played **one note at a time**
after going back. Restoring only `0xFA000`–`0xFBFFF` to V15's factory data fixed it. V15's own update and voice
commands can't put that record back. One unit; no answer from M-VAVE in the issue by 2026-11-26.

**M-VAVE's FM1 Open** declares a different layout **[CODE]** (`sdk/Package.cs`, at
[4c7496d](https://github.com/m-vave/fm1_open/tree/4c7496d)): the app from `0x4000`, then its VM (at least 32 KiB)
up to `0x7B000`, BTIF `[0x7B000, 0x7C000)`, and USER `[0x7C000, 0xFF000)` (536,576 bytes). Inside USER, projects
take the first 68 KiB, a 456 KiB sample bank follows, and the last five sectors hold the four DX7 banks and the
tuning and session settings (`app/instrument.h`).

**Third-party firmwares use these areas differently.** Felucca/SLOOP can keep projects and samples inside the
stock VM range and settings elsewhere; Baud Girl moves its VM to `0xD9000`. Gaps are not necessarily free: another
firmware may have used them. m-bootloader 2.4 (2026-11-13) removed its "Wipe user data" menu item for this reason:
"Firmwares keep their settings, projects and samples in different places, often outside the areas they declare or
next to data you want" ([v2.4 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v2.4)).

**Practical rule:** when your firmware starts, don't trust what it finds in its storage. Check a magic number,
a version and a CRC on everything you load, and fall back to defaults. Another firmware may have left its own data
in the same sectors.

## One firmware's map (example)

Felucca's family, as in X0X, AMB-1 and DUB-1 **[CODE]**:

| Where | What |
| --- | --- |
| `0xFC000` | the project / global settings (Felucca's `FL_GLOB`) |
| `0xDC000` | panel calibration (Felucca's user-preset area) |
| `0xE0000`… | **the update loader's staging area: never store data here** |

Other Felucca-family firmwares use more of the flash. Dinghy's and Felucca's layout as summarised in mvave-fm1's
development skill (§3), and TransitFM1's notes **[CODE]**:

| Where | What |
| --- | --- |
| `0x97000…0xDFFFF` | songs and sounds (Felucca family), inside the stock VM range |
| `0xE0000…0xE4FFF` | update-loader staging; the update record's flash copy at `0xE4F00` (TransitFM1, ChoralRoot) |
| `0xE5000…0xE6FFF` | Felucca's autosave, A/B |
| `0xE6000` | where **M-VAVE's V15** stages its update loader ([Update Protocol and Installers](Update-Protocol-and-Installers.md)) |
| `0xE7000` | Felucca's FM6 voice bank |
| `0xFC000…0xFEFFF` | settings and globals (`SETTINGS_AT = 0xFC000` in Dinghy) |

Note the overlaps (`0xE6000` in two rows): each firmware's map is its own, and a firmware that follows V15 may find
V15's loader where it keeps data. Never assume a sector is free because your own map says so. **[?]**

**Three address spaces.** A physical flash offset, an offset into the decoded `app.bin` (byte 0 = flash `0x4120`
in the stock layout) and a CPU address in the XIP window (`0x02000120`) are different numbers for the same byte.
Never copy an address from another firmware or revision into a write (mvave-fm1 development skill §3).

## Writing safely: A/B sectors, header last

**[CODE]** Felucca-family storage (`src/storage.c`, GPL-3.0-only) gives every object two 4 KiB sectors. A save
goes to the copy that is *not* current: erase it, program the payload (from offset 256), then the 32-byte header at
offset 0 **last**. The header is the commit record (magic, type, slot, sequence number, length, CRC-32 of the
payload, CRC of the header). On load, the valid copy with the highest sequence number wins. A save cut off by power
loss leaves the previous copy in charge.

## Erasing and programming with code running from flash

**[CODE]** While the flash is being erased or programmed, it can't be read, so XIP stops. The code that drives
the erase must:

- live in RAM (a `.ram_text` section copied at start-up);
- touch only registers, the stack and RAM buffers: **no constants in flash, no library calls, no helper functions
  in flash**;
- run with **interrupts off** (an interrupt handler in flash would fetch from the flash);
- be called through a far pointer ([Memory Map](Memory-Map.md)).

Felucca's build checks that `.ram_text` contains no `call`. Its sequence follows the SDK's flash driver
(`enter_spi_code`, `norflash_erase`, `norflash_wait_ok`, `exit_spi_code`).

**Never go back to XIP while the part is busy.** **[CODE]** felucca-bsp's old driver gave up waiting after a fixed
bound (500 ms for an erase, 10 ms for a page program) and switched XIP back on while the part could still be busy.
The next code fetched from flash was then "whatever the bus returned". Since 2026-11-05 it waits until the busy bit
clears; past a longer bound (2 s, 50 ms) it resets the flash part (`66h`, `99h`) and then the chip, never returning
to XIP (`chip/fm1_flash.h`, `chip/fm1_flash_wait.h`;
[2d37ec8](https://github.com/hugelton/felucca-bsp/commit/2d37ec8), from jvitkauskas's
[PR #12](https://github.com/hugelton/felucca-bsp/pull/12)). The bounds are generic NOR figures, not measurements
of the FM-1's part. If your driver is older, check this.

**The audio stops while you erase.** With interrupts off, the audio DMA keeps looping the last buffer: zero the
buffer first, or it buzzes. **[HW]** On FM-1s with the Puya part (`856014`) a 4 KiB sector erase takes about
**17–18 ms** with interrupts off (felucca-bsp `docs/EXPERIMENTAL.md`: 17 ms; charlesvestal: 17.9 ms, 2026-11-22),
against the datasheet's 8 ms typical and 12 ms maximum. A page program holds them off for 1.70 ms (hdavid). That is
longer than an audio half (2.9 or 5.8 ms), so it is an audible gap. Save when the user asks, or when the music is
stopped, not in the middle of a performance. FM1 Open's flash calls say the same: "Writes/erases briefly silence
audio" (`sdk/include/fm1.h`).

**Erasing in short windows.** **[HW]** The sliced erase (hdavid's, from Optimist; felucca-bsp `FL_ERASE_SLICED`,
fm1-base `FM1B_ERASE_SLICED`, both off by default) suspends the erase (`75h`) after about 1 ms, lets the interrupts
and XIP run, and resumes it (`7Ah`). Measured on FM-1s with the `856014` part and both cores rendering
([felucca-bsp#46](https://github.com/hugelton/felucca-bsp/pull/46), merged 2026-11-22):

| | Sliced | Inline (control) |
| --- | --- | --- |
| Longest interrupts-off window | ~1.0 ms (hdavid); 1.7 ms with the restore's page programs (charlesvestal) | 17.4 ms |
| Time per sector | 21–25 ms (19 windows) | 17.9 ms |
| Audio underruns | 0 | 6 in one pass |
| Sectors erased and checked blank | about 2,500 + 438, no failures | 73 |

The render and the second core keep running, but the 2 kHz USB service does not keep up: a USB audio recording
made during 8.9 s of erases had 634 breaks (hdavid). Not tested: the suspend on the newer UC25HQ80 part (`B36014`).
felucca-bsp's `docs/EXPERIMENTAL.md` still says the sliced erase was "never run on a device"; the PR's measurements
supersede that.

## Big data beyond the app slot

The app slot is 581,564 bytes in the stock layout. Data that doesn't fit (samples, wavetables) can live elsewhere
in the flash and be read through the XIP window or by SPI reads. fm1-base offers `plat_data*` with a data area and
two more windows (`PLAT_XDATA0`, and `PLAT_XDATA1` at `0xE5000`–`0xFBFFF`, which covers the stock VM and BTIF
sectors above).

**[CODE]** fm1-base can also make the app area bigger: `app_end` in `fm1.toml`, on a 4 KiB boundary from `0x93000`
up to `0xE0000`. `0xCE000` allows an `app.bin` of 823,228 bytes. The data area then starts at `app_end`. fm1-base's
update loaders erase an older app's code above their own end; "Loaders from fm1-base before this, Felucca's and
M-VAVE's don't", so that code stays in the next app's data sectors until it is overwritten (README "A bigger app
slot", [03fe71b](https://github.com/charlesvestal/fm1-base/commit/03fe71b), 2026-11-06).
Anything read through XIP is decrypted on the fly if it is outside the "plain" window that the SFC is told about
(Felucca's `fl_plain_window_init`). **[CODE]**

## Bigger flash

felucca-bsp has code for a larger flash part soldered in place of the stock SOIC-8, never run on a device. How
much the XIP window maps is unknown (felucca-bsp `docs/EXPERIMENTAL.md`). **[?]**

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §5.5.
- [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) `docs/01-hardware.md` §2, `docs/02-update-and-recovery.md`
  §2.3, `docs/05-reverse-engineering.md` §2 (CC BY-NC 4.0; 54252ff, 2026-10-17).
- [FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §2.
- Felucca's `firmware/src/storage.c` (GPL-3.0-only).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) `chip/fm1_flash.h`, `chip/fm1_flash_wait.h`, `chip/fm1_xip.h`,
  [EXPERIMENTAL.md](https://github.com/hugelton/felucca-bsp/blob/main/docs/EXPERIMENTAL.md) (MPL-2.0); Felucca's
  `hal/` copies moved there in 1.7.0. [PR #12](https://github.com/hugelton/felucca-bsp/pull/12),
  [PR #46](https://github.com/hugelton/felucca-bsp/pull/46) and its comments.
- [fm1-base README](https://github.com/charlesvestal/fm1-base), `docs/bt/BLE-HW-FACTS.md` §14;
  [issue #14](https://github.com/charlesvestal/fm1-base/issues/14).
- [FM1 Open](https://github.com/m-vave/fm1_open) `sdk/Package.cs`, `app/instrument.h`, `sdk/include/fm1.h`;
  [issue #16](https://github.com/m-vave/fm1_open/issues/16).
- [m-bootloader v2.4 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v2.4).
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) development skill §3 (MIT); [Dinghy](https://github.com/zvenson/dinghy) (GPL-3.0).
- [TransitFM1 docs/SAFETY.md](https://github.com/Quixotic7/TransitFM1/blob/main/docs/SAFETY.md) (record at `0xE4F00`; V15's loader at `0xE6000`).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
