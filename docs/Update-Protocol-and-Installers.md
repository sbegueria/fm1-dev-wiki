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

**The wrong MIDI port name.** Some units enumerate under another name and installers say "not found";
[fm-static](https://seajaysec.github.io/fm-static/) handles that.

## Sources

- Felucca's `src/ota.c`, `loader/loader.c`, `tools/fm1_install.py` (GPL-3.0-only).
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) OTA protocol and host-updater analysis.
- [m-bootloader](https://github.com/jvitkauskas/m-bootloader) README (M-UPGRADE behaviour).
- [fm1-base](https://github.com/charlesvestal/fm1-base) (`tools/fm1_install.py`, `web/installer.html`),
  [felucca-bsp](https://github.com/hugelton/felucca-bsp) commit history.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Contents](../README.md)
