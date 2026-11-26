---
title: "Recovery"
parent: Start
nav_order: 3
---

# Recovery

What to do when an FM-1 doesn't start, or no installer finds it. **Read this before your first install.**

> Use these tools at your own risk. Every tool below makes, or asks you to make, a full flash backup first. Keep
> that backup: it holds your unit's own data (presets, calibration).

## First: make a backup while things work

[M-Flasher](https://github.com/jvitkauskas/m-flasher) (MIT; Windows, macOS, Linux) backs up the whole 1 MiB flash
through the chip's download mode. On a unit running stock firmware or Felucca, it switches the unit into download
mode over USB-MIDI by itself (its DOWNLOAD MODE button sends `F0 22 24 35 7D F7`). Do this once, before anything
else.

A backup can also come over USB MIDI, with no download mode: m-bootloader's MIDI mode, and its **RAM stage** on a unit
without m-bootloader, answer read commands for the whole flash, boot area and key area included. The stage goes on
through the running firmware's own update route and writes nothing but its own copy in the update-loader area
(m-bootloader README, SPEC §11.6–§11.7). The 3.0 release notes report the whole flash read this way and matching, on an
FM-1 and an SMK-37 Pro (2026-11-22). **[HW]**

**Don't trust a tool's "same unit" check alone.** Two different FM-1s with the Puya flash reported the same unit id
(`4150354433333610`) in M-Flasher 0.12.1, so its restore could take one unit's backup for the other's and write that
unit's factory data ([m-flasher#61](https://github.com/jvitkauskas/m-flasher/issues/61), wellingtonsaam 2026-11-16,
open). Label your backups yourself. **[HW]**

## The ladder: from easiest to hardest

