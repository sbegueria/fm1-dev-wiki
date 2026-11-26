---
title: "SMK-37"
parent: Hardware
nav_order: 9
---

# SMK-37

The M-VAVE SMK-37 Pro and Elite are 37-key controllers with 16 pads, 8 encoders, 4 faders, pitch and mod wheels,
a colour screen and a built-in synth. Inside: **the same JieLi AC791N (WL82) chip as the FM-1, the same
second-stage bootloader, the same update loader and the same chip key. Only the panel wiring is different**
(genkigenki's [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37)). **[HW]** The MKE-P37 and the Donner
Starrykey 37 Play share this hardware and run the same firmware, one package per model name (fm1-to-smk37 01). **[RE]**

"The same bootloader" means the FM-1's **V15** one. The SMK-37's stock 016 packages carry JieLi's AC79 SDK 1.1.9
`uboot.boot`, the same file as FM-1 V15's, and their boot configuration is byte-identical to FM-1 V15's after the
chip-key record (360 MHz). FM-1 V16 brought a newer bootloader with a 320 MHz set-up; the SMK-37 has nothing like it
(m-bootloader [docs/STOCK-V15-V16.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md),
121cfbe, 2026-11-21). **[RE]**

In the r/MVaveFM1 thread, several people asked that the SMK-37 be treated as a first-class target from the start,
so the community doesn't split in two.

**About fm1-to-smk37's facts.** Its hardware map marks each fact *verified* (checked on one SMK-37 Pro with a custom
firmware: its debug console, a flash read in download mode, or plain use) or *analysis* (read from stock firmware 016,
not yet confirmed on a unit). This page tags them **[HW]** and **[RE]**. Its text is under
[CC BY-NC 4.0](https://creativecommons.org/licenses/by-nc/4.0/) (its `LICENSE`), so this page gives its facts in
our own words with a link, not its tables; read the originals for the full detail. Checked at commit `54252ff`
(2026-10-17).

## What differs from the FM-1

| | FM-1 | SMK-37 Pro / Elite | Tag |
| --- | --- | --- | --- |
| Key matrix | 2 × 74HC595, 16-bit word, MSB first | 4 × 74HC595, a 32-bit word shifted by DMA on SPI2 pin group C (CLK PA3, MOSI PA4), sent **least significant byte first**, MSB first within each byte; latch PA1; output enable PA0, active low | [HW] |
| Matrix rows | PA0, PA5–PA8, PB7 | eight return lines, rows 0–7: PA2, PA5, PA6, PA7, PA9, PH6, PH8, PH9. Keybed on rows 0–4, the 18 buttons on rows 5 and 6 (PH6, PH8), the encoders on row 7 (PH9). The four keys m-bootloader reads (`<`, `>`, SEQ PLAY, SEQ REC) are all on PH8 | [HW] |
| LEDs | PA9, PA10, PH6, PH9 lines | **no key LEDs.** 19 button LED lines in two groups (see below), whose commons sit on the high-speed USB PHY's D+/D− pads (`0x16A0C`); 16 RGB pad LEDs | [HW] |
| Analog inputs | MASTER pot (PB6) and battery (PB1) on the ADC | three 8:1 muxes on ADC ch1 (PA8), ch8 (PC9) and ch2 (PA10): 16 pads, 4 faders, both wheels, pedal, battery | [HW] |
| Screen SPI1 | group B: CLK PC9, MOSI PC10, CS PC7, D/C PC8; backlight PA2 | group A: CLK PB6, MOSI PB7, CS PC8, D/C PB5; no controllable backlight (it is on whenever the unit is on); `BAUD` 4, about 12 MHz | [HW] |
| Panel init | m-bootloader sends a short ST7789 sequence; M-VAVE's firmware and FM1 Open send the full one too (FM1 Open `app/board/lcd.inc`), and newer FM-1 panels need it ([Display](Display.md)) | full ST7789 sequence with porch, gate, VCOM, voltages and gamma; not the FM-1's values (VCOM `32` against M-VAVE's FM-1 `18`, other gamma) | [HW] [CODE] |
| Flash layout (stock) | firmware `[0x4000, 0x93000)` | firmware `[0x4000, 0x9D000)`, VM `[0x9D000, 0xC0000)`, USRTRIM and USRFLASH (below) | [HW] |
| Calibration | none known | **factory calibration** at `0xC1000` (`trimdat` marker) and wheel values in the presets header: **unit-specific, never copy between units** | [HW] |
| Flash chip (JEDEC ID) | `0x856014` (Puya), or `0xB36014` (UC25HQ80) on newer units | `0xEB6014` on two units reported so far ([fm1_open#11](https://github.com/m-vave/fm1_open/issues/11), dsgnloopz 2026-10-21; the SMK-37 Pro in m-bootloader's [v3.0 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0)); the same ID on fm1-to-smk37's unit. Firmware that checks the ID must accept it | [HW] |
| MIDI jack | TRS MIDI in | **no MIDI input.** The 3.5 mm jack is **MIDI OUT**: UART1 transmit on PC7, 31,250 baud (`BAUD` 383 from the 48 MHz UART clock), as stock 016 sets it up. Snoop37 2.6.4 has a driver built that way, not yet confirmed on a unit; a community spec sheet calls the jack type A, not measured (fm1-to-smk37 01 §9) | [RE] |
| Bootloader keys (m-bootloader) | REC / PLAY | SEQ REC / SEQ PLAY | [HW] |
| Soft key | first USB-MIDI port "FM-1 MIDI 1" | first of three ports, "… Private"; **disputed**, see below | [HW] [?] |
| Audio | | the same path; fm1-to-smk37's port needed no change to the audio driver, and the synth engines run unchanged | [HW] |

(m-bootloader SPEC §5.5, §8.2–§8.4; M-VAVE's [smk37_open](https://github.com/m-vave/smk37_open) README;
fm1-to-smk37 01 §1–§9.) An earlier version of this page said the SMK has a DIN MIDI out, with no source; the jack is
the 3.5 mm one above.

**Two sources disagree on the soft key.** m-bootloader's SPEC (§4.2, at eb4f2c1, 2026-11-23) says M-VAVE's stock 016
on the SMK-37 Pro and Elite enters download mode on `F0 22 24 35 7D F7` received on its first USB-MIDI port, the one
named "… Private", and that this was seen on an SMK-37 Pro with M-VAVE's bootloader. **[HW]** fm1-to-smk37
(02 §2, 2026-10-17) says only the Felucca/SLOOP update loader and app understand it, "not the stock app", and its
backup method starts a custom update loader first, then sends the soft key to that ([Recovery](Recovery.md)). **[HW]**
It doesn't say which of the three ports it tried. If a stock SMK doesn't answer the soft key, the kit's route still
works. **[?]**

## The panel in detail

A summary of fm1-to-smk37's hardware map (01 §4–§7), in our words. The key, button and encoder codes, the mux table
and the LED tables are there.

**Keys, buttons and encoders.** **[HW]**
- The matrix has 16 columns and 8 rows. In the 32-bit word, bits 31–16 select one column (active low, one at a
  time), and the low 16 bits drive the button LEDs, except bits 5–3, which carry the analog mux address.
- The order inside a scan step matters: read the rows (they belong to the column latched one step earlier), pulse
  the latch, then start the DMA of the next column's word. With another order, the kit's codes shift by one column.
- Each of the 37 keys has two contacts. Velocity, as stock firmware does it: the time from the first to the second
  contact in 0.1 ms units goes through **that key's five-point curve from the factory calibration**, giving 0–1023;
  the MIDI velocity is that value shifted right by 3. The note ends when the first contact opens.
- Encoders K1–K8 are quadrature pairs on row 7. One detent is a whole quadrature cycle, 4 counts: divide by 4 for
  one step per click.
- The kit's port scans one column per tick of a 10 kHz timer: a frame takes 1.6 ms, 625 frames a second, against
  the FM-1's 909. Code that counts time in scan frames runs at another speed on the SMK; count milliseconds
  (fm1-to-smk37 08). **[CODE]**

**Button LEDs.** The 19 LED lines are the low bits of the 595 word, active low, multiplexed in two groups: one scan
frame shows group 0, the next group 1. The group commons are bits 17 and 18 of `0x16A0C` (low = on), with that
register's GPIO-mode bits kept set. **[HW]** for the LEDs the kit's port drives, **[RE]** for the rest of its table.
One line is the second colour of SEQ PLAY's LED (fm1-to-smk37 08, "Hardware").

**Pads, faders and wheels.** **[HW]**
- The top row of pads is hardware pads 0–7 (mux A), the bottom row 8–15 (mux B), both left to right. The same index
  serves the pad's reading and its LED. Checked with a test on a unit (2026-10-10); older notes that say the
  opposite are wrong (01 §6.3).
- Each pad's velocity goes through its own five-point curve from the calibration. Pressure while held uses the stock
  formula (01 §6.3). **[RE]**
- Use the SMK's ADC timing (a conversion takes about 40 µs). The FM-1's setting takes about 200 µs, so at one
  conversion per 100 µs tick it never finishes and the pads and faders read nothing.
- Faders read up to 1023. Filter them with a **signed** filter: an unsigned IIR wraps when a fader moves down.
- On the tested unit the pitch wheel rests near 535 and travels about 445–640; the mod wheel reads about 474–629, and
  **up gives the lower value**. The travel differs between units: calibrate at power-on, or teach the ends.
- The pedal and the battery sit on the third mux (addresses 6 and 7). Stock firmware's battery levels are at raw 530,
  560 and 602; the divider ratio is not known. **[RE]**

**Screen and pad LEDs.** **[HW]**
- The panel is an ST7789V, 240 × 240, with no window offset and no reset pin (**[RE]** for the reset).
- The 16 pad LEDs are WS2812-class, colour order **GRB**, one SPI byte per LED bit, on PC10: SPI1's MOSI in pin
  group B. So they share SPI1 with the screen (group A). The simple way: wait until the screen's DMA is done and its
  CS (PC8) is high, switch to group B at a slower clock (`BAUD` 8), send the LED frame by DMA, then switch back.
  Never switch the group while a DMA runs.
- The chain doesn't run in pad order; the kit gives the mapping (01 §4).
- The reset gap: the kit's verified frame starts with 36 zero bytes (01 §4), while its pitfalls list says newer
  WS2812 parts need about 250, or the chain shifts and a pad lights its neighbour (08, "Hardware"). Both are the
  kit's own; the longer reset costs little.

**Pins to leave alone.** PH6, PH8, PH9 and PA9 are LED or MIDI pins on the FM-1 and matrix inputs on the SMK; PA2 is
a matrix row too. Driving them as on the FM-1 shorts them against the matrix. **[HW]** PB1 is the charger status
(low = charging), and held low for 8 s it resets the chip: never drive it or turn on its pull-down, and don't copy the
FM-1's ADC set-up, which reads the battery there. There is no soft power latch: the power switch is a slide switch.
**[RE]**

## Flash and calibration

The stock layout, from stock 016's region table (fm1-to-smk37 01 §2; m-bootloader SPEC §5.5 agrees): **[RE]**

| Range | Stock use |
| --- | --- |
| `[0x00000, 0x04000)` | boot area, byte-identical to the FM-1's |
| `[0x04000, 0x9D000)` | app area (`app.bin` at `0x4120`) |
| `[0x9D000, 0xC0000)` | VM |
| `[0xC0000, 0xC1000)` | BTIF: Bluetooth pairing |
| `[0xC1000, 0xC2000)` | USRTRIM: **factory calibration** |
| `[0xC2000, 0xEB000)` | USRFLASH: presets, with the unit's wheel values in their header |
| `[0xEB000, 0xF4000)` | not in the table |
| `[0xF4000, 0xFE000)` | USR: stock resources |
| `[0xFF000, 0x100000)` | `key_mac` |

The region table's offsets are absolute, not relative to `0x4000` ([Flash Storage](Flash-Storage.md)).

**The calibration record** at `0xC1000` holds five-point curves: one for each of the 16 pads from `+0x008`, one for
each of the 37 keys from `+0x0A8`, and one more (the pedal's) at `+0x21A`; the marker `trimdat` and a zero byte follow
at `+0x224`, flash `0xC1224`. **[RE]** fm1-to-smk37's port reads it and never writes it, and key velocity with the
factory curves works on the tested unit. **[HW]**

> **Stock firmware rewrites calibration in two cases.**
> - If the record fails its check (marker wrong, or a curve not strictly rising), stock firmware writes default curves
>   over it (fm1-to-smk37 01 §2.1). **[RE]**
> - If USRFLASH is blank, stock firmware has been seen to reset the presets and **erase USRTRIM and BTIF**, and
>   default presets don't bring the wheel values back (m-bootloader SPEC §5.5). **[HW]**
>
> So **never leave USRFLASH (`0xC2000`–`0xEAFFF`) erased when stock firmware may start.** fm1-to-smk37's
> return-to-stock section (02 §6, "Data left behind") offers erasing its data ranges, USRFLASH included, to `0xFF`
> for a clean state; by m-bootloader's observation that can cost the unit its calibration. Write those ranges back
> from your own backup of the same unit instead ([Recovery](Recovery.md)).

**fm1-to-smk37's port map.** **[CODE]** App slot `0x97400` bytes (the stock app's size); its own store
`0x9D000`–`0xBAFFF`, with an owner tag at `0xAF000`; update staging `0xBB000`–`0xBFFFF`; user samples
`0xC2000`–`0xE9FFF`, **over the stock presets and the wheel values in them**: only a backup brings those back. It
never writes `0x0`–`0x3FFF`, `0xC0000`–`0xC1FFF`, `0xF4000`–`0xFDFFF` or anything from `0xFF000`, with an allow-list
checked at build time and a second guard in the flash driver. On its first start it clears data without its owner
tag, the stock settings in VM included. An FM-1 firmware's settings at `0xFC000`/`0xFD000` fall inside the SMK's USR
and must move (01 §2.2; 02 §4; 08 "Flash and data").

## Packages and updates

The update path is the FM-1's ([Update Protocol and Installers](Update-Protocol-and-Installers.md)). Stock SMK-37 Pro
016 accepts a SLOOP-style package built for the SMK and starts the custom update loader from it (fm1-to-smk37 02 §1).
**[HW]** From fm1-to-smk37 02 §2 and 08, **[RE]** unless marked:

- **Identities.** Stock app `SMK-37 Pro_016`. The update loader is the same file on both devices and takes its USB
  name from the update record: stock's is `ota-SMK-37 Pro_016` (18 bytes, zero-padded). A custom app that hands over
  to that loader must write this name, not `ota-FM-1_015`, or M-VAVE's tool may not recognise the loader on a return
  to stock. The kit's custom loader answers `ota-SMK-37 Pro_900`, the same name as m-bootloader's MIDI mode.
- **Marker blocks.** Stock Pro 016 packages use 36 blocks (14 identity characters, then 22 blocks of `0x7D`); the
  older v011 used 20 ([Firmware Package Format](Firmware-Package-Format.md)). An installer that assumes 20 reads a
  016 package 16 bytes out of place. The kit's rule: keep counting blocks while fewer than 20 or while the marker is
  `0x7D`, check the CRCs of `flash.bin` and `ota.bin`, and fall back to 20 if they fail.
- **Stored CRCs.** In stock packages the CRCs stored for the small resource files (USR, `isd_config.ini`,
  `script.ver`, `blimit.bin`) don't match the bytes served; only `flash.bin` and `ota.bin` match. An installer that
  checks every file rejects every stock package.
- **Region table.** An SMK package should carry the stock SMK region table with the stock flags, so the official
  loader treats it the same way (02 §2.3).
- **The stock app resets** after the check step even when the handshake fails; SLOOP's loader is stricter.
- **Models.** The stock updater takes only the unit's own model, so the kit builds one package per model (Pro,
  Elite, MKE-P37). The Starrykey 37 Play's name, `STARRYKEY-37 PLAY` plus a version, is 21 characters and doesn't fit
  a 20-block identity; the kit uses its Pro build there. Its installers take any SMK model's package on any SMK model,
  but never FM-1 on SMK or the other way. Only the Pro was tested. **[CODE]**
- **The resume path.** One installer checked the model on a fresh install but not when the unit was already in
  update mode, so an SMK could be handed an FM-1 package (found in a SLOOP review; Felucca 1.1.5 forks too). Check the
  model on both paths (08, "Identity, models and installers"). **[CODE]**

## USB and MIDI

- Stock VID:PID `4353:B64B` with names "SMK-37 Pro Midi", "SMK-37 Pro Audio" and others (fm1-to-smk37 01 §10). **[RE]**
  Installers must also accept MIDI port names that contain `SMK`.
- Download mode is the FM-1's, `4C4A:8057`. On fm1-to-smk37's unit the disk showed as **`WL82 UBOOT1.00`**
  (02 §3.1). **[HW]** That is the SCSI vendor `WL82` and product `UBOOT1.00`; the USB string may read
  `WL80UBOOT1.00`, as on FM-1s (m-bootloader SPEC §4.2; [MIDI and USB](MIDI-and-USB.md)).
- Felucca-family firmware brings its own USB descriptors, and they work unchanged on the SMK. **[HW]**
- The SMK gives **no VBUS** on its USB-C port: USB host experiments need a powered hub or a 5 V supply (fm1-to-smk37
  08). **[HW]**
- MIDI out on the jack, as Snoop37 does it (06, "TRS jack: MIDI out"): hand each USB-MIDI packet to the jack's driver
  too, convert it by its code index number, drop SysEx, queue whole messages in a 256-byte ring, and send one byte per
  timer tick, giving up on a byte after 3 ms. Never wait in a loop. The update loader builds `usb.c` too, so guard the
  hook out of the loader build (08). **[CODE]**

## Where to look

- **M-VAVE's [smk37_open](https://github.com/m-vave/smk37_open)**: SMK37 OPEN V01, the same app as FM1 Open, with a
  binary SDK; board drivers in `app/board/`. Licence: GPLv3 with an SDK linking exception. A documented reference
  for the SMK's pins, pads and faders. What its code does (at b2d6bcb, 2026-10-24): **[CODE]**
  - Identities `SMK-37 Pro_8000001` and `SMK-37 Elite_8000001`; update families 8 (Pro) and 9 (Elite). The kit pins
    M-VAVE's stock V16 packages (dated 2026-08-12): Pro `2d98ce72…`, Elite `b2ee38ab…`. The stock Pro V16 package
    uses 36 identity marker slots, not 20 ([Firmware Package Format](Firmware-Package-Format.md)).
  - It **does not read the factory calibration** at `0xC1000`. It reads its own records 51–54, or falls back to
    default curves (`app/board/scan.c`).
  - Its package uses the FM-1's layout (`sdk/Package.cs`): its VM ends at `0x7B000`, its BTIF is `[0x7B000, 0x7C000)`,
    and its `USER` region is `[0x7C000, 0xFF000)`. Against the SMK's stock map, that USER region starts inside the
    stock app area (which runs to `0x9CFFF`) and spans the stock VM, the calibration and Bluetooth sectors
    `0xC0000`–`0xC1FFF`, USRFLASH and USR. fm1-to-smk37 says an SMK package should carry the stock SMK region table
    (02 §2.3). Whether SMK37 Open's firmware or updater ever writes the calibration is not known
    ([Open Questions](Open-Questions.md) 41). **[?]**
  - Its return to stock writes only the stock `USR` resource, `0xF4000`–`0xFDFFF` (37,382 bytes, checked by
    SHA-256), with the comment "Never touch calibration, user patterns, VM or Bluetooth addresses"
    (`sdk/WebUpdate.js`).
  - M-VAVE has not answered its issues #1, #2, #4 or pull request #3 (checked 2026-11-26).
- **[fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37)** (CC BY-NC 4.0): a kit to move an FM-1 firmware to
  the SMK-37 with a coding agent, keeping the FM-1 build byte-identical; the SMK hardware map; a list of ports already
  done (SLOOP, X0X, FoMni, Felucca and others) with their upstream pull requests; pitfalls and mapping patterns.
  The ported firmware code lives in the firmwares' own repositories, under their licences (its README).
- **[m-bootloader](https://github.com/jvitkauskas/m-bootloader)**: tested on the SMK-37 Pro; its SPEC has the
  SMK's scan, LED and panel facts. It also guards the SMK's factory calibration (keeps a copy and puts it back if it
  is erased). Its MIDI mode answers as `ota-SMK-37 Pro_900` and never writes `0xC0000`–`0xC1FFF`; it can't tell a
  Pro from an Elite, so an Elite installed through it reports itself as a Pro (SPEC §11.6). The 3.0 code was tested
  on an SMK-37 Pro under its 2.4 version strings; the exact 3.0 build was not (v3.0 release notes). **[HW]**
- **[M-Unbricker](https://github.com/jvitkauskas/m-unbricker)**: a working SMK-37 can revive a dead FM-1, and the
  other way round; it never writes the SMK's `0xC0000`–`0xC1FFF` ([Recovery](Recovery.md)). **[HW]**
- **felucca-bsp**: its README still lists the SMK-37 Pro and Elite as "Planned" (fdd709f, 2026-11-26). A board layer
  is **in progress** in pull request [felucca-bsp#30](https://github.com/hugelton/felucca-bsp/pull/30) (keremimo,
  open since 2026-11-06, updated 2026-11-26): `boards/smk37/` for the SMK-37 Pro, Elite and MKE-P37, with the app area
  `0x4000`–`0x9D000`, MIDI out on UART1, and the boot head, BTIF, USRTRIM (calibration), USR and key_mac never written.
  Its board facts come from fm1-to-smk37, marked *verified* or *analysis*; its audio set-up is not yet measured on an
  SMK. **[CODE]**
- fm1-base "needs a PR for SMK-37 support".

## Advice for a firmware that wants both

- Keep the board's facts (pins, matrix, LEDs, panel) behind one board layer, chosen at build time
  (e.g. `BOARD_SMK37`), so the FM-1 build stays byte-identical. More in the
  [Programming Guide](Programming-Guide.md#targeting-both-fm-1-and-smk-37).
- Decide early what the pads, faders and extra keys do in your UI: it's easier than retrofitting.
- **Never write the SMK's calibration area**, and never ship an image that contains one unit's calibration.
- **Back up the whole flash before the first install, and keep it.** USRFLASH holds the unit's presets and wheel
  values, and a firmware that stores data there overwrites them.
- Test with M-VAVE's stock firmware back on the unit afterwards: stock firmware has been seen to reset presets and
  erase USRTRIM/BTIF when USRFLASH is blank (m-bootloader SPEC §5.5). **So don't erase USRFLASH on the way back to
  stock; restore it from your backup** (above).
- Back up before installing Baud Girl: it overwrites the SMK-37's stock presets, so stock firmware installed
  afterwards has none until a backup is restored, and its installer rewrites the Bluetooth sector at `0xC0000`, which
  may lose pairings; calibration stays (m-bootloader README). **[HW]**
- Not every SMK firmware takes the same update path. ChanceTheMaker's SMK firmware (`SMK-37_9NN`) answers the upgrade
  message by starting its own updater, `ota-SMK-37_930`, which takes only its own kind of package (m-bootloader SPEC
  §11.7). **[HW]**

## Further reading

From [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) (CC BY-NC 4.0); its README is the landing page, with
the porting steps, the list of ports and their upstream pull requests, a table of the flash ranges a port must never
write, and a contents table.

- [01-hardware.md](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/01-hardware.md): the SMK-37 datasheet
  against the FM-1, with the key, button, encoder, mux and LED tables.
- [02-update-and-recovery.md](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/02-update-and-recovery.md):
  update protocol, package container, full backup, recovery and return to stock (read the calibration warning
  above first).
- [05-reverse-engineering.md](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/05-reverse-engineering.md):
  how the facts were obtained, with public tools.
- [07-develop-for-both.md](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/07-develop-for-both.md) and
  [08-pitfalls.md](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/08-pitfalls.md): one source for both
  boards, and what went wrong in earlier ports.
- [10-smk37-to-fm1.md](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/10-smk37-to-fm1.md): the reverse
  port, a firmware made for the SMK getting an FM-1 target.

The kit disagrees with itself on two points: its hardware map gives the screen's SPI clock as `BAUD` 4, about 12 MHz,
verified, while 10 §5 speaks of "the SMK's 20 MHz LCD clock"; and the pad LED reset (36 or about 250 zero bytes,
above). Its recovery section (02 §5) says no rescue dongle had been tried on an SMK; since then M-Unbricker reports
reviving units in both directions ([Recovery](Recovery.md)).

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §4.2, §5.5, §8, §11.6,
  §11.7, README and [docs/STOCK-V15-V16.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md)
  (eb4f2c1); [v3.0 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0).
- [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) (genkigenki; CC BY-NC 4.0, per its `LICENSE`) at
  `54252ff` (2026-10-17): README, `docs/01-hardware.md` §1–§11, `docs/02-update-and-recovery.md` §1–§6,
  `docs/05-reverse-engineering.md` §2, `docs/06-mapping-patterns.md` "TRS jack: MIDI out", `docs/07-develop-for-both.md`,
  `docs/08-pitfalls.md`, `docs/10-smk37-to-fm1.md` §5.
- [smk37_open](https://github.com/m-vave/smk37_open) `app/version.h`, `app/board/scan.c`, `sdk/Package.cs`,
  `sdk/WebUpdate.js` (b2d6bcb); [FM1 Open](https://github.com/m-vave/fm1_open) `app/board/lcd.inc` (4c7496d),
  [fm1_open#11](https://github.com/m-vave/fm1_open/issues/11).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp) README (fdd709f), [PR #30](https://github.com/hugelton/felucca-bsp/pull/30).
- [M-Unbricker](https://github.com/jvitkauskas/m-unbricker) README (204f88d).
- [smk37_open](https://github.com/m-vave/smk37_open) README.
- [r/MVaveFM1 thread](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
