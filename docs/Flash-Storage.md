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

**Third-party firmwares use these areas differently.** Felucca/SLOOP can keep projects and samples inside the
stock VM range and settings elsewhere; Baud Girl moves its VM to `0xD9000`. m-bootloader's "wipe user data"
erases exactly the VM and BTIF ranges the installed firmware declares, "not a promise to remove every
third-party setting or sample". Gaps are not necessarily free: another firmware may have used them.

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

**The audio stops while you erase.** With interrupts off, the audio DMA keeps looping the last buffer: zero the
buffer first, or it buzzes. A 4 KiB sector erase takes tens of milliseconds on typical NOR parts (not measured
on the FM-1 here **[?]**), which is an audible gap. Save when the user asks, or when the music is stopped, not in
the middle of a performance.

## Big data beyond the app slot

The app slot is 581,564 bytes. Data that doesn't fit (samples, wavetables) can live elsewhere in the flash and be
read through the XIP window or by SPI reads. fm1-base offers `plat_data*` with a data area and two more windows.
Anything read through XIP is decrypted on the fly if it is outside the "plain" window that the SFC is told about
(Felucca's `fl_plain_window_init`). **[CODE]**

## Bigger flash

felucca-bsp has code for a larger flash part soldered in place of the stock SOIC-8, never run on a device. How
much the XIP window maps is unknown (felucca-bsp `docs/EXPERIMENTAL.md`). **[?]**

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §5.5.
- [FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §2.
- Felucca's `src/storage.c`, `hal/fm1_flash.h`, `hal/fm1_xip.h` (GPL-3.0-only).
- [fm1-base README](https://github.com/charlesvestal/fm1-base); [felucca-bsp EXPERIMENTAL.md](https://github.com/hugelton/felucca-bsp/blob/main/docs/EXPERIMENTAL.md).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