| The unit… | Try | Tag |
| --- | --- | --- |
| starts, but the firmware is broken or unwanted | install another firmware, or stock V15 or V16, with a web or command-line installer ([Update Protocol and Installers](Update-Protocol-and-Installers.md)) | [CODE] |
| starts, but its installer can't reach it | Felucca and forks: hold **OCT− and OCT+ for 5 s** (a countdown shows from 2 s; letting go cancels). fm1-base firmwares: hold **REC at power-on** (the screen says "UPDATE MODE"). Both put the chip in ROM **download mode** (`4C4A:8057`), not the update loader: use M-Flasher, MvaveFM1Unbricker or `fm1_rescue` | [CODE] [HW] |
| starts but crashes at once, twice | Felucca-family firmwares fall into a **safe mode** (no audio, USB on) after two failed starts: install from there | [CODE] |
| has m-bootloader | hold **REC** at power-on (download mode) or **PLAY** (menu: SD card, boot app or boot anyway, download mode, MIDI mode). With a firmware that fails its check, the menu opens by itself in **MIDI mode**, so a browser installer can reinstall the unit with no keys pressed | [HW] |
| is dead, and you have a second, working FM-1 or SMK-37 | [M-Unbricker](https://github.com/jvitkauskas/m-unbricker) (below) | [HW] |
| shows on USB as `4C4A:8057` / `WL80UBOOT1.00` | it is already in ROM download mode: use M-Flasher, [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker), or `fm1_rescue` | [HW] |
| shows nothing on USB, but the firmware is crashing | without hardware (MvaveFM1Unbricker README): **leave it on for two minutes** (Felucca-family watchdog and boot-loop guard enter ROM boot after two failed boots); **hold OCT− and OCT+ at power-on for 10 s** (if the firmware reaches its main loop; on Felucca this is the download-mode key above); **three quick restarts** (on 10 s, off, three times: three crashed boots land in ROM boot, which is also how a unit can get there by accident) | [CODE] |
| shows nothing at all on USB | force download mode with a **USB_KEY dongle** (below) | [HW] |

The OCT− + OCT+ key is in Felucca's `firmware/src/main.c` (at [ac8cc0e](https://github.com/hugelton/Felucca/blob/ac8cc0e/firmware/src/main.c),
2026-11-24) and was seen on a unit running 1.8.1: it came up as `4c4a:8057` "WL80UBOOT1.00", and jl-uboot-tool read the
flash twice with matching dumps ([Felucca#320](https://github.com/hugelton/Felucca/issues/320), ronbarrantes
2026-11-09). On Linux the UBOOT disk reports no medium, so jl-uboot-tool retried its open for ever until it opened the
disk with `O_NONBLOCK` (same comment). fm1-base's REC key is in `device/main_fm1.c` (3039770). Optimist's
`docs/INSTALL-PATHS.md` lists the keys for more firmwares.

### On a Mac, a unit in download mode

Download mode appears as a mass-storage disk. macOS tries to mount it, upsets it, and it reboots a few seconds
later. Felucca-family `fm1_rescue` (macOS) waits for it, takes it away from macOS (hence `sudo`) and keeps it
(its header comments). **[HW]**

### On Linux, with MvaveFM1Unbricker

[MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker) (MIT) does the backup and restore through
jl-uboot-tool on Linux and Windows. **macOS is not supported** for the flash work (jl-uboot-tool has no macOS SCSI
back end), though its `softkey` command works on macOS too. Its `restore` writes only `[0x4000, 0x93000)`, and only
from a package whose firmware region (`0x4000…0x92FFF`, `0x8F000` bytes) has the official V15 hash
`6edf3c37fb5bbbc33607c89375ee024d5477c17914d72221c8c68e58a8255686`. Before the backup it also reads 512 bytes of
RAM at `0x01C7C000` (where Melodee leaves its crash record) as `retained-ram.bin`. (Its README.) **[CODE]**

Lessons from one run (mvave-fm1's firmware skill §10, Fedora with a toolbox container, 2026-10-22): **[HW]**
- In a toolbox or distrobox, run `modprobe sg` on the host, and `chmod 666 /dev/sg0` on the host **each time** the
  unit re-enumerates; inside the container even root could not open the node.
- **A download-mode session lasted about one tool run.** Running `info` and then `backup` separately made the chip
  reset and come back as a normal FM-1. Run `backup` straight after `softkey`, and redo both if the node goes.
- `softkey` needs `--yes` when there is no terminal.
- That unit, reporting `FM-1_015`, had a firmware region hash of `8c1d13a5…`, **not** the official V15 region hash.
  Keep your own dump: it, not the official file, is the way back to exactly what your unit had
  ([Open Questions](Open-Questions.md)).
- Keep flash dumps out of public repositories: they hold your songs, sounds and settings.

**A disputed antivirus report.** A user reported a Microsoft Defender detection on MvaveFM1Unbricker
([Reddit thread](https://www.reddit.com/r/MVaveFM1/comments/1xu1d30/severe_virus_in_mvavefm1unbricker/)). The author
disputes it. The resource hub notes that neither malware nor a false positive has been independently established
([RESOURCES.md](https://github.com/aroum/fm1-custom-fw/blob/main/RESOURCES.md), 2026-11-24). This wiki has not
checked either claim. **[?]**

### With a second unit: M-Unbricker

[M-Unbricker](https://github.com/jvitkauskas/m-unbricker) (MIT, by m-bootloader's author; a web page for Chrome or
Edge, Chrome on Android, or the MIDIWeb app on iOS) uses a **working** FM-1 or SMK-37 to revive a broken one over a
USB-C cable, with no dongle (its README, 2026-11-26): **[HW]** (the author's tests)

1. The page starts m-bootloader's RAM stage on the working unit through its firmware's update route (or uses
   m-bootloader's MIDI mode), then runs M-Unbricker there as a RAM app. The working unit's firmware and settings stay.
2. With the cable moved to the broken unit, the working unit sends the **USB_KEY** (`0x16EF`) and the calibration
   pulses, then acts as a USB host to the broken unit's ROM download mode.
3. It writes a fresh **m-bootloader**, **erases everything else up to `0xFEFFF`**, and verifies it. `0xFF000` and up
   (and on the SMK-37, `0xC0000`–`0xC1FFF`, its Bluetooth pairing and calibration) are never written. The repaired
   unit then waits in m-bootloader's MIDI mode for a firmware from any browser installer.

Either model repairs either model; the README reports it tested both ways, and on an FM-1 that hung at start with no
USB at all. Your songs and settings on the broken unit are erased: back up first if the unit still reaches download
mode.

## How the download-mode tools write

[FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter), MvaveFM1Unbricker, `fm1_rescue` and
M-Flasher follow the same safety rules, which were checked on hardware:

1. A flash loader runs in RAM (JieLi's `wl82loader.bin` from [jl-uboot-tool](https://github.com/kagaimiq/jl-uboot-tool),
   checked against its hash; or m-bootloader's own MIT flash loader in M-Flasher).
2. The chip key must be `0x980F` and the flash a known 1 MiB part, or nothing is done. Older tools know only
   `856014` (Puya); units since about autumn 2026 carry `B36014` (UC25HQ80): `fm1_rescue` accepts it from DUB-1
   1.4.4, and felucca-bsp's from 2026-11-05 ([f690fa6](https://github.com/hugelton/felucca-bsp/commit/f690fa6))
   ([Hardware Overview](Hardware-Overview.md)).
3. The whole flash is read and saved first.
4. Only `[0x4000, 0x93000)` is written, and only the 4 KiB sectors that differ; each is erased, written and read
   back. The boot area below `0x4000` and the device data above `0x93000` are not touched.
5. A final full read must match the expected image.

(From `fm1_rescue.py`'s header, which credits FM-1 Transporter's `docs/PROTOCOL.md`.) felucca-bsp's current
`fm1_rescue.py` takes V15 or V16: it picks the package whose head (`[0, 0x4000)`) equals the unit's, writes nothing if
none does, and writes up to the end of that package's app area: `0x93000` for V15, `0xA9000` for V16 (its header,
felucca-bsp fdd709f). **[CODE]**

## The newer flash chip: firmwares that can't install anything

Units with the UC25HQ80 flash (`0xB36014`) running a firmware that knows only the Puya part keep saves in RAM, and
**start no update at all**: installers stop "after 0 requests", and M-UPGRADE times out. **[HW]**

- **Felucca 1.8.1 and earlier** (fixed in 1.8.2, 2026-11-13, commit
  [f4cbb6a](https://github.com/hugelton/Felucca/commit/f4cbb6a)). The way out, from the 1.8.2 release notes: hold
  OCT− and OCT+ for 5 s (download mode), put V15 back with a tool such as M-Flasher, then install 1.8.2
  ([Felucca#320](https://github.com/hugelton/Felucca/issues/320), #324; the fix confirmed on a unit by mistakenideas).
- **fm1-base releases from before 2026-11-05** (fixed by fm1-base PR #35). Run felucca-bsp's `fm1_rescue.sh` once to
  put M-VAVE's firmware back; then current releases install normally. Still open: a unit whose first 16 KB match
  neither V15 nor V16, which the rescue script refuses to write over (fm1-x0x#55)
  ([fm1-base#36](https://github.com/charlesvestal/fm1-base/issues/36), charlesvestal 2026-11-21).
- fm1-nes's app-only updater accepts only `0x856014` (its `APP_UPDATES.md`, 90d41c5). **[CODE]**

## USB_KEY: when nothing answers

**[HW]** The mask ROM listens for a "USB key" on the USB data lines at power-on: the 16-bit word `0x16EF`, MSB
first, D+ as clock and D− as data sampled on the rising edge, at about 50 kHz (10 µs half periods), with 150 µs
gaps. The chip acknowledges by holding both lines low for 1–2 ms. Then release the lines and move the cable to
the computer (m-bootloader SPEC §4.3). This works even when the bootloader itself is broken.

Ready-made dongles:
- [FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter): Seeed XIAO RP2040 (recommended) or a
  Raspberry Pi Debug Probe; three wires (D+, D−, GND), **no VBUS** (the FM-1 runs on its battery). Also reads and
  writes the flash itself.
- [flipper-fm1-recovery](https://github.com/merthsoft/flipper-fm1-recovery): a Flipper Zero.
- Lunar Modulator's [USB_KEY dongle notes](https://github.com/ip2k/lunar-modulator/blob/main/docs/10-usb-key-dongle.md).

## The stock firmware

M-VAVE's V15 package is linked from [awesome-fm-1](https://github.com/cicloid/awesome-fm-1#official). Its SHA-256 is
`db1642b2b6fa5c2cccb11ffd13878068bb28601678d3644049f99dc40e7edb8a`. Check the hash of any copy you use. M-VAVE's
FM1 Open lists this file (`sdk/firmware/FM-1_015.fwsc`) as "distributed by M-VAVE under the Apache License,
Version 2.0", binary redistribution included, with the Apache-2.0 terms
([THIRD_PARTY.md](https://github.com/m-vave/fm1_open/blob/4c7496d/THIRD_PARTY.md) at 4c7496d). **[CODE]**

The tools that pin stock files by hash now accept **V15 and V16**: felucca-bsp's `fm1_rescue.py` (above), fm1-base's
installer (since 2026-11-05, [fce9ebb](https://github.com/charlesvestal/fm1-base/commit/fce9ebb)) and Felucca's web
installer and `fm1_install.py` (since 2026-11-15, [5044cce](https://github.com/hugelton/Felucca/commit/5044cce),
[Felucca#329](https://github.com/hugelton/Felucca/issues/329)). The V16 file they pin is the one below
(`FM-1_016`, 718,196 bytes, `d017fb65…`). Older copies of these tools accept only V15. **[CODE]**

**Since 2026-10-21 M-VAVE's download address serves V16, not V15** (awesome-fm-1; the resource hub). The V15 file
is kept in M-VAVE's FM1 Open repository (`sdk/firmware/FM-1_015.fwsc` at commit `b178ac4`, linked from
awesome-fm-1) and in the Internet Archive (linked from czietz's
[fm1-firmware-patcher](https://github.com/czietz/fm1-firmware-patcher)). A V15-only tool given today's download
will refuse it. **[CODE]** Checked for this wiki on 2026-10-29: the FM1 Open copy has the V15 hash above; the
download address gave a 718,196-byte file with SHA-256
`d017fb65e70aca97ee29d944470c3515a74581df5c0576bf70de917c7b2b61d5` (the resource hub lists another hash for V16,
starting `a21368e0`, so M-VAVE may have replaced the file again; check before you rely on either).

**V15 and V16 use the flash differently.** V16 writes 88 KiB more (its `flash.bin` ends at `0xA9000`, not
`0x93000`) and moves the app. Switching between V16 and Felucca erases Felucca's saved projects and samples
(Felucca#329, kurogedelic 2026-11-15, tested on an FM-1). **[HW]** **Going from V16 back to V15 leaves garbage presets
and wrong pitch, with any bootloader**: V16 overwrites the presets once it runs, and official updates don't restore
them; only a backup with the original area does (m-bootloader README). On one unit, V15 then played one note at a
time: V16 had changed V15's area `0xF2000`–`0xFBFFF`, though its own regions end at `0xECFFF`; V15's settings record
at `0xFBD80`–`0xFBE09` read `FF` bytes where V15's factory data has zeros, and restoring `0xFA000`–`0xFBFFF` from V15's
factory data made it polyphonic again ([fm1_open#16](https://github.com/m-vave/fm1_open/issues/16), ocean 2026-11-02,
open, no answer from M-VAVE). **[HW]** Keep a backup made **before** your first V16 install.

**Back on stock, presets look garbled** after Felucca-family firmwares, which keep their data inside the stock
settings range: use stock **System > Reset Patches** (TransitFM1 README; mvave-fm1). **Felucca 0.9-beta to 1.0.x**
could soft-brick on its old preset layout; the fix reported is erasing `0x97000` (length `0x8000`) and `0xFC000`
(length `0x2000`) in download mode, which deletes that user data (resource hub §1.3). **[CODE] [?]**

## Never do this

- Don't write the boot area (`[0, 0x4000)`) unless you are installing a bootloader with a tool made for it.
- Don't copy another unit's flash wholesale: the areas above `0x93000` can hold unit data (on the SMK-37,
  factory calibration). See [Flash Storage](Flash-Storage.md). A tool's "same unit" check is not proof (m-flasher#61, above).
- Don't connect VBUS from a dongle to the FM-1.

## Sources

- [m-bootloader](https://github.com/jvitkauskas/m-bootloader) README, SPEC.md §4.2, §4.3, §5.5, §11.6, §11.7
  (eb4f2c1, 2026-11-23); [v3.0 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0);
  [M-Flasher](https://github.com/jvitkauskas/m-flasher) README; [m-flasher#61](https://github.com/jvitkauskas/m-flasher/issues/61).
- [M-Unbricker](https://github.com/jvitkauskas/m-unbricker) README (MIT; 204f88d, 2026-11-26).
- Felucca `firmware/src/main.c` (ac8cc0e), [Felucca#320](https://github.com/hugelton/Felucca/issues/320),
  [#329](https://github.com/hugelton/Felucca/issues/329), [1.8.2 release notes](https://github.com/hugelton/Felucca/releases/tag/v1.8.2)
  (GPL-3.0-only); felucca-bsp `tools/fm1_rescue.py` (fdd709f; MPL-2.0).
- [fm1-base](https://github.com/charlesvestal/fm1-base) `device/main_fm1.c` (3039770), [fm1-base#36](https://github.com/charlesvestal/fm1-base/issues/36).
- [Optimist](https://github.com/w0ts/optimist) `docs/INSTALL-PATHS.md` (f7dc1b9, 2026-11-21).
- M-VAVE's [FM1 Open](https://github.com/m-vave/fm1_open) `THIRD_PARTY.md` (4c7496d), [fm1_open#16](https://github.com/m-vave/fm1_open/issues/16).
- [fm1-nes](https://github.com/Keitark/fm1-nes) `APP_UPDATES.md` (90d41c5).
- FM-1 resource hub [RESOURCES.md](https://github.com/aroum/fm1-custom-fw/blob/main/RESOURCES.md) (e0ae52c, 2026-11-24).
- [FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter) README;
  [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker).
- Felucca-family `tools/fm1_rescue.py`, `tools/fm1_rescue.sh` (GPL-3.0-only).
- [fm1-dub#14](https://github.com/sbegueria/fm1-dub/issues/14): M-Flasher installing a community `.fwsc` on a UC25HQ80 (2026-11-22).
- [MvaveFM1Unbricker README](https://github.com/Quixotic7/MvaveFM1Unbricker) (MIT), "does not show up as WL82
  UBOOT1.00" and `restore` sections.
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) firmware skill §5, §10 (MIT).
- [TransitFM1 README](https://github.com/Quixotic7/TransitFM1); [fm1-firmware-patcher README](https://github.com/czietz/fm1-firmware-patcher).
- [FM-1 resource hub REVERSE_ENGINEERING.md](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) §1.3.
- [awesome-fm-1](https://github.com/cicloid/awesome-fm-1), "Firmware update and recovery".

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
