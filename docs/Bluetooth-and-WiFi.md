---
title: "Bluetooth and WiFi"
parent: Hardware
nav_order: 8.5
---

# Bluetooth and WiFi

The AC791N has one 2.4 GHz radio for **Bluetooth 5.0 (dual mode)** and **WiFi 802.11b/g/n**. Stock firmware uses
Bluetooth only. Community firmware can use either, through JieLi's closed libraries or through open stacks written
since October 2026. This page gathers what is known: the stacks, what each can do, the radio facts they rest on, and
what nobody has measured yet. MIDI over Bluetooth is also on [MIDI and USB](MIDI-and-USB.md); the build options are on
[fm1-base as a Base](FM1-Base.md).

Most of the open work is by **Henri David** (hdavid / w0ts, Optimist) and **Charles Vestal** (fm1-base), with
**DimmKirr** (modd) for a second WiFi driver. Checked against the sources on 2026-11-27.

## The stacks at a glance

| Stack | Bluetooth | WiFi | Licence | Used by |
| --- | --- | --- | --- | --- |
| **JieLi's libraries** through fm1-base's shim | BLE MIDI, peripheral and central **at the same time** | station, WPA2 and open networks, lwIP with **TCP, DNS, HTTP(S)** | binaries under the AC79 SDK's Apache-2.0 (see [Licences](#licences)) | fm1-base `ble = true` / `wifi = true`, CTL-1, fm1-radio, FIELD |
| **M-VAVE's FM1 SDK** (JieLi's stack inside `libfm1_engine.a`) | BLE MIDI (one link, peripheral or central) and **A2DP sink**: a phone plays through the FM-1 | none: its build refuses WiFi | M-VAVE's binary terms, FM-1 devices only | FM1 Open, SMK37 Open, foma's `mvave` platform |
| **The own stacks** (Henri David; in Optimist, fm1-base, foma and many others) | BLE MIDI (one link at a time), pairing, central role, **A2DP source**: the FM-1 plays to headphones | WPA2-PSK station, own IP layer: ARP, DHCP, IGMP, **UDP only** (no TCP, no DNS); access point, JANK, USB network, Ableton Link | `GPL-3.0-only OR MPL-2.0`; Link GPL-3.0-only | Optimist, fm1-base `ble_stack = "own"` / `wifi_stack = "own"`, foma, Melodee, LOOM-8 and others |
| **modd's driver** (DimmKirr) | uses Optimist 0.4's BLE stack, off in every shipped configuration | WPA2-PSK station, **lwIP with TCP, DNS and HTTP**; no access point, no Link | GPL-3.0-only | modd |

