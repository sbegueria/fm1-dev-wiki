---
title: "MIDI and USB"
parent: Hardware
nav_order: 8
---

# MIDI and USB

## The ports

| Port | Direction | How | Tag |
| --- | --- | --- | --- |
| USB-MIDI | in and out | USB0 full-speed device, class compliant | [HW] |
| TRS MIDI | **in** | UART1 RX on PH8, 31250 baud, RX by DMA into a ring | [CODE] |
| TRS MIDI out | ? | M-VAVE's FM1 Open lists only USB and BLE as outputs. No known open firmware sends on the TRS jack. | [?] |
| BLE MIDI | in and out | JieLi's closed Bluetooth stack; in fm1-base (`ble = true`) | [HW] |

Felucca's UART header says its MIDI-in driver was "untested on hardware" when written; fm1-base lists MIDI "over
USB and TRS" in its platform. If you have tested TRS input on a given firmware, add it here.

## USB identities seen on the FM-1

| VID:PID | Name | Who | Tag |
| --- | --- | --- | --- |
| (M-VAVE's) | "FM-1 MIDI 1" (and an "FM-1 Audio" UAC1 function) | stock firmware | [RE] [HW] |
| `1209:0001` | the firmware's name | Felucca and forks (pid.codes **test** ID) | [CODE] |
| `1209:0002` | "Felucca Update" | Felucca's update loader | [CODE] |
| `4C4A:8057` | `WL80UBOOT1.00` / `UBOOT1.00`, mass storage | the chip's mask-ROM download mode | [HW] |

`1209:0001` is pid.codes' shared test PID. A firmware that wants its own identity can apply to
[pid.codes](https://pid.codes) for one.

**Installers must not probe every MIDI port they see.** Felucca's installer only opens ports whose names match the
FM-1 or known firmware names, so it never sends SysEx to other gear. Some FM-1s show up under unexpected names;
[fm-static](https://seajaysec.github.io/fm-static/) exists for that case.

## SysEx messages to know

| Message | Meaning | Tag |
| --- | --- | --- |
| `F0 22 24 35 7D F7` | **Soft key**: the running firmware enters the ROM's download mode (writes `usb_update_mode` at `0x01C7FD80` and resets). Stock firmware (FM-1 015; SMK-37 016) and Felucca answer it. M-Flasher's DOWNLOAD MODE button sends it. | [HW] |
| `F0 22 24 35 7F F7` | **Upgrade**: start an M-UPGRADE update session | [RE] [CODE] |
| `F0 00 32 45 00 00 00 40 7F F7` | handshake query used by Felucca's installer | [CODE] |
| `F0 43 0n …` | Yamaha DX7 voice and bank SysEx (stock firmware) | [RE] |

The update session's framing is on [Update Protocol and Installers](Update-Protocol-and-Installers.md). Any firmware that wants to be updatable
by the community's installers should answer the soft key and the upgrade message.

**On a stock unit with the stock bootloader, the soft key is the only way into download mode without hardware.**
That's why M-Flasher can back up a stock unit before you change anything.

## USB implementation notes (Felucca family)

**[CODE]**
- The USB controller is a MUSB-style SIE. Felucca's driver polls it from the 10 kHz timer interrupt at 2 kHz; no
  USB interrupt is used. Everything that touches the controller's INDEX register runs in one place, so nothing
  can interleave.
- MIDI in goes into a ring that the audio interrupt drains; MIDI out comes from a ring the audio interrupt fills.
  The audio code never waits on USB.
- If the USB poll falls inside the audio render, it waits for the next tick outside it, because both share the
  MIDI queues.
- Optional functions: a CDC-ACM serial port (a console during development) and a UAC1 audio input.
- **USB host** (for plugging a USB MIDI keyboard into the FM-1) is ported in felucca-bsp from Melodee, opt-in, and
  needs **external power**: the FM-1's port does not supply VBUS. Input has run on hardware; the BSP port and the
  output path still need validation (felucca-bsp `docs/EXPERIMENTAL.md`).

## Sources

- Felucca's `src/usb.c`, `src/ota.c`, `hal/fm1_usb.h`, `hal/fm1_uart.h`, `tools/fm1_install.py` (GPL-3.0-only).
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §4.2.
- [FM1 Open README](https://github.com/m-vave/fm1_open) (MIDI section).
- [FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §1, §5.
- [felucca-bsp docs/EXPERIMENTAL.md](https://github.com/hugelton/felucca-bsp/blob/main/docs/EXPERIMENTAL.md).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
