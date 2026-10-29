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
   (model, underscore, version). The installer checks the model against the package's ([Firmware Package Format](Firmware-Package-Format.md)).
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
bootloader). It runs with interrupts off, polling USB at ~2 kHz and feeding the watchdog. **[CODE]**

**Where the loader is staged differs.** Felucca-family firmwares stage it at `0xE0000`. **M-VAVE's V15 stages it
at `0xE6000`**: a flash read after V15 installed TransitFM1 showed TransitFM1's loader there and `0xE0000…0xE4FFF`
erased (TransitFM1 `docs/SAFETY.md`, 2026-10-17). **[HW]** A firmware's rescue code that looks for a staged loader
only at `0xE0000` won't find V15's.

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

A unit that comes back in update mode shows three short flashes of the key lights and a black screen, and
enumerates as `ota-FM-1` ("Felucca Update"). **Don't power-cycle it twice in a row in update mode**: a second
arrival with no normal boot in between drops into the ROM's USB boot (TransitFM1 README). **[CODE]**

An update cut off by the FM-1's own flat battery can leave a unit with no USB device at all: the half-written
firmware hangs before its own safety code runs, and the bootloader does not fall back to the update loader at
power-on. Only a USB_KEY dongle helps then ([Recovery](Recovery.md); MvaveFM1Unbricker README). **[HW] [?]**

### Package identities

The identity answered in the handshake (`FM-1_` and a number) tells installers which firmware they talk to.
Numbers seen (mvave-fm1's development skill, from the resource hub; fm1-base and RemoteFM1 READMEs) **[CODE]**:

| Identity | Firmware |
| --- | --- |
| `FM-1_014`, `FM-1_015` | M-VAVE stock V14, V15 (also answered by TransitFM1, so every installer writes over it) |
| `FM-1_900` | SLOOP, ORBIT |
| `FM-1_91N` | Felucca |
| `FM-1_93N`, `FM-1_97N`, `FM-1_98N`, `FM-1_99N` | sloopDX, zp12, Doom, Dinghy |
| `FM-1_982` | OpenIFW-fm1 |
| `FM-1_000` | ORBIT's rescue |
| `FM-1_52XXYYZZ` | fm1-base's scheme (release X.Y.Z), e.g. RemoteFM1 0.2 |

Pick your own and keep the `FM-1_` form: installers resume a cut-off install only with loaders they recognise.
There is no registry; check the hub before choosing. **[?]** What V16 answers has not been written down here.

### A firmware's side of the bargain

To be installable by the community's installers, a firmware should:
- answer the identity handshake and the upgrade message;
- run step 1 itself, or hand over to a loader that does (Felucca's `ota.c` takes hooks from the app: send a
  SysEx, get the next SysEx, time, idle, flash erase/program/read, show progress, commit);
- answer the soft key `F0 22 24 35 7D F7` with download mode ([MIDI and USB](MIDI-and-USB.md));
- offer an update mode from the keys, in case its UI is broken (Felucca: OCT− + OCT+ for 5 s; fm1-base: REC at
  power-on).

## Installers

| Installer | Platform | Notes |
| --- | --- | --- |
| M-UPGRADE (M-VAVE) | Windows, macOS | the official updater; refuses the version a unit already has |
| Project web pages (Felucca family, fm1-base, …) | Chrome or Edge with Web MIDI SysEx | most common for open firmware |
| `tools/fm1_install.py` (Felucca, felucca-bsp, fm1-base) | Python, mido + python-rtmidi | finishes a cut-off install if the unit is still in update mode |
| [fm1-linux-update](https://github.com/fuleo/fm1-linux-update) | Linux | stock V14 → V15 |
| [FM1Kit](https://github.com/ocean/FM1Kit) | iOS, macOS (Swift) | not widely tested (its README) |
| [Slipway](https://github.com/Discodave-Glitch/slipway) | Windows | lists firmwares, keeps stock V15 as the way back |
| ORBIT's `tools/fm1_install.py` | Python | the Felucca installer with exit codes: 0 done, 1 cancelled or error, 2 bad arguments or package, 3 FM-1 not found, 4 connection lost, 5 timeout, 6 wrong model or foreign loader (mvave-fm1 firmware skill §5.3) |
| [TransitFM1](https://github.com/Quixotic7/TransitFM1) | web, Python | a silent stopover firmware between two custom firmwares (below) |
| [FM-1-RE](https://github.com/AL-255/FM-1-RE) `tools/fm1_ota.py` | Python | the reverse-engineered protocol, byte-exact (per the resource hub) |
| [M-Flasher](https://github.com/jvitkauskas/m-flasher) | Windows, macOS, Linux | not this protocol: writes through download mode ([Recovery](Recovery.md)) |

## Known problems

**M-VAVE's FM1 Open blocks other installers.** FM1 Open's updater only accepts a package it was armed for. A
unit on FM1 Open gives confusing errors with Felucca-style installers. fm1-base's installer detects it and says
to put M-VAVE's stock firmware (V15) back first; FM1 Open's own workbench can return to V15. **[CODE]**

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

**V16 and the V15 download.** M-VAVE's firmware download address now serves **V16** (2026-10-21). Tools that
accept only the V15 file by hash (e.g. Felucca-family `fm1_rescue`) will refuse it; V15 is kept in M-VAVE's FM1
Open repository at `sdk/firmware/FM-1_015.fwsc` (commit `b178ac4`, linked from awesome-fm-1). Whether the
community's installers and the soft key work with V16 is in [Open Questions](Open-Questions.md). **[?]**

**The wrong MIDI port name.** Some units enumerate under another name and installers say "not found";
[fm-static](https://seajaysec.github.io/fm-static/) handles that.

## Sources

- Felucca's `src/ota.c`, `loader/loader.c`, `tools/fm1_install.py` (GPL-3.0-only).
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) OTA protocol and host-updater analysis.
- [m-bootloader](https://github.com/jvitkauskas/m-bootloader) README (M-UPGRADE behaviour).
- [FM-1 resource hub REVERSE_ENGINEERING.md](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) §4.
- [TransitFM1](https://github.com/Quixotic7/TransitFM1) README and `docs/SAFETY.md` (GPL-3.0; 0.1.3, 2026-10-18).
- [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker) README (flat-battery case).
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) skills (identities, ORBIT installer exit codes);
  [RemoteFM1](https://github.com/Quixotic7/RemoteFM1) README; [awesome-fm-1](https://github.com/cicloid/awesome-fm-1) "Official".
- [fm1-base](https://github.com/charlesvestal/fm1-base) (`tools/fm1_install.py`, `web/installer.html`),
  [felucca-bsp](https://github.com/hugelton/felucca-bsp) commit history.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