**[CODE]** for each row: fm1-base README and `LICENSING.md` at [`e4863fa`](https://github.com/charlesvestal/fm1-base/tree/e4863fa);
FM1 Open `sdk/include/fm1.h` and `sdk/Build.ps1` at `4c7496d`; modd `kconf/` and `firmware/src/` at
[`a833219`](https://github.com/DimmKirr/modd/tree/a833219).

A fifth, Julius Vitkauskas's "open BLE" (a clean-room controller with Apache NimBLE as host, peripheral only), was
offered to fm1-base as [PR #13](https://github.com/charlesvestal/fm1-base/pull/13) on 2026-10-26 and closed unmerged
the same day. Its notes claim tests with macOS, iOS, Android and BlueZ and an 8-hour soak; not checked here. **[?]**

**On JieLi's libraries, Bluetooth or WiFi, not both**: together they don't fit in RAM. **On the own stacks, both**,
from the main loop. modd's WiFi can't run with Bluetooth either ("mutually exclusive with BLE in this version",
`kconf/wifi.kconfig`). **[CODE]**

## Bluetooth

### What each stack does

- **JieLi's, through fm1-base.** Peripheral (BLE MIDI to a computer) and, with `ble_central`, central (to a keyboard)
  at once, one link each. It asks for 11.25–26.25 ms connection intervals, advertises every 100 ms as `FM-1 XXXX`, and
  keeps the radio calibration it measured at first start in the app's own store, not in the stock VM. Cost: +124 KB
  flash, +45 KB RAM, +24 KB pool (fm1-base `ble/midi/le_midi.c`, `ble/shim/misc.c`, README). **[CODE]**
- **M-VAVE's SDK.** `fm1_bluetooth_configure(FM1_BT_AUDIO | FM1_BT_MIDI)`: BLE MIDI and **Bluetooth audio in**
  (A2DP sink, 16-bit PCM blocks the app mixes), usable together. One BLE MIDI connection, as peripheral or central
  (switching roles disconnects), a 15 s scan listing up to 8 devices (FM1 Open `sdk/include/fm1.h` l.46–90). SMK37
  Open advertises two names, one for BLE MIDI and one for audio pairing (its README). **[CODE]**
- **The own stack.** Written from the Bluetooth Core spec (LE 1M only) and the MIDI Association's BLE-MIDI 1.0, in
  portable C without malloc. It runs in the radio's two interrupts (IRQ 45 and 29, priority 2, below audio) and hands
  data to the main loop through rings (fm1-base `docs/bt/BLE-STACK.md` §1, §3, §7). **[CODE]**
  - **LE:** advertising, peripheral and central, ATT (MTU 247), the GAP/GATT/BLE-MIDI services, LE legacy pairing with
    a passkey shown on the FM-1 (Just Works when the peer can't type), encryption in software, one remembered device.
    Not built: LE Secure Connections, two links at once, LE 2M or Coded PHY.
  - **Classic (BR/EDR):** Secure Simple Pairing, L2CAP, SDP, AVDTP, **A2DP source** with AOSP's SBC encoder, and a
    minimal AVRCP target (enough for AirPods). No HFP, no SPP, no HID (`docs/bt/BT-CLASSIC.md`).
  - **One BLE link at a time:** while the FM-1 is connected to a device, a computer can't connect, and the FM-1 doesn't
    advertise (fm1-base README, "Two Bluetooth stacks").
  - **Bonds and keys** live in the app's own store (`OBJ_BLE`), never in the stock VM; up to four A2DP devices are
    remembered. Each firmware keeps its own keys, so **after changing firmware, forget the FM-1 on the phone or
    speaker**. The FM-1's identity key is fixed, because macOS refuses a new one from a device it already paired with
    (README; `device/bt_own.c`). **[CODE]**
  - Cost on fm1-base's template: BLE MIDI +41 to 46 KB flash, +10 KB RAM; with pairing, devices and A2DP +57 to 62 KB,
    +8 KB, +30 KB of pool more ([fm1-base as a Base](FM1-Base.md)). **[CODE]**
- **modd** carries Optimist 0.4's BLE stack unchanged, plus two GPL-3.0-only additions by DimmKirr: an **HID client**
  that reads an Xbox Series controller and turns it into MIDI CC (the only HID code seen on the FM-1), and a **file
  transfer over a GATT service**. At `a833219` the file transfer is a skeleton: a fixed list of three entries, the flash
  writes stubbed, the CRC never checked. BLE is off in all four shipped configurations. No hardware test of either is
  documented. **[CODE]** [?]

### BLE MIDI in practice

- **Devices that worked with the own stack [HW]:** macOS (Audio MIDI Setup); an iPhone 14 (GarageBand, Cubasis,
  ZenBeats, Moog Model D); a CME WIDI Master with an SP-404 behind it; a Korg microKEY; a Roland FP-30X as the FM-1's
  central target after the pairing fix ([#44](https://github.com/charlesvestal/fm1-base/pull/44),
  [#45](https://github.com/charlesvestal/fm1-base/pull/45)); another FM-1. On units of Henri David, Jeremy
  Murray-Wakefield and Charles Vestal, and two FM-1s by ejtb76 (2026-10 and 11).
  **Never reported: Windows and Android.**
- **Link quality, measured.** With the receive-window fixes, CRC failures stayed between 0.17 % and 0.68 % and every
  notification was acknowledged: an iPhone at 7.5 ms with a 240 BPM sequencer (8 min), the WIDI with an SP-404 (8 min),
  and both with Bluetooth audio streaming at the same time (Jeremy Murray-Wakefield's FM-1,
  [optimist#17](https://github.com/w0ts/optimist/issues/17), 2026-11-06). **[HW]**
- **The WIDI fix.** With a WIDI Master (7.5 ms interval, 50 ppm clock), half the packets failed CRC and a dead link
  stayed "connected". The receive window needed a 16 µs floor and a wider steady window, and the supervision timer had
  to fire. Commit [`0eec347`](https://github.com/charlesvestal/fm1-base/commit/0eec347) is Henri David's, with Jeremy
  Murray-Wakefield as co-author; the issue of record is [optimist#17](https://github.com/w0ts/optimist/issues/17)
  (2026-10-24 to 11-06). **[HW]**
- **Things learnt the hard way [HW]:** an iPhone as peripheral refused notifications after Just Works pairing (it wants
  an authenticated key, hence the passkey); sending "Service Changed" made a Mac never find the MIDI characteristic
  (2026-10-09, `BLE-STACK.md` §5, §9.2); while A2DP streams, intervals under 30 ms drop about one audio packet in ten,
  so the FM-1 then asks for 30–45 ms (2026-10-24 to 26).
- **Timestamps.** The own stack sends real 13-bit timestamps (stock sends `80 80`); with `ble_midi_time`, incoming
  messages are placed by the sender's time. One odd timestamp once shifted that mapping by over a second; fixed
  ([fm1-base#54](https://github.com/charlesvestal/fm1-base/issues/54), 2026-11-15). **[HW]**
- **Still open on JieLi's stack:** with a computer connected first, a device's service search times out; device first,
  then computer, works ([fm1-base#10](https://github.com/charlesvestal/fm1-base/issues/10), 2026-10-23, no update
  since). On the own stack the case can't arise, since it has only one link. **[HW]**
- **Latency and jitter: not measured** by anyone, on any stack.
- **Stock BLE MIDI.** V13 uses JieLi's full dual-mode stack, with the BLE-MIDI service beside two JieLi vendor services
  (FM-1-RE [`io/10-bluetooth.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/10-bluetooth.md)). **[RE]**
  Disagreements: FM-1-RE gives the default interval as 33.75–410 ms and the name family as `FM-1 Midi` (V13);
  Optimist's emulator trace of V15 shows requests for 7.5–11.25 ms then 15 ms, a BLE name `FM-1_BLE`, and a Security
  Request at every connection. Different versions, or defaults against requests; both kept. **[RE]**

### Bluetooth audio (A2DP)

- **Two directions, two stacks.** The **sink** (a phone plays through the FM-1) exists only in M-VAVE's SDK. The
  **source** (the FM-1 plays to headphones or a speaker) exists only on the own stack (fm1-base `bt_audio = true`, from
  Optimist, 2026-10-26). While it streams, the FM-1's own output is muted. **[CODE]**
- **Codec:** SBC, bitpool up to 53, 44.1 kHz preferred (48 kHz if the sink wants it), basic rate. AAC and aptX were
  tried and rejected for their CPU cost. **[CODE]**
- **Latency:** about 150–300 ms end to end, mostly in the headphones: for listening, not for playing (inferred,
  `BT-CLASSIC.md` §15). **[?]**
- **No resampler.** The encoder is set to 44,100 Hz and fed the FM-1's real 44,117.6 Hz; a full queue (40 packets,
  ~350 ms) drops a whole packet (`bt/br/bt_acl.c`). What that does to the sound over minutes is not measured. **[CODE]** [?]
- **CPU cost, two readings:** 3.1 % of a core in the emulator (a lower bound, no cache or flash waits), and "~31 % of a
  block, up to 54 %" measured on an FM-1, which is why the SBC frames now run as render jobs on either core
  (fm1-base [`bca6c41`](https://github.com/charlesvestal/fm1-base/commit/bca6c41), 2026-11-06). **[HW]**
- **Sinks tried [HW]:** AIAIAI TMA-2, Anker SoundCore 2, Sony WH-1000XM4, AirPods Pro. A speaker may not answer for a
  minute or two after an abrupt drop. The stream crackled as the radio warmed until AFH (dropping bad channels) was
  added, and a headset was lost when its AFH message went out late; it now goes from the interrupt (2026-11-06).

## WiFi

### What each stack does

- **JieLi's, with lwIP** (fm1-base `wifi = true`, a network task `plat_net_task`): Ralink's RT30xx station driver as
  compiled by JieLi, its WPA supplicant and lwIP. About 300 KB of flash, 100 KB of RAM and 276 KB of pool, leaving an
  app about 170 KB of flash; no Bluetooth in the same build. **[CODE]** [fm1-radio](https://github.com/charlesvestal/fm1-radio)
  shows what it can do: WPA2 and open networks, hidden networks, six networks remembered, TCP, DNS, HTTP and HTTPS
  (BearSSL, TLS 1.2, **no certificate check**), internet radio (fm1-radio README at `494c2a6`). **[CODE]**
- **The own stack** (fm1-base `wifi_stack = "own"`, merged 2026-11-25; Henri David, from Optimist):
  - **Joins only WPA2-PSK networks with CCMP.** Open, WPA3-only, TKIP and networks that require protected management
    frames are listed but greyed. Channels 1 to 13.
  - **Sends at 1 and 2 Mb/s only**: made for small datagrams, not bulk data.
  - **IP:** ARP, DHCP client, IGMPv2 (one group), UDP (up to four ports). **No TCP, no DNS**, no task: all from the
    main loop. The WPA2 key derivation is spread over main-loop passes; the password is never stored, only the key.
  - **Modes:** station; **MODE AP** (WPA2-PSK, up to 4 clients, 192.168.4.1, DHCP server, no internet behind it);
    **MODE JANK** (FM-1s of a group exchange open, unencrypted frames with no router; anyone in range can read and
    inject, so it is opt-in); **USB network** (below).
  - Needs the own Bluetooth stack's radio start-up; since 2026-11-06 it also starts when Bluetooth is switched off.
  - Cost on top of the own BLE stack: +18.6 KB flash, +3.8 KB RAM; Link +8.8 KB, +1.8 KB; MODE AP +6.4 KB, +1.7 KB of
    pool (fm1-base README, October 2026; Optimist's builder measured 26.6 KB / 1.9 KB / 3.7 KB pool for WiFi and Link
    together, another build). **[CODE]**
  - (fm1-base `docs/wifi/README.md`, `device/wifi_own.c`; felucca-bsp `chip/fm1_wifi.h` at `fdd709f`.)
- **modd's driver** (DimmKirr, first public commit 2026-11-07): about 9,000 lines in `firmware/src/wifi/`, with lwIP
  2.2.1 in `firmware/src/net/`: **TCP, DNS, DHCP, UDP**; no IGMP (so no Link as configured), no IPv6, no sockets API.
  Station only, WPA2-PSK, no access point. On it: an HTTP client (a weather page through ip-api.com and Open-Meteo, in
  plain HTTP), an LLM client to a local proxy, and an on-device REST/WebSocket server on port 9222 (`/api/...`, page
  state, input, audio capture) with **no authentication** found. `CONFIG_WIFI_MIDI` (rtpMIDI/OSC) has no code behind
  it yet. **[CODE]** at `a833219`.
  - **Its "clean room".** modd's `docs/hardware/wifi/CLEAN_ROOM_METHODOLOGY.md` describes a reader who writes a
    register specification (`SPECSHEET.md`, about 3,500 lines, every fact tagged) and a separate implementer who writes
    the driver from it. By its own table this covers the RF start-up, calibration, RF-die table, frame DMA and BBP; the
    MAC, station, WPA2 and IP layers are written from public standards and GPL/BSD references. The document itself says
    that mechanical results are no proof of independence. **[CODE]**
  - **On hardware:** the last dated record in the repo (2026-11-04 to 06, DimmKirr's FM-1) shows transmit, receive and
    a 13-channel scan with 16 networks, then authentication with no unicast frame acknowledged, then a transmit-order
    bug found. Later commit messages say it joins a network and fetches over HTTP, but no dated record of a join, a
    lease or the weather fetch is in the repo; DimmKirr's Reddit post (late November 2026) shows the weather demo.
    **[?]** until the build, unit and date are given.
  - Two points in the Reddit post don't match the repo at `a833219`: fm1-radio is not listed among its upstreams, and
    since 2026-11-21 hardware builds use JieLi's toolchain by default (the open pi32v2 toolchain is the emulator's).
- **Stock firmware and M-VAVE:** V15 never turns WiFi on (emulator trace, Optimist `docs/WIFI-HW-FACTS.md` §2b), and
  FM1 Open's build script refuses any WiFi code (`sdk/Build.ps1`). **[RE] [CODE]**

### What WiFi is used for

- **Ableton Link** (tempo, beat, start/stop; no Link Audio): Henri David's C port, Ableton's wire format, GPL-3.0-only.
  A firmware with `link = true` is GPL-3.0-only, and fm1-base's build refuses it otherwise. Since 2026-11-10 the session
  clock is filtered (the fastest half of the round trips, drift learned), which departs from Ableton's method. **[CODE]**
  Used by Optimist (since 0.3) and fm1-base's `examples/link`.
- **MODE JANK**: FM-1s jamming in one Link session without a router. A companion stick,
  [jank](https://github.com/w0ts/jank) (ESP32-S3), joins a group and brings in a computer, MIDI clock and analog clock
  outputs. **[CODE]**
- **A USB network** (fm1-base `usb_net`, felucca-bsp `FELUCCA_NCM`, merged 2026-11-26): the FM-1 appears to the
  computer as a **CDC-NCM network adapter**, with link-local addresses (no DHCP). With WiFi off, the IP layer and Link
  run over the cable; in MODE JANK the computer joins the group's air. **[HW]** On two FM-1s with macOS 26: ping 3.1 ms
  on average (20/20, Charles Vestal) and about 5.7 ms (5/5, Henri David), and **Live 12 joined the FM-1's Link session
  over the cable** ([fm1-base#95](https://github.com/charlesvestal/fm1-base/pull/95), 2026-11-25/26). Windows and Linux
  untried.
- **modd's USB network is the opposite:** the FM-1 is the USB *host* and drives a USB Ethernet adapter (CDC-ECM, written
  for an ASIX AX88179A) or an iPhone's tethering ("still being tested"), behind a hub; lwIP prefers wired, then WiFi,
  then tethering (`firmware/src/net/usbnet_ecm.c`). A commit implies a test with an adapter; no unit or date. **[CODE]** [?]
- **Internet audio:** fm1-radio (Radio Garden streams, MP3/AAC over HTTP/HTTPS) and FIELD
  ([AveyCole/FM1-Field](https://github.com/AveyCole/FM1-Field), field recordings from Radio Aporee), both on JieLi's
  stack. **[CODE]**
- **Getting the password in:** Optimist's installer asks for it after installing a WiFi build and sends it once;
  fm1-base and fm1-radio use `fm1_remote.py wifi join SSID PASSWORD` over USB, or the keys. **[CODE]**
- **Not found anywhere:** OSC, RTP-MIDI, firmware updates over WiFi, network time, a web server on fm1-base's stacks.

### Measured on WiFi

- **Own stack on Optimist [HW]** (the owner's FM-1): receive, scan, WPA2 join and group traffic on 2026-10-26; a DHCP
  lease, the Link multicast group and **Live in sync** on 2026-10-27; two FM-1s in MODE JANK 2 m apart, -44 dBm, no
  frame lost, on 2026-11-04.
- **Own stack on fm1-base [HW]:** `examples/link` found three networks on Charles Vestal's FM-1 (2026-11-10); Optimist
  built on fm1-base main joined WPA2 and ran JANK with the jank stick (Henri David, 2026-11-19)
  ([fm1-base#31](https://github.com/charlesvestal/fm1-base/pull/31)); the USB network above (2026-11-26).
- **MODE AP, two accounts:** fm1-base's `docs/wifi/README.md` says an iPhone with Ableton Note joined the FM-1's access
  point and saw one Link peer on 2026-10-28; Optimist's builder text still says "not tested on the device yet" and its
  limits note (same day, earlier?) says no client had joined. Probably the notes predate the test; nobody has said so.
- **Link timing:** the C port matched Ableton's library to 0.003 ms of phase on one Mac (host, loopback). On the air,
  only commit messages: an FM-1 on Link's own method "ramps and steps by a few hundred µs"; after the filter, a wander
  of "sd 19.6 µs on the audio bench" (fm1-base [`98b33ff`](https://github.com/charlesvestal/fm1-base/commit/98b33ff),
  Optimist `a1bb6756`, 2026-11-10). Method, unit and length not written down. **[HW]** [?]
- **Throughput on JieLi's stack:** fm1-radio paces downloads at 48 KB/s, because "flat out, records came in corrupted
  and WiFi dropped" (`671b5e7`, 2026-10-16). **[HW]**
- **Transmit strength, two readings:** a weak -86 dBm turned out to be a buffer problem (-48 dBm after the fix,
  2026-10-27); a later note has the access point hearing the FM-1 about 20 dB low at 3–4 m and joins working about 6 in
  8 times, "never explained" (Optimist `docs/WIFI-CLAUDE-LIMITS.md` §3, 2026-10-28). **[HW]**
- **Not measured on any stack:** range, join time, battery cost, audio dropouts while the radio works, a long soak,
  WiFi beside Bluetooth audio.

## The radio underneath

Both radios share one front end, one PLL and one start-up; the own stacks start the radio once per boot and keep
stock's air-time priorities. The register sources are Optimist's
[`docs/WIFI-HW-FACTS.md`](https://github.com/w0ts/optimist/blob/2b559c1d/docs/WIFI-HW-FACTS.md), fm1-base's
`docs/bt/BLE-HW-FACTS.md` (from Optimist), felucca-bsp's `chip/fm1_wifi.h`, and modd's
[`SPECSHEET.md`](https://github.com/DimmKirr/modd/blob/a833219/docs/hardware/wifi/SPECSHEET.md), now the largest WiFi
register reference. Addresses are on [Memory Map](Memory-Map.md).

- **The WiFi MAC** is JieLi's wl30 core with Ralink RT3070's register layout at `0x30000` + r. JieLi's driver is
  Ralink's RT30xx USB station driver, file names unchanged, behind a shim that is really memory-mapped. MAC_CSR0 reads
  `0x00300100` on an FM-1, not RT3070's ID (2026-10-26). No microcode is loaded, unlike a real RT3070. **[SDK] [HW]**
- **Frame DMA** at `0x30F00`–`0x30F1C`, one frame at a time. **It reads garbage above `0x01C20000`**: frames went out
  corrupted with no error until the buffers moved to main RAM (2026-10-27). The Bluetooth engine has the same limit;
  fm1-base's build checks it. **[HW]**
- **Receive interrupt 52 never latched** in Optimist's probes; both open stacks poll. Open. **[HW]**
- **Receive needed the coexistence controller:** 565 words to `0x14004` that stock V15 never writes, plus `0x14000 |=
  3`. Without them, no frames at all (2026-10-26). **[HW]**
- **The power set-up matters for WiFi:** without M-VAVE's supply settings, scans came back empty or nearly
  (felucca-bsp `fm1_power_stock`, 2026-10-23); at the bootloader's voltages, flash reads failed under WiFi load in
  fm1-radio ([Clocks and Power](Clocks-and-Power.md)). **[HW]**
- **The Bluetooth engine** is hardware-timed and driven by a control block in SRAM. Its transmit-buffer flag had been
  read with the wrong polarity, "which explained every TX stall seen" (2026-10-08 to 10). The Bluetooth slot clock steps
  backwards during a connection, so the own stack times with TIMER4. **[HW]**
- **TX power** is a level 0–11; stock uses 6. dBm per level, sensitivity and range are unmeasured. Nothing is known
  about the antenna. **[RE]** [?]
- **Radio start versus audio:** JieLi's start-up draws about 500 ms of high current, and flash reads during it can
  return corrupted data. fm1-base holds the render for the first second; foma pauses the sound; CTL-1 crashed when its
  synth read samples from flash then (fm1-base `device/radio.c`). **[HW]**

### Calibration, and where it lives

- **Stored on a stock FM-1 [HW]** (one unit, 2026-10-08): the SDK's VM at `0xE8000` holds records 106 (crystal load),
  107 (PA bias), 108 (per-rate gains) and **187 (the measured calibration)**, plus 109 and 113, which are **BLE keys,
  not calibration**. 106, 107 and 108 hold the SDK demo board's constants, so M-VAVE never tuned them; only 187 is
  measured per unit. BTIF at `0xE9000` holds the Bluetooth address ([Flash Storage](Flash-Storage.md)). **[RE]**
- **Many firmwares overwrite that VM** (sample banks and user areas reach `0xE5000`–`0xFBFFF`), so a firmware can't
  count on it ([fm1-base#14](https://github.com/charlesvestal/fm1-base/issues/14), open).
- **Three approaches, all on hardware:**
  - **fm1-base calibrates at every start** with felucca-bsp's `chip/fm1_rf_cal.h`, JieLi's calibration rewritten in C
    from the library (Apache-2.0, JieLi's copyright); nothing stored is needed (one unit, Charles Vestal, 2026-10-26).
    On JieLi's WiFi stack too, because a kept calibration was Bluetooth's and the access point could not hear the FM-1.
  - **foma** found that same calibration hung or faulted in most of its builds (2026-10-31). It uses the unit's stored
    records when present, else constants, plus its own scan of the VCO bands; with another unit's bands, reception
    went bad within seconds (foma `platform/bt.c`).
  - **Optimist** refuses to start the radio without stored trims (`BLE-STACK.md` §8.2).
- The VCO band scan runs at every boot (about 45 ms) and is stored nowhere. The rest of the own stacks' start-up is a
  replay of what stock V15 writes, captured in an emulator (`chip/ble_rf_tables_v15.h`, Apache-2.0). **[RE]**

## Licences

- **JieLi's binaries:** fm1-base, fm1-radio and Optimist treat the AC79 SDK, its precompiled libraries included, as
  Apache-2.0, so an MPL-2.0 firmware may link them; a GPL firmware needs a GPL-3.0 section 7 permission from every
  copyright holder of its GPL code (fm1-base `LICENSING.md`, [fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99)).
  modd's methodology instead treats the precompiled libraries as proprietary and reads them only under the
  interoperability exceptions. Both views kept; not legal advice.
- **The own stacks** link no JieLi binary, but they do ship JieLi-derived code under Apache-2.0: the radio start-up
  tables, the 565 coexistence words, and felucca-bsp's C rewrite of the calibration. fm1-base's copy of the stack's
  notes calls it "clean room"; Optimist's current text says "an independent implementation", with the register facts
  read from the vendor binary by a separate reader. modd's methodology excluded `fm1_rf_cal.h` as a derivative of the
  vendor library.
- **Ableton Link** makes a firmware GPL-3.0-only. Whether a release may say "Ableton Link" also depends on Ableton's
  Link guidelines; Optimist's design notes say they must be read first, and no one has said they were.

## Open questions

See [Open Questions](Open-Questions.md#the-radios) 44 to 52: BLE MIDI latency, A2DP drift, the SBC cost, radio work and
audio dropouts, battery cost, Link phase against Live, range, modd's join, Windows and Android hosts.

## Sources

- [fm1-base](https://github.com/charlesvestal/fm1-base) at `e4863fa` (2026-11-27): README, `LICENSING.md`,
  `docs/bt/BLE-STACK.md`, `docs/bt/BT-CLASSIC.md`, `docs/bt/BLE-HW-FACTS.md`, `docs/bt/RF-CALIBRATION.md`,
  `docs/wifi/README.md`, `ble/`, `bt/`, `device/`; issues and PRs #10, #13, #14, #31, #44, #45, #54, #95, #99; commits
  `0eec347`, `bca6c41`, `98b33ff`.
- [Optimist](https://github.com/w0ts/optimist) at `2b559c1d` (2026-11-27): `docs/WIFI-HW-FACTS.md`,
  `docs/WIFI-TX-HANDOFF.md`, `docs/WIFI-LINK-HANDOVER.md`, `docs/WIFI-CLAUDE-LIMITS.md`, `docs/LINK-DESIGN.md`,
  `docs/BLE-STACK.md`, `docs/BUILDER.md`; [optimist#17](https://github.com/w0ts/optimist/issues/17).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) at `fdd709f`: `chip/fm1_wifi.h`, `chip/fm1_rf_cal.h`,
  `chip/ble_rf_tables_v15.h`, `chip/fm1_sys.h`; PRs #24, #51.
- [modd](https://github.com/DimmKirr/modd) at `a833219` (DimmKirr, GPL-3.0-only): `kconf/`, `firmware/src/wifi/`,
  `firmware/src/net/`, `firmware/src/ble/`, `docs/hardware/wifi/`; r/MVaveFM1, DimmKirr, "I've implemented a clean
  room WiFi and USB OTG Ethernet with lwIP" (November 2026).
- [fm1-radio](https://github.com/charlesvestal/fm1-radio) at `494c2a6`; [FM1-Field](https://github.com/AveyCole/FM1-Field)
  at `475fa10`; [jank](https://github.com/w0ts/jank) at `c36a5f2`; [foma](https://github.com/konsumer/foma) at `885100f`
  `platform/bt.c`.
- [M-VAVE FM1 Open](https://github.com/m-vave/fm1_open) at `4c7496d`: `sdk/include/fm1.h`, `sdk/Build.ps1`;
  [smk37_open](https://github.com/m-vave/smk37_open) README.
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) `docs/io/10-bluetooth.md` (V13, WTFPL).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
