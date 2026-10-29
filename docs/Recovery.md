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

## The ladder: from easiest to hardest

| The unit… | Try | Tag |
| --- | --- | --- |
| starts, but the firmware is broken or unwanted | its update mode (Felucca and forks: hold OCT− and OCT+ for 5 s; fm1-base: REC at power-on), then install stock V15 or another firmware | [CODE] |
| starts but crashes at once, twice | Felucca-family firmwares fall into a **safe mode** (no audio, USB on) after two failed starts: install from there | [CODE] |
| has m-bootloader | hold **REC** at power-on (download mode) or **PLAY** (recovery menu: boot, download mode, boot anyway, wipe user data) | [HW] |
| shows on USB as `4C4A:8057` / `WL80UBOOT1.00` | it is already in ROM download mode: use M-Flasher, [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker), or `fm1_rescue` | [HW] |
| shows nothing on USB, but the firmware is crashing | without hardware (MvaveFM1Unbricker README): **leave it on for two minutes** (Felucca-family watchdog and boot-loop guard enter ROM boot after two failed boots); **hold OCT− and OCT+ at power-on for 10 s** (update mode, if the firmware reaches its main loop); **three quick restarts** (on 10 s, off, three times: three crashed boots land in ROM boot, which is also how a unit can get there by accident) | [CODE] |
| shows nothing at all on USB | force download mode with a **USB_KEY dongle** (below) | [HW] |

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

## How the download-mode tools write

[FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter), MvaveFM1Unbricker, `fm1_rescue` and
M-Flasher follow the same safety rules, which were checked on hardware:

1. A flash loader runs in RAM (JieLi's `wl82loader.bin` from [jl-uboot-tool](https://github.com/kagaimiq/jl-uboot-tool),
   checked against its hash; or m-bootloader's own MIT flash loader in M-Flasher).
2. The chip key must be `0x980F` and the flash `856014` (1 MiB), or nothing is done.
3. The whole flash is read and saved first.
4. Only `[0x4000, 0x93000)` is written, and only the 4 KiB sectors that differ; each is erased, written and read
   back. The boot area below `0x4000` and the device data above `0x93000` are not touched.
5. A final full read must match the expected image.

(From `fm1_rescue.py`'s header, which credits FM-1 Transporter's `docs/PROTOCOL.md`.)

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

M-VAVE's V15 package is linked from [awesome-fm-1](https://github.com/cicloid/awesome-fm-1#official).
Felucca-family `fm1_rescue` accepts only that exact file, by SHA-256
`db1642b2b6fa5c2cccb11ffd13878068bb28601678d3644049f99dc40e7edb8a`. Check the hash of any copy you use.

**Since 2026-10-21 M-VAVE's download address serves V16, not V15** (awesome-fm-1; the resource hub). The V15 file
is kept in M-VAVE's FM1 Open repository (`sdk/firmware/FM-1_015.fwsc` at commit `b178ac4`, linked from
awesome-fm-1) and in the Internet Archive (linked from czietz's
[fm1-firmware-patcher](https://github.com/czietz/fm1-firmware-patcher)). A V15-only tool given today's download
will refuse it. **[CODE]**

**Back on stock, presets look garbled** after Felucca-family firmwares, which keep their data inside the stock
settings range: use stock **System > Reset Patches** (TransitFM1 README; mvave-fm1). **Felucca 0.9-beta to 1.0.x**
could soft-brick on its old preset layout; the fix reported is erasing `0x97000` (length `0x8000`) and `0xFC000`
(length `0x2000`) in download mode, which deletes that user data (resource hub §1.3). **[CODE] [?]**

## Never do this

- Don't write the boot area (`[0, 0x4000)`) unless you are installing a bootloader with a tool made for it.
- Don't copy another unit's flash wholesale: the areas above `0x93000` can hold unit data (on the SMK-37,
  factory calibration). See [Flash Storage](Flash-Storage.md).
- Don't connect VBUS from a dongle to the FM-1.

## Sources

- [m-bootloader](https://github.com/jvitkauskas/m-bootloader) README, SPEC.md §4.2, §4.3, §5.5;
  [M-Flasher](https://github.com/jvitkauskas/m-flasher) README.
- [FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter) README;
  [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker).
- Felucca-family `tools/fm1_rescue.py`, `tools/fm1_rescue.sh` (GPL-3.0-only).
- [MvaveFM1Unbricker README](https://github.com/Quixotic7/MvaveFM1Unbricker) (MIT), "does not show up as WL82
  UBOOT1.00" and `restore` sections.
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) firmware skill §5, §10 (MIT).
- [TransitFM1 README](https://github.com/Quixotic7/TransitFM1); [fm1-firmware-patcher README](https://github.com/czietz/fm1-firmware-patcher).
- [FM-1 resource hub REVERSE_ENGINEERING.md](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) §1.3.
- [awesome-fm-1](https://github.com/cicloid/awesome-fm-1), "Firmware update and recovery".

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
