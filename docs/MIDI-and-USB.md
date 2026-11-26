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
| TRS MIDI | **in** | UART1 RX on PH8, 31250 baud, RX by DMA into a ring | [CODE] [HW] |
| TRS MIDI out | ? | M-VAVE's FM1 Open lists only USB and BLE as outputs. No known open firmware sends on the TRS jack. The resource hub says the jack's circuit only passes MIDI **in** (an input buffer), so there is no TRS out without a hardware change. The hub now also lists a DIN MIDI out mod that it says works with a Dinghy 0.3.0 fork; unchecked ([Pinout](Pinout.md)). Stock V13 does have a UART transmit path: it copies incoming MIDI into the UART's TX DMA (`0x12114`/`0x12118`) as MIDI thru (FM-1-RE `io/05-midi.md` §1, §2.1, marked high) **[RE]**. Whether anything reaches the jack is the circuit question above. | [?] |
| BLE MIDI | in and out | JieLi's closed Bluetooth stack, or fm1-base's own (`ble = true`, `ble_stack = "own"`) | [HW] |

Felucca's UART header first said its MIDI-in driver was "untested on hardware". Since Felucca 1.0 (2026-10-05,
[727f272](https://github.com/hugelton/Felucca/commit/727f272)) it says "RX verified on hardware", and 1.0 added TRS
MIDI in, clock, bend and sustain. felucca-bsp's `chip/fm1_uart.h` carries the same line. Dinghy's older copy
still says "Untested on hardware". fm1-base lists MIDI "over USB and TRS" in its platform.

**BLE MIDI notes (fm1-base).** **[HW]** Links to some centrals (a CME WIDI Master at a 7.5 ms interval) dropped
within seconds, and a dead link stayed "connected"; three fixes in the own stack's receive window and supervision
timer held a link for 10+ minutes ([0eec347](https://github.com/charlesvestal/fm1-base/commit/0eec347), from
Jeremy Murray-Wakefield, [fm1-base#34](https://github.com/charlesvestal/fm1-base/issues/34), 2026-11-04). As a
central, a Roland FP-30X refused notifications until the link was paired, and once more after encryption; fm1-base
pairs and asks again ([PR #44](https://github.com/charlesvestal/fm1-base/pull/44) on JieLi's stack,
[PR #45](https://github.com/charlesvestal/fm1-base/pull/45) on its own, 2026-11-08 and 18). Still open on
2026-11-26: a device can't connect while a computer is connected
([fm1-base#10](https://github.com/charlesvestal/fm1-base/issues/10)).

**Stock BLE MIDI (V13).** **[RE]** FM-1-RE reads the stock GATT database: the standard BLE-MIDI service, its
characteristic value at handle `0x72` (notify, and write without response), beside two JieLi vendor services. The
default connection interval runs from 33.75 to 410 ms, and the app asks for the fast end
([`io/10-bluetooth.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/10-bluetooth.md) §1, §4, §6, marked med).

## USB identities seen on the FM-1

| VID:PID | Name | Who | Tag |
| --- | --- | --- | --- |
| `4C4A:C755` | "FM-1 MIDI 1" (and an "FM-1 Audio" UAC1 function) | stock firmware (ID seen by mvave-fm1 on a V15 unit, 2026-10-22) | [RE] [HW] |
| `4D4A:4155` | `ota-FM-1` | **M-VAVE's own update loader**: a capture saw it enumerate this way and answer `ota-FM-1_015` between step 1 (48 reads) and step 2 (1,167 reads) of a stock V15 install (FM-1-RE `io/12-v15-reflash-proof.md`, one unit, 2026-09-04) | [HW] |
| `1209:0001` | the firmware's name | Felucca and forks (pid.codes **test** ID) | [CODE] |
| `1209:0002` | "Felucca Update" | Felucca's update loader | [CODE] |
| `1209:0002` | "FM-1 Update" (answers as `ota-FM-1_900`) | m-bootloader's MIDI mode (SPEC §11.6) | [CODE] |
| `4C4A:8057` | `WL80UBOOT1.00` / `UBOOT1.00`, mass storage; the disk shows as `WL82 UBOOT1.00` (SCSI vendor `WL82`, product `UBOOT1.00`: m-bootloader SPEC §4.2; seen so on an SMK-37 Pro, fm1-to-smk37 02 §3.1) | the chip's mask-ROM download mode | [HW] |

**[RE]** In stock V13 the device descriptor template has `4C4A:4155`, not the `C755` seen on units; where `C755` comes
from is not found (FM-1-RE [`io/06-usb.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/06-usb.md) §1). Its
USB serial number is 16 hex digits made from 8 bytes at `0x01C20190`, which FM-1-RE calls the chip ID (§4.4, marked
high). Its "FM-1 Audio" function is playback, from the computer ([Audio Output](Audio-Output.md#usb-audio)).

`1209:0001` is pid.codes' shared test PID. A firmware that wants its own identity can apply to
[pid.codes](https://pid.codes) for one.

**Installers must not probe every MIDI port they see.** Felucca's installer only opens ports whose names match the
FM-1 or known firmware names, so it never sends SysEx to other gear. Some FM-1s show up under unexpected names;
[fm-static](https://seajaysec.github.io/fm-static/) exists for that case.

## SysEx messages to know

| Message | Meaning | Tag |
| --- | --- | --- |
| `F0 22 24 35 7D F7` | **Soft key**: the running firmware enters the ROM's download mode (writes `usb_update_mode` at `0x01C7FD80` and resets). Stock firmware (FM-1 015; SMK-37 016) and Felucca answer it (m-bootloader SPEC §4.2). M-Flasher's DOWNLOAD MODE button sends it. **Disputed for the SMK-37:** [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) (02 §2, 2026-10-17) found that the SMK's stock app doesn't answer it, only the Felucca/SLOOP loader and app ([SMK-37](SMK-37.md)). | [HW] [?] |
| `F0 22 24 35 7F F7` | **Upgrade**: start an M-UPGRADE update session | [RE] [CODE] |
| `F0 00 32 45 00 00 00 40 7F F7` | handshake query used by Felucca's installer. The answer unpacks to a 34-byte identity block starting `00 59 11`. Stock firmware answers **only** parameter ID `0x40`: a scan of all other IDs got no reply (resource hub). | [CODE] [RE] |
| `F0 43 0n …` | Yamaha DX7 voice and bank SysEx (stock firmware) | [RE] |
| `F0 43 10 gg pp vv F7` | DX7 parameter change: stock V13 writes value `vv` at address `gg × 128 + pp` of the edit buffer (FM-1-RE `io/05-midi.md` §5.1) | [RE] |

The update session's framing is on [Update Protocol and Installers](Update-Protocol-and-Installers.md). Any firmware that wants to be updatable
by the community's installers should answer the soft key and the upgrade message.

**On a stock unit with the stock bootloader, the soft key is the only way into download mode without hardware.**
That's why M-Flasher can back up a stock unit before you change anything.

## USB implementation notes (Felucca family)

**[CODE]**
- The USB controller is a MUSB-style SIE. Felucca's driver polls it from the 10 kHz timer interrupt at 2 kHz; no
  USB interrupt is used. Everything that touches the controller's INDEX register runs in one place, so nothing
  can interleave.
- **Space out the controller's register accesses.** **[HW]** Built at `-O2 -mcpu=r3`, felucca-bsp's `sie_wr` and
  `sie_rd` were inlined, back-to-back SIE accesses ran a few cycles apart, and EP1 IN stopped: 0 of 10 identity
  requests answered, against 10 of 10 at `-Os` (dsablic, [felucca-bsp#14](https://github.com/hugelton/felucca-bsp/issues/14),
  2026-10-28; reproduced by Charles Vestal). Fixed by making them `noinline`
  ([14610bd](https://github.com/hugelton/felucca-bsp/commit/14610bd)). m-bootloader found the same: accesses through
  `0x11804` a few cycles apart made the controller babble on IN, and the host disabled the port, on about a third of
  transmits (SPEC §11.6).
- **Receive buffers need 4 bytes more than the packet size**: the controller's DMA writes past a full packet
  (m-bootloader SPEC §11.6). **[HW]**
- **A SETUP right after a STALL.** **[HW]** About 11% of control requests that followed a STALLed one at once timed
  out (196 of 1,800 on macOS), because the SETUP that arrived with the STALL's interrupt was not handled in that pass.
  Fixed in felucca-bsp [3c80360](https://github.com/hugelton/felucca-bsp/commit/3c80360) (imijes,
  [felucca-bsp#28](https://github.com/hugelton/felucca-bsp/issues/28), 2026-11-21).
- **Interface order for macOS.** **[HW]** macOS 13 to 15 create no USB audio device while MIDI is interface 1. With
  `FELUCCA_UAC_MIDI_LAST` the order is audio control 0, audio streaming 1, MIDI 2; Linux still gets its MIDI port.
  Tested on macOS 15.8.2 (dhilowitz, [hugelton/Felucca PR #322](https://github.com/hugelton/Felucca/pull/322)); in
  Felucca **1.8.3** (2026-11-22, [78ad72c](https://github.com/hugelton/Felucca/commit/78ad72c),
  [#67](https://github.com/hugelton/Felucca/issues/67)). fm1-base: `usb_midi_last`. Quit Chrome when testing:
  Chrome holding the FM-1 open keeps macOS's audio service away from it.
- MIDI in goes into a ring that the audio interrupt drains; MIDI out comes from a ring the audio interrupt fills.
  The audio code never waits on USB.
- If the USB poll falls inside the audio render, it waits for the next tick outside it, because both share the
  MIDI queues.
- Optional functions: a CDC-ACM serial port (a console during development) and a UAC1 audio input
  ([Audio Output](Audio-Output.md)). Newer ones in felucca-bsp, each off by default **[CODE]**:
  - a USB mass-storage **disk mode** (`FELUCCA_MSC`, [PR #25](https://github.com/hugelton/felucca-bsp/pull/25),
    2026-11-02): no MIDI or audio while it is on;
  - a **CDC-NCM network adapter** (`FELUCCA_NCM`, [PR #51](https://github.com/hugelton/felucca-bsp/pull/51),
    2026-11-26), for fm1-base's own WiFi stack (`usb_net`);
  - **real-time MIDI sent at once** (`FELUCCA_MIDI_RT`): a clock tick goes into EP1 from the 0.1 ms tick. Through the
    normal queue, it waited for the 2 kHz poll, and clock out scattered by 418 µs (standard deviation), up to 1.5 ms,
    on a Mac ([f4692d7](https://github.com/hugelton/felucca-bsp/commit/f4692d7), 2026-11-10) **[HW]**;
  - a **USB serial number** (`FELUCCA_USB_SERIAL_FN`; fm1-base `usb_serial`, from the chip's ID): without one, macOS's
    MIDI setup tells FM-1s apart by USB port only, so a renamed FM-1, or two swapped, show an old name
    ([290da0d](https://github.com/hugelton/felucca-bsp/commit/290da0d)).
- **MIDI timing in FM1 Open-style SDKs.** **[HW]** In M-VAVE's SDK, MIDI is sent from the control task
  (`fm1_midi_send`, "Control task context"). On the SMK37 Open SDK that task runs every 10 ms, so sequencer MIDI out
  had 10–14 ms of jitter on DIN and USB (SeattleUser0,
  [m-vave/smk37_open#4](https://github.com/m-vave/smk37_open/issues/4), 2026-11-10, one developer; no reply by
  2026-11-26). Not measured on FM1 Open.
- **USB host** (for plugging a USB MIDI keyboard into the FM-1) is ported in felucca-bsp from Melodee, opt-in, and
  needs **external power**: the FM-1's port does not supply VBUS. Input has run on hardware; the BSP port and the
  output path still need validation (felucca-bsp `docs/EXPERIMENTAL.md`). An Elektron Digitone II needed a 50 ms root
  reset (USB 2.0's TDRSTR) instead of 30 ms, and declares 512-byte bulk endpoints even at full speed, now used as 64
  bytes; with both, it was found and its MIDI clock arrived on an FM-1 (Henri David,
  [felucca-bsp PR #36](https://github.com/hugelton/felucca-bsp/pull/36), merged 2026-11-18). **[HW]**

## Sources

- Felucca's `src/usb.c`, `src/ota.c`, `hal/fm1_usb.h`, `hal/fm1_uart.h`, `tools/fm1_install.py` (GPL-3.0-only).
  Since Felucca 1.7.0 (2026-10-28) `usb.c`, `ota.c` and `fm1_usb.h` are felucca-bsp's `chip/usb.c`, `chip/ota.c`,
  `chip/fm1_usb.h` (MPL-2.0), in Felucca's `firmware/bsp` submodule; `fm1_uart.h` is `firmware/hal/fm1_uart.h`.
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) at `fdd709f`: commits and PRs linked above; issues
  [#14](https://github.com/hugelton/felucca-bsp/issues/14), [#28](https://github.com/hugelton/felucca-bsp/issues/28).
- [Felucca](https://github.com/hugelton/Felucca) PR [#322](https://github.com/hugelton/Felucca/pull/322), issue
  [#67](https://github.com/hugelton/Felucca/issues/67), [1.8.3 release notes](https://github.com/hugelton/Felucca/releases/tag/v1.8.3).
- [fm1-base](https://github.com/charlesvestal/fm1-base) commit [0eec347](https://github.com/charlesvestal/fm1-base/commit/0eec347), PRs #44, #45, issues #10, #34.
- [M-VAVE FM1 Open](https://github.com/m-vave/fm1_open) `sdk/include/fm1.h` at `4c7496d`; [smk37_open#4](https://github.com/m-vave/smk37_open/issues/4).
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §4.2, §11.6 (at `eb4f2c1`, 2026-11-23).
- [FM1 Open README](https://github.com/m-vave/fm1_open) (MIDI section).
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) (WTFPL; stock V13 unless said): [architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §1, §5;
  [`io/05-midi.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/05-midi.md) §1, §2.1, §5.1;
  [`io/06-usb.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/06-usb.md) §1, §4.4, §6;
  [`io/10-bluetooth.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/10-bluetooth.md) §1, §4, §6;
  [`io/12-v15-reflash-proof.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/12-v15-reflash-proof.md) (V15, 2026-09-04).
  Its code addresses need `0x120` added ([Memory Map](Memory-Map.md#the-stock-app-runs-at-0x02000120)).
- [felucca-bsp docs/EXPERIMENTAL.md](https://github.com/hugelton/felucca-bsp/blob/main/docs/EXPERIMENTAL.md).
- [FM-1 resource hub REVERSE_ENGINEERING.md](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) §1.2, §4.1, §4.2.
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) firmware skill §2, §10.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
