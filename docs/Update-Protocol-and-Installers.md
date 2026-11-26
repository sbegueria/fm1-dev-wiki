---
title: "Update Protocol and Installers"
parent: Boot and install
nav_order: 3
---

# Update Protocol and Installers

How a firmware gets onto an FM-1 over a plain USB cable, without download mode. This is the path every
"install from the browser" page uses.

## The protocol (M-UPGRADE's)

The protocol was reverse-engineered from M-VAVE's updater and stock firmware
([FM-1-RE OTA protocol](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/11-ota-protocol.md)), and is
implemented by Felucca's update path (`src/ota.c`, `loader/`, `tools/fm1_install.py`, `web/fm1ota.js`).

**Framing [CODE]:** SysEx `F0 pack7(00 59 cmd len24 body chk) F7`. `pack7` packs 8-bit bytes into 7-bit bytes,
least significant bit first. The checksum is the complement of the byte sum over the body.

**The device pulls the package; the host only answers.** **[RE] [CODE]**

**[RE]** The read messages in more detail (resource hub §4.3, from FM-1-RE's protocol notes): on the wire a
request or answer starts `F0 00 32 41 41` (the packed form of `00 59 30`); `f1`, `addr` and `len` are 32-bit
little-endian values sent as four 7-bit groups; `len` carries `(length << 4) | flash type`; in a request `f1` is 0,
in an answer `f1 = length >> 4`. The answer's data is followed by a checksum byte,
`~(flash type + sum(data) + sum of the 4 address bytes + sum of the 3 length bytes) & 0xFF`. The final
"success?" request is address `0xF0000000`, length 8; the host answers `"success\0"`.

1. **Handshake.** The host asks for the identity (cmd `0x11`); the device answers with a string like `FM-1_900`
   (model, underscore, version) in a 27-byte body. The installer checks the model against the package's
   ([Firmware Package Format](Firmware-Package-Format.md)). Identities and their pitfalls: [below](#package-identities).
2. **Upgrade.** The host sends `F0 22 24 35 7F F7`.
3. **Step 1, the running firmware.** It asks for parts of the package with read requests (cmd `0x30`: an address
   in the logical image and a length, up to 512 bytes), checks them, and **stages the package's update loader**
   in a spare flash area (Felucca uses `0xE0000`). It then asks "success?" (`0xE0000000`). Only after the host
   confirms does it write the update record (`UPDATA_PARM`: in flash near the staging area, and at RAM
   `0x01C7FD88`) and reset. **Nothing is committed before the host's "success"**; any failure erases the staging
   area and the firmware carries on.
4. **The bootloader** sees the record and starts the staged loader from RAM (`0x01C0A800`).
5. **Step 2, the loader.** A small RAM program (Felucca's answers as `ota-FM-1_900`, USB `1209:0002`) reads the
   whole package again with read requests and writes the app area, then reports (`0xF0000000`) and restarts.
6. **Check.** The installer waits for the device to come back and reads its identity again.

Felucca's loader writes only `[0x4000, 0x93000)` and the update-record area, **never the head** (the
bootloader). It runs with interrupts off, polling USB at ~2 kHz and feeding the watchdog. **[CODE]** felucca-bsp's
loader (used by fm1-base) can also write a bigger app area, up to `0xE0000`, for a firmware that asks (`FL_APP_END`,
[13e4c44](https://github.com/hugelton/felucca-bsp/commit/13e4c44), 2026-11-21), and an app's resource and extra regions
after the app ([Firmware Package Format](Firmware-Package-Format.md)). **M-VAVE's own loader does write the head**: an
official V15 → V16 update replaced m-bootloader with the stock bootloader from V16's package
([Boot Chain and Bootloaders](Boot-Chain-and-Bootloaders.md)).
**[HW]**

**Where the loader is staged differs.** Felucca-family firmwares stage it at `0xE0000`. **M-VAVE's V15 stages it
at `0xE6000`**: a flash read after V15 installed TransitFM1 showed TransitFM1's loader there and `0xE0000…0xE4FFF`
erased (TransitFM1 `docs/SAFETY.md`, 2026-10-17). **[HW]** A firmware's rescue code that looks for a staged loader
only at `0xE0000` won't find V15's.

m-bootloader's SPEC §11.7 ([f1b6863](https://github.com/jvitkauskas/m-bootloader/commit/f1b6863), 2026-11-06) gives
the rule, found by offering packages that differ in one field: **[HW]**
- Stock firmware stores `ota.bin` so that it **ends where its update record begins**: `0xE7F00` on FM-1 V15
  (`0xBEF00` on the SMK-37's 016), with 20,224 bytes reserved before it. (A loader of about 8 KiB then starts near
  `0xE6000`, as TransitFM1 saw.)
- **V16 takes the place from the package's `VM` record** when the package has a `flash.bin`: it stores the loader to end
  `0x3000` below the end of the listed `VM`. With no `flash.bin` it used its own `VM` and ended at `0xD7000`; with one
  listing `VM` at `0x93000` + `0x56000` it ended at `0xE6000`, inside V16's own `SEQ64` area. Listing `VM` as zero stored
  nothing on V16, and on V15 reached `0xEA000`, over `BTIF`. So a package's region table decides where V16 writes.
- **What M-VAVE's firmware checks in `flash.bin` before it stages anything** (a refusal writes nothing): at `+0` a flash
  header with a correct CRC, file-system version `0x10`, block alignment 16, and `0.01` at `+4` and `AC791N_STORY` at
  `+16` as stored; a boot-area record named `app_dir_head`; and at `+0x4000` valid `app_area_head` and `VM` records.
  It is not the size.
- **Felucca-family firmware** (SLOOP, ChoralRoot, Melodee) stores an updater other than M-VAVE's only if the package
  lists a `flash.bin` and has the mark `FELUCCA-LOADER-1` among the first literals of its stream
  ([e1f4db7](https://github.com/jvitkauskas/m-bootloader/commit/e1f4db7)). **[CODE]** Felucca's own `fm1_install.py`
  looks for the same mark (`LOADER_MARK`). For M-VAVE's loader, which rewrites the head, Felucca checks the head: before
  1.7.0 it had to equal Felucca's own (an install of FM1 Open V14 stopped "after 2 requests", error `-5`); since 1.7.0 it
  accepts any well-formed package whose boot area the FM-1 can boot
  ([Felucca#296](https://github.com/hugelton/Felucca/issues/296), 2026-10-28;
  felucca-bsp [afd83ed](https://github.com/hugelton/felucca-bsp/commit/afd83ed)). **[CODE]**

### The order of the write, and the unprotected window

**[CODE]** V15, Felucca and most custom firmwares write the app area in address order. Once the first sector
(`0x4000`, where the bootloader jumps) is rewritten, no rescue code is left on the unit, so almost the whole
one-to-two-minute write is unprotected: a pulled cable or a flat battery can leave the FM-1 dark.

ChoralRoot's **boot stub** (flash `0x4120…0x4FFF`, documented in its `docs/BOOT-SAFETY.md`) and the loaders that
use it reverse this: the stub checks the firmware before starting it and starts update mode if the write was cut
off, and the loader writes the new firmware's **first sector last**. [TransitFM1](https://github.com/Quixotic7/TransitFM1)
carries the same stub, byte for byte, and measured on the host (power cut after every erase and every programmed
page, the real stub code judging the flash) that only the final sector, about 0.1 s, stays unprotected when
leaving for a stub-less firmware such as V15, and none between stub-carrying firmwares (its README, 0.1.3). **[CODE]**

**[HW]** Without a boot stub, the stock bootloader alone does the same when the first sector is blank. A cut
install that had already rewritten `0x4000` left a half-old, half-new app that the bootloader **started** (the
mixed app crashed; 2026-10-31, DUB-1 1.5.0 beta 1). DUB-1's update loader since 1.5.0 erases `0x4000` first (as
soon as any sector differs) and writes it last: a deliberate cut at about 35% (cable out and power off together,
2026-11-23) left `0x4000` erased; at power-on the bootloader ran the staged loader from the update record in
flash, which was back on USB as "Felucca Update" about 7 s later, and the installer resumed and finished. One
power cut, one unit (a Puya flash): the bootloader's rule (a blank app start: the flash update record) seen once.
DUB-1's loader also keeps the RAM update record armed while it writes, with a start counter (3 tries), so a reset
of the loader itself comes straight back (host-tested only).

A unit that comes back in update mode shows three short flashes of the key lights and a black screen, and
enumerates as `ota-FM-1` ("Felucca Update"). **Don't power-cycle it twice in a row in update mode**: a second
arrival with no normal boot in between drops into the ROM's USB boot (TransitFM1 README). **[CODE]**

m-bootloader's MIDI mode does the same (`0x4000` erased first and written last; SPEC §11.6). **[CODE]**

An update cut off by the FM-1's own flat battery can leave a unit with no USB device at all: the half-written
firmware hangs before its own safety code runs, and the bootloader does not fall back to the update loader at
power-on. Only a USB_KEY dongle helps then ([Recovery](Recovery.md); MvaveFM1Unbricker README). **[HW] [?]**

### The fast install (felucca-bsp's loader)

**[HW]** felucca-bsp's loader, from Henri David's pull request
[felucca-bsp#26](https://github.com/hugelton/felucca-bsp/pull/26) (merged 2026-11-05 as 028f5c1 and others), can write
only the sectors that changed, with 4 KiB reads, and resume a cut-off install:

1. Before step 2's upgrade message, the host sends **cmd `0x31`** with body `[1, 4096 as 16-bit LE]` (version 1, its
   largest read). A loader that knows it answers with its own read size. No answer within 0.5 s: the classic install.
2. The loader reads a **sector table** at `0xD0000000` + `flash.bin`'s offset: per 4 KiB sector, its CRC16 (the UFW's)
   and its CRC-32 (zlib), little-endian, 6 bytes a sector. Combined, the CRC16s must give `flash.bin`'s CRC16, or the
   install is refused (`-12`) before anything is erased.
3. A sector whose two CRCs already match the flash is kept and never sent. The others come as one 4 KiB read each, are
   erased as they arrive, checked, programmed and verified. A cut-off install resumes the same way.
4. The loader reports progress with **cmd `0x32`** `[sectors compared, total]` (16-bit LE each).

Hosts that speak it: felucca-bsp's and fm1-base's `tools/fm1_install.py` and `web/fm1ota.js` (fm1-base
[5ab0a19](https://github.com/charlesvestal/fm1-base/commit/5ab0a19), felucca-bsp
[628cbd0](https://github.com/hugelton/felucca-bsp/commit/628cbd0)), and Optimist's installers. They send no hello for a
package without the Felucca loader mark, so M-VAVE's loader never sees it. Measured on an FM-1: a full install's step 2
in 23.4 s (PR #26); classic 55 s and fast 12 s with nothing changed, and a kill at 21% resumed (charlesvestal,
2026-11-05). On the host, one package took 149 answers and 674,990 bytes, against 2,797 answers and 1,690,475 bytes
the classic way (PR #26).

**Note:** M-VAVE's FM1 Open uses the same command numbers, `0x31` and `0x32`, for something else (below). **[CODE]**

### Package identities

The identity answered in the handshake (`FM-1_` and a number) tells installers which firmware they talk to.
Numbers seen **[CODE]** (mvave-fm1's development skill, from the resource hub; the projects' sources as linked;
Optimist's `docs/INSTALL-PATHS.md`, 2026-11-21, for Baud Girl, Lunar and Optimist):

| Identity | Firmware |
| --- | --- |
| `FM-1_014`, `FM-1_015` | M-VAVE stock V14, V15 (also answered by TransitFM1, so every installer writes over it) |
| `FM-1_016` | M-VAVE stock V16 (Felucca `web/fm1pkg.js`, 5044cce) |
| `FM-1_020` … `FM-1_098` | Baud Girl |
| `FM-1_099` | m-bootloader's M-UPGRADE installer packages (labelled 099 because M-UPGRADE refuses the version a unit has) |
| `FM-1_5NN` | Lunar |
| `FM-1_700`, `FM-1_720` … `769` | Optimist's own firmware |
| `FM-1_8000012` … `FM-1_8000014` | M-VAVE's FM1 Open V12 to V14 (`app/version.h`; its packager takes only `FM-1_8` + 6 digits) |
| `FM-1_8` + 6 digits | foma built with `"platform": "mvave"` (on the FM1 Open SDK; foma README) |
| `FM-1_900` | SLOOP, ORBIT |
| `FM-1_91N` | Felucca (1.8.x: `FM-1_918`) |
| `FM-1_93N`, `FM-1_97N`, `FM-1_98N`, `FM-1_99N` | sloopDX, zp12, Doom, Dinghy (Dinghy 0.Y: `FM-1_99Y`) |
| `FM-1_982` | OpenIFW-fm1 |
| `FM-1_000` | ORBIT's rescue; Optimist's USB rescue; an fm1-base development build with the default prefix |
| `ota-FM-1_900`, `ota-SMK-37 Pro_900` | Felucca-family update loaders, and m-bootloader's MIDI mode |
| fm1-base: `id_prefix` + `XXYYZZ` | each fm1-base firmware's own scheme (below) |

**fm1-base's scheme** (from its `tools/build.py` and `DESIGN.md` "Build", 3039770): each firmware picks an `id_prefix`
in its `fm1.toml`, `FM-1_` and digits (default `FM-1_0`; the template uses `FM-1_11`). A development build answers the
prefix + `00`; release X.Y.Z answers the prefix + `XXYYZZ`, two digits each. Examples: CTL-1 1.0 `FM-1_6010000`;
RemoteFM1 uses prefix `52`. ("FM-1_0 + 6 digits" in Optimist's notes is the default prefix only.)

**M-UPGRADE 3.11's limit, and the short answer.** M-UPGRADE 3.11 (on m-vave.com since 2026-11-03) accepts an identity
only as a name, `_` and 1 to 3 digits, and refuses the rest as if the unit did not answer: CTL-1 `FM-1_6010000`,
PERF-1 `FM-1_7001608` and FM1 Open `FM-1_8000014` were all refused
([fm1_open#20](https://github.com/m-vave/fm1_open/issues/20), charlesvestal 2026-11-05, open). **[HW]** So felucca-bsp's
`chip/ota.c` (used by fm1-base) answers the plain `0x11` with the name, `_` and the **first three digits only**
(`FM-1_6010000` → `FM-1_601`), and the whole identity only to `0x11` with the one-byte body `01`. fm1-base's installers
ask that way when the plain answer is three digits. The scheme is PERF-1's. Update loaders always answer whole.
**[CODE]** Short answers can collide: PERF-1's would be `FM-1_700`, which Optimist uses too. **[?]**

**Bytes 24–26 of the identity reply** can carry the flash's JEDEC ID (felucca-bsp
[#29](https://github.com/hugelton/felucca-bsp/pull/29), Felucca 1.8.2; 0 = not read). fm1-base tried it and took it
out again on 2026-11-06: The Firmware Hub's installer then found "No FM-1" (it stripped only trailing zeros), while
M-UPGRADE 3.11 and the Felucca-family installers read the text to its first zero
([3ae2a65](https://github.com/charlesvestal/fm1-base/commit/3ae2a65)). **[HW]**

Pick your own identity and keep the `FM-1_` form: installers resume a cut-off install only with loaders they
recognise. There is no registry; check the hub before choosing. **[?]**

### A firmware's side of the bargain

To be installable by the community's installers, a firmware should:
- answer the identity handshake and the upgrade message;
- run step 1 itself, or hand over to a loader that does (Felucca's `ota.c` takes hooks from the app: send a
  SysEx, get the next SysEx, time, idle, flash erase/program/read, show progress, commit);
- answer the soft key `F0 22 24 35 7D F7` with download mode ([MIDI and USB](MIDI-and-USB.md));
- offer a way into download mode from the keys, in case its UI or USB is broken (Felucca: OCT− + OCT+ for 5 s;
  fm1-base: REC at power-on; both reset into the ROM's download mode, [Recovery](Recovery.md)).

### Reading the flash over MIDI, and a trap

m-bootloader's MIDI mode and RAM stage answer read-only host commands, in the same frame (SPEC §11.6,
[bced20b](https://github.com/jvitkauskas/m-bootloader/commit/bced20b)): `0x12` (who is answering: `MBOOT`, kind,
chip key, capacity, JEDEC ID, the flash's unique ID, version), `0x23` (read up to 512 bytes of raw flash), `0x24`
(the chip's CRC-32 of a range), `0x25` (load and run a RAM app), and `F0 22 24 35 7E F7` (restart). **[CODE]**

**[HW]** **Stock firmware's update service uses the same frame, and on some versions its `0x21` erases a flash sector
with no check.** The frame's checksum does not cover the command byte, and `0x21` and `0x23` differ by one bit. A host
must send `0x12` first to learn what is answering, and never build these commands from data (SPEC §11.6).

## Installers

| Installer | Platform | Notes |
| --- | --- | --- |
| M-UPGRADE (M-VAVE) | Windows, macOS | the official updater; refuses the version a unit already has. 3.11 (since 2026-11-03) accepts only identities of 1–3 digits (above). On macOS it rejected the whole scan while a MIDI device with two ports was plugged in ([fm1_open#18](https://github.com/m-vave/fm1_open/issues/18), pfkellogg 2026-11-04) |
| [m-bootloader](https://github.com/jvitkauskas/m-bootloader)'s MIDI mode | on the unit | not an installer but a target: browser installers (Felucca's, SLOOP's, Melodee's, ChoralRoot's) install through it, from a phone too; Baud Girl's refuses it (README) |
| Project web pages (Felucca family, fm1-base, …) | Chrome or Edge with Web MIDI SysEx | most common for open firmware |
| `tools/fm1_install.py` (Felucca, felucca-bsp, fm1-base) | Python, mido + python-rtmidi | finishes a cut-off install if the unit is still in update mode |
| [fm1-linux-update](https://github.com/fuleo/fm1-linux-update) | Linux | stock V14 → V15 |
| [FM1Kit](https://github.com/ocean/FM1Kit) | iOS, macOS (Swift) | not widely tested (its README) |
| [Slipway](https://github.com/Discodave-Glitch/slipway) | Windows | lists firmwares, keeps stock V15 as the way back |
| ORBIT's `tools/fm1_install.py` | Python | the Felucca installer with exit codes: 0 done, 1 cancelled or error, 2 bad arguments or package, 3 FM-1 not found, 4 connection lost, 5 timeout, 6 wrong model or foreign loader (mvave-fm1 firmware skill §5.3) |
| [TransitFM1](https://github.com/Quixotic7/TransitFM1) | web, Python | a silent stopover firmware between two custom firmwares (below) |
| [FM-1-RE](https://github.com/AL-255/FM-1-RE) `tools/fm1_ota.py` | Python | the reverse-engineered protocol, byte-exact (per the resource hub) |
| [M-Flasher](https://github.com/jvitkauskas/m-flasher) | Windows, macOS, Linux | not this protocol: writes through download mode ([Recovery](Recovery.md)) |
| [fm1-nes](https://github.com/Keitark/fm1-nes) `scripts/jltool_update.py` | Windows | not this protocol: app-only updates through download mode and jl-uboot-tool; only onto stock 010/V14 layouts and the Puya flash; "not hardware-qualified" beyond one V14 bench install (51 sectors changed, 2026-10-03; `APP_UPDATES.md`) |

## Known problems

**M-VAVE's FM1 Open blocks other installers.** FM1 Open's updater only accepts a package it was armed for. A
unit on FM1 Open gives confusing errors with Felucca-style installers. fm1-base's installer detects it and says
to put M-VAVE's stock firmware (V15) back first; FM1 Open's own workbench can return to V15. **[CODE]** How FM1 Open
arms an update (its `sdk/WebUpdate.js`, 4c7496d): **[CODE]**
- `0x11`, the identity (27 bytes), then **`0x31`**: a 47-byte capability block starting `FMC1`, with the family
  (FM-1 7; SMK-37 Pro 8, Elite 9 in SMK37 Open), a state byte and a 32-byte build hash.
- **`0x32`** arms the update with the family, the image size and the package's build hash; then the usual
  `F0 22 24 35 7F F7`.
- Its loader answers `ota-FM1C_` + 16 hex digits of the build hash; M-VAVE's stock loader answers `ota-FM-1_015`.
- Its return to stock accepts only the V15 file by SHA-256 (`db1642b2…`) and can restore V15's factory voices.

The other way round is fixed: Felucca up to 1.6.1 refused FM1 Open's package (its head check); since 1.7.0 it accepts
any well-formed package (Felucca#296, above).

**Installs that stop half-way.** Reported with Felucca-family installers: "the loader stopped answering after
562 requests" (about half the package), a blank screen, then "FM-1 not found"; unplugging, replugging and
installing again finished it (DUB-1, October 2026). The upstream installers have the same error path and no
automatic reconnect. The cause (a USB drop, the loader's poll falling behind) is not known. **[HW] [?]**
- A progress screen in the loader helps users tell a stall from a dead link: felucca-bsp's loader now shows
  "UPDATING / DO NOT UNPLUG" and a bar (2026-10-23).
- If a write was cut off, the unit usually stays in update mode on the next start: run the installer again.

**Custom over custom.** Installing one custom firmware straight over another has bricked units. The common
advice is to go through stock V15 first, but V15's own write is in address order, so the step out of V15 is the
risky one. [TransitFM1](https://github.com/Quixotic7/TransitFM1) (GPL-3.0) is a stopover that answers as V15,
carries ChoralRoot's boot stub and writes the next firmware itself, first sector last. **TransitFM1 0.1.2 was
withdrawn**: it let Baud Girl's installer write an image larger than its loader writes, and the unit was left dark;
0.1.3 refuses such packages (TransitFM1 README, tested on a unit 2026-10-18). For Baud Girl, go through V15. **[HW]**

**V16 and the V15 download.** M-VAVE's firmware download address now serves **V16** (2026-10-21). Older tools that
accept only the V15 file by hash will refuse it; current Felucca, fm1-base and felucca-bsp tools take V15 and V16
([Recovery](Recovery.md)). V15 is kept in M-VAVE's FM1 Open repository at `sdk/firmware/FM-1_015.fwsc` (commit
`b178ac4`, linked from awesome-fm-1). **What is now known about V16** **[HW]**:
- It answers **`FM-1_016`** (Felucca `web/fm1pkg.js`; fm1_open#16).
- The community's installers work over it: Felucca 1.8.2 → V16 → Felucca → V16 on an FM-1
  ([Felucca#329](https://github.com/hugelton/Felucca/issues/329), kurogedelic 2026-11-15); felucca-bsp's `hwtest` installed
  over stock V16 with `fm1_install.py` ([felucca-bsp#17](https://github.com/hugelton/felucca-bsp/issues/17),
  charlesvestal 2026-10-31); M-Unbricker's helper started through V16's update route (its README).
- Where it stages its loader depends on the package (above).
- Whether V16 answers the soft key `F0 22 24 35 7D F7` is not written down in the sources checked here. **[?]**

**The wrong MIDI port name.** Some units enumerate under another name and installers say "not found";
[fm-static](https://seajaysec.github.io/fm-static/) handles that.

## Sources

- Felucca's `src/ota.c`, `loader/loader.c`, `tools/fm1_install.py` (GPL-3.0-only).
- [DUB-1](https://github.com/sbegueria/fm1-dub) `firmware/loader/ldr_core.c` (from 1.5.0), `tests/ldr_test.c`; the cut-install test, 2026-11-23.
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) OTA protocol and host-updater analysis.
- [m-bootloader](https://github.com/jvitkauskas/m-bootloader) README (M-UPGRADE behaviour).
- [FM-1 resource hub REVERSE_ENGINEERING.md](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) §4.
- [TransitFM1](https://github.com/Quixotic7/TransitFM1) README and `docs/SAFETY.md` (GPL-3.0; 0.1.3, 2026-10-18).
- [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker) README (flat-battery case).
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) skills (identities, ORBIT installer exit codes);
  [RemoteFM1](https://github.com/Quixotic7/RemoteFM1) README; [awesome-fm-1](https://github.com/cicloid/awesome-fm-1) "Official".
- [fm1-base](https://github.com/charlesvestal/fm1-base) (`tools/fm1_install.py`, `web/installer.html`, `tools/build.py`,
  `DESIGN.md`, at 3039770; commits 5ab0a19, 3ae2a65),
  [felucca-bsp](https://github.com/hugelton/felucca-bsp) (`chip/ota.c` at fdd709f; PR #26, #29; issue #17; commits
  628cbd0, 13e4c44, afd83ed).
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §11.6, §11.7 (eb4f2c1) and README.
- [Felucca](https://github.com/hugelton/Felucca) issues #296, #329; commit 5044cce.
- M-VAVE's [FM1 Open](https://github.com/m-vave/fm1_open) `sdk/WebUpdate.js`, `app/version.h` (4c7496d); issues
  [#18](https://github.com/m-vave/fm1_open/issues/18), [#20](https://github.com/m-vave/fm1_open/issues/20);
  [SMK37 Open](https://github.com/m-vave/smk37_open) `sdk/WebUpdate.js` (b2d6bcb).
- [Optimist](https://github.com/w0ts/optimist) `docs/INSTALL-PATHS.md` (f7dc1b9); [foma](https://github.com/konsumer/foma) README (885100f);
  [Dinghy](https://github.com/zvenson/dinghy) CHANGELOG (14cda7a); [fm1-nes](https://github.com/Keitark/fm1-nes) `APP_UPDATES.md` (90d41c5).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
