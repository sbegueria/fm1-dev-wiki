---
title: "Open Questions"
parent: Community
nav_order: 4
---

# Open Questions

Things nobody has settled in public, or where sources disagree. If you can answer one, add the measurement or
the source, move the fact to its page with the right tag, and leave a line here saying it was resolved.

Each entry: **the question**, what we know, and how to find out.

## Hardware

1. **The audio converter.** Felucca's HAL says ALNK0 I2S feeds an "external codec"; FM-1-RE says an "internal
   audio DAC". Which part is it, and what do the control lines PC0, PC1, PC2 and PC6 do?
   *Since:* the resource hub (`REVERSE_ENGINEERING.md` §1.2, October 2026) reads an external codec with PC0 = MCLK,
   PC1 = BCLK, PC2 = LRCK, PC6 = the codec's power/reset, from photos and JieLi's reference schematics, but gives no
   codec part number ([Audio Output](Audio-Output.md)).
   *Since:* two code sources contradict the hub on PC6. felucca-bsp's `boards/fm1/fm1_audio.h` says PC6 "is the I2S
   data line, not a codec reset or mute pin"; FM1 Open's `app/board/fm1_audio.h` calls it the DAC line. The part
   itself is still unknown ([Pinout](Pinout.md), November 2026).
   *Since:* FM-1-RE's later notes on stock V13 (`docs/io/03-audio-dac.md` §5.2, §5.3, marked med) find the synth
   sent through the `0x12E00` block (ALNK0), with set-up code configuring PC0, PC1, PC2 and PC6, while its overview still
   says "internal audio DAC". So stock firmware likely uses the same output as the open firmwares; the part on the
   other end is still unknown ([Audio Output](Audio-Output.md#what-m-vaves-own-firmware-uses)).
   *How:* a board photo with the part number; a datasheet; probe the lines.
2. **The true sample rate.** Open firmwares assume 44,100 Hz; FM-1-RE reports the stock audio server at 44,118 Hz.
   Is the rate the same under 360 and 320 MHz? *How:* record a long sine from the FM-1 and measure its frequency
   against the computer's clock, under each bootloader.
   *Since:* mostly answered under 360 MHz. One frame is 544 ticks of the 24 MHz crystal, so the rate is about
   44,117.6 Hz [CODE] (felucca-bsp `docs/BOARD_IO.md`); macOS clocked the FM-1's USB audio at 44,122 Hz [HW]
   (felucca-bsp [d18879b](https://github.com/hugelton/felucca-bsp/commit/d18879b)); FM1 Open sets
   `FM1_AUDIO_HARDWARE_RATE 44118U`. The audio PLL is a separate one from the CPU's, so the rate should not change
   at 320 MHz, but no one has measured it there yet ([Audio Output](Audio-Output.md)).
3. **The instruction cache.** Size, ways, line size, replacement policy, and whether lines can be locked. DUB-1's
   notes say 32 KB, 8 ways × 4 KB, 32-byte lines; the SDK has `CACHE_LINE_COUNT 32` and `ICACHE_WAY`/`DCACHE_WAY`
   registers. *How:* time a loop over code of growing size from XIP; read the SDK's `cache.h` users.
   *Known so far:* running DUB-1's 55 KB audio path from SRAM instead of XIP cut its peaks by up to 35% and stopped
   its dropouts ([Performance](Performance.md)), so misses are costly; the size itself is still unmeasured.
   The **data** cache is now known: 8 ways of 4 KiB, which can be turned into RAM at `0x01F28000` (felucca-bsp
   `docs/EXPERIMENTAL.md`; [Performance](Performance.md)). The instruction cache is still open.
4. **The D-cache and DMA.** Must DMA buffers be flushed or kept out of the D-cache? Felucca-family firmware keeps DMA
   buffers in plain RAM and has worked, but no one has written down why.
5. **The cycle counter.** Resolved: see below.
6. **The stall counters** (`C0_IF_UACNT`, `C0_RD_UACNT`, `C0_WR_UACNT`): the SDK's `cpu_effic_calc` reads them.
   Do they count on the FM-1, and do they show I-cache misses?
   *Since:* like the cycle counter, they count once `DBG_CON` bits 0–2 are set
   ([Performance](Performance.md)). Whether they show cache misses is still unknown.
7. **SRAM in total.** The SDK says 578 KB; firmwares use 512 KB at `0x01C00000`. Where is the rest, and can a
   firmware use it?
8. **Subnormal floats.** Does the FPU slow down on subnormal numbers, or flush them? *How:* time a loop on
   subnormal and normal operands.
9. **Flash timing.** Sector erase and page program times of the stock part (`856014`), and whether XIP reads stall
   the other core during them.
   *Since:* the times are measured on the Puya part: a 4 KiB erase takes 17–18 ms, a page program 1.70 ms
   (felucca-bsp [#46](https://github.com/hugelton/felucca-bsp/pull/46); [Flash Storage](Flash-Storage.md)). Still
   open: the same on the UC25HQ80 (`B36014`), whether erase-suspend works there, and the other core during an erase.
10. **Battery.** The ADC reading against the battery's voltage, and the battery cost of M-VAVE's voltage settings.
    *Since:* stock V13 turns a 256-reading average of ADC channel 3 into four levels at raw values of about 520,
    531, 550 and 591, and switches GPIO 149 around each reading (FM-1-RE `docs/io/07-input.md` §4.2;
    [Clocks and Power](Clocks-and-Power.md#battery)). **[RE]** The voltages those values stand for, and what
    GPIO 149 does, are still open.

11. **The chip's exact part.** The first r/MVaveFM1 thread says AC79118B; the resource hub reads AC7911B, QFN-48,
    marking `JL C156211-11A-B`, from board photos; mvave-fm1 repeats AC7911B. FM-1-RE reads its unit's marking as
    `C156211-11B8`, **LQFP48** (`docs/01-firmware-analysis.md`), against the hub's QFN-48. *How:* a sharp photo of
    the marking and the package's leads.
12. **Where the flash is.** The hub says the Puya P25Q80H die is inside the SoC package (SiP), on PD0–PD6;
    felucca-bsp's experimental notes speak of replacing "the stock SOIC-8". Is there an 8-pin flash on the board?
    *How:* a board photo; the AC7911B datasheet's package description.
13. **SDRAM in the package?** The hub says the AC7911B comes as AC7911BB with 8 MB of SDRAM inside, or AC7911BA
    with 2 MB. This wiki says there is no external RAM (`0x04000000` absent, [SDK] [RE]). Which variant is on the
    FM-1, and does anything answer at `0x04000000`? *How:* the marking's suffix; a careful read probe from a debug
    build (it may fault: read with the guards in mind).
14. **The TRS jack's circuit.** The hub says it only passes MIDI in, through an input buffer, and that PH7 (UART1
    TX) shares the package pin with PH8. *How:* trace the jack on a board photo.
    *Since:* the resource hub's `HARDWARE_MODS.md` lists a DIN-5 MIDI-out mod on pin 39 with a 74LVC1G07 buffer
    [?]; its firmware repository is gone and upstream Dinghy has no MIDI out ([Pinout](Pinout.md)).
    *Since:* stock V13 has a UART transmit path: it copies incoming MIDI into the UART's TX DMA as MIDI thru
    (FM-1-RE `docs/io/05-midi.md` §1, §2.1) **[RE]**. Whether that pin reaches the jack is the same question
    ([MIDI and USB](MIDI-and-USB.md)).
    *Since (2026-11-27):* no open FM-1 firmware transmits on UART1. The mod's "pin 39" is **PD6** in the hub's own pin
    map, which it calls the flash's power gate, so the claim doesn't fit the map; `ehn96/dinghy` is still 404. The hub
    now calls the circuit "an input optocoupler/buffer", unsourced. *How, without a photo:* set PH7 as UART1 TX at
    31,250 baud sending a known pattern and probe the jack's tip and ring with nothing plugged in.
36. **The V15 supply voltages.** Two readings disagree. A static reading of V15's code (fc79, fm1-x0x#21) gives
    I/O 3.2 V, VDC14 1.60 V and SYSVDD 1.38 V. m-bootloader's RAM-app probe read I/O 3.5 V, core 1.20 V while V15
    or V16 ran, and saw 3.2 V only from the 016 bootloader until its firmware starts. The two may have been read at
    different moments ([Clocks and Power](Clocks-and-Power.md)). *How:* read the P33 registers from inside a running
    V15 build, under load, and measure the rails with a meter.

## CPU

15. **Is there an FPU?** JieLi's SDK and Felucca, DUB-1 and FM1 Open compile with `-mfprev1` and use `float` on
    hardware. Dinghy's `docs/RULES.md` (the SLOOP line), and mvave-fm1's skills after it, say "there is no FPU and
    no libgcc", so floats fail to link. The likely reading is that Dinghy's build links no float support, not that
    the chip lacks an FPU, but no one has written that down. *How:* build a float test with `-mfprev1`, run it on
    a unit and time it against the same code built soft-float.
    *Since:* M-VAVE's V13 links software helpers for doubles only, none for single floats (FM-1-RE
    `docs/function-index.md`; [Performance](Performance.md)) **[RE]**, which fits single floats in hardware.

16. **The 64-bit multiply erratum.** Which exact instruction pairs fail? Is 320 MHz clear of it? Does it ever hit
    float code? (felucca-bsp `docs/CPU.md`.) A new case: the 48 kHz USB resampler `uac_tap48` had the pattern
    (felucca-bsp [#16](https://github.com/hugelton/felucca-bsp/issues/16), 2026-11-18). Is 396 MHz
    ([Clocks and Power](Clocks-and-Power.md)) worse for it?
17. **The second core's misreads.** Why do some units' core 1 misread shared RAM? Is it the supply, the clock, the
    bus? Is it fixed by M-VAVE's voltages? (melodee#17, fm1-x0x#10.) Stock firmware starts core 1 too (V13 and V15:
    [Second Core](Second-Core.md)), so a stock unit's core 1 runs under M-VAVE's voltages; whether stock code ever
    shows the fault is not known.
18. **Flag gains.** What do FM1 Open's flags (`-flto`, `-pi32v2-enable-simd=true`, `-pi32v2-large-program=true`,
    IPRA, `rep-memop`) gain on real DSP code, and do they change float results?

19. **The second core's idle polling.** On one unit, pacing the worker's idle loop (32 NOPs in RAM between
    mailbox polls) stopped invalid jobs that appeared even with an empty job (kmplngj/Felucca, 2026-10-22). Is
    this the same fault as the misreads above? Does it hold on other units, and why does polling rate matter?
    *How:* the same paired test on units known to misread.
37. **Two cores at 360 MHz under m-bootloader.** One user's Skiff unit crashed with both cores on at 360 MHz under
    m-bootloader, but not under the stock 015 or 016 bootloaders, and not at 320 MHz; another unit was unaffected
    (m-flasher [#55](https://github.com/jvitkauskas/m-flasher/issues/55), 2026-11-14/15). What differs between the
    bootloaders' hand-over that the second core cares about? *How:* compare the clock and supply registers each
    bootloader leaves; repeat on more units.
38. **`DBG_CON` left on.** Setting bits 0–2 starts the cycle and stall counters (Q5, Q6). Is it safe to leave them
    set while a firmware runs, or do they arm debug faults? *How:* run a firmware with them set for hours.
39. **Cache-RAM ways and the other masters.** Can the second core and DMA read and write data-cache ways turned
    into RAM at `0x01F28000`? *How:* a test from core 1 and a DMA transfer into a cache-RAM way.
    *Since:* still unknown: Optimist has not tried either. It gives cache RAM only to its undo ring, which the main
    loop reads; the blocks core 1 and the audio read stay in SRAM (hdavid, [optimist#295](https://github.com/w0ts/optimist/issues/295), 2026-11-27).

## Memory map

20. **The XIP offset in the stock image.** Resolved: see below.
21. **The stock clock in the app.** FM-1-RE reads a `pll_clock_init` → 240 MHz in V13's start-up; measurements under
    M-VAVE's bootloader, with an open firmware running, give 360 MHz. What does V15's own start-up set, and when?
    *Since:* the bootloaders' hand-over is now measured (015: 360 MHz, 016: 320 MHz; m-bootloader
    `docs/STOCK-V15-V16.md`), but neither probe sees what the stock app sets afterwards.
40. **m-bootloader 1.2: 320 or 360 MHz?** felucca-bsp read 320 MHz with PLL1 off under m-bootloader 1.2;
    m-bootloader's SPEC, its 1.3 notes and 1.2's source say 360 MHz ([Clocks and Power](Clocks-and-Power.md)).
    Was the felucca-bsp reading taken on a unit with V16's boot area? *How:* read the clock registers under 1.2 on
    a unit with V15's boot area.

## Installing

22. **Installs that stall half-way** ("the loader stopped answering" after ~half the requests). A USB drop, or the
    loader's poll falling behind? *How:* log the last request and its timing in the installer; a USB capture.
    *Since:* one cause is known for new hosts: the device drops an answer whose declared length is wrong, and a
    stock install stalled on its last 481-byte request that way (FM-1-RE `docs/io/12-v15-reflash-proof.md`, one unit,
    2026-09-04) **[HW]**. Felucca-family installers declare the length from the data, so it does not explain their
    stalls ([Update Protocol](Update-Protocol-and-Installers.md)).
23. **m-bootloader and full packages.** Which installers keep m-bootloader and which put the stock one back?
    (SLOOP's kept it; M-VAVE's and Baud Girl's restore stock.) *Since:* M-UPGRADE's V15→V16 update puts the stock
    016 bootloader back (m-bootloader [v3.0 notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0));
    m-bootloader's own MIDI mode writes only the app, never a bootloader ([Boot Chain](Boot-Chain-and-Bootloaders.md)).
    Other installers are still to be listed.
    *Since:* **Optimist's installers keep whatever bootloader is there [CODE].** They use the running firmware's
    update loader whenever it takes the package, and install m-bootloader only when asked (`--install-mboot`, or the
    installer's checkbox), through M-Flasher's download mode after a full backup; M-Flasher's `firmware` is run without
    `--with-bootloader`, which would write the package's stock bootloader. Optimist's own updater refuses M-VAVE's V15
    package when the unit's boot area differs from the package's (as it does under m-bootloader), so the installers
    take download mode for it; fm1-base's updater accepts m-bootloader's installer package
    ([`docs/INSTALL-PATHS.md`](https://github.com/w0ts/optimist/blob/bfa17d5e/docs/INSTALL-PATHS.md), 2026-11-26; its
    tables are generated by `tools/fm1_route.py` and checked by a test, so the page is the reference if things move:
    hdavid, [optimist#295](https://github.com/w0ts/optimist/issues/295)). Felucca's, SLOOP's and Baud Girl's loaders are assumed there, not read.
24. **The loader's watchdog.** How long can the update loader go without a request before it gives up, per base?
    *Since:* the host side of M-UPGRADE is known: it waits 2,000 ms after each upgrade command and 8,000 ms per
    receive, with no keep-alive (FM-1-RE `docs/io/11-ota-protocol.md`, decompiled) **[RE]**. The loaders' own
    limits are still open.

25. **USB IDs of the update loaders.** Resolved for M-VAVE's loader: see below.
26. **V16.** Since 2026-10-21 M-VAVE's download address serves V16. Mostly answered (November 2026): V16 answers
    `FM-1_016`; Felucca and fm1-base install over it and back (Felucca
    [#329](https://github.com/hugelton/Felucca/issues/329), felucca-bsp
    [#17](https://github.com/hugelton/felucca-bsp/issues/17)); where it stages its loader depends on the package's
    VM record (m-bootloader SPEC §11.7) ([Update Protocol](Update-Protocol-and-Installers.md)). Still open: does it
    answer the soft key `F0 22 24 35 7D F7`? And the file at the download address hashed `d017fb65…` on
    2026-10-29, while the hub lists `a21368e0…` for V16 (2026-10-21): has V16 been re-issued?
27. **The V15 region hash on real units.** One unit reporting `FM-1_015` had a firmware-region hash
    (`0x4000…0x92FFF`) of `8c1d13a5…`, not the official file's `6edf3c37…` (mvave-fm1, 2026-10-22). Was it an
    earlier V15 build, or does stock V15 write inside its own app area? *How:* compare dumps from several V15
    units sector by sector.
28. **Download-mode sessions that end.** On one Linux run, the chip reset back to normal after one tool run in
    download mode (`info`, then `backup`, failed). A timeout in the ROM, or the host re-probing the disk?
    *How:* time how long the ROM stays in download mode idle, on Linux and Windows.
29. **The watchdog's timeout codes.** Dinghy documents `0xA` = 1 s to `0xF` = 32 s for `P3_WDT_CON`; 8 s for
    `0x0D` assumes a doubling per step. *How:* time a reset with no feed.
30. **Can anything run stock firmware off the device?** RemoteFM1's README says no emulator exists for the FM-1's
    CPU, so stock firmware can't run on a computer; fm1-emu and fm1-emulator run community firmware images
    unchanged ([Simulators and Emulators](Simulators-and-Emulators.md)). Do they run stock V15?

31. **The clock after a V16 install.** Resolved: see below.
32. **A GPL firmware with Bluetooth MIDI.** Resolved: see below.
33. **m-bootloader on the UC25HQ80.** *Since (November 2026):* the sources still disagree. m-bootloader's README
    dropped its UC25HQ80 warning ([eb4f2c1](https://github.com/jvitkauskas/m-bootloader/commit/eb4f2c1),
    2026-11-23) and its author answered "Yes" when asked whether 2.4 was tested on a `B36014` unit
    (m-bootloader [#37](https://github.com/jvitkauskas/m-bootloader/issues/37)), but the
    [v3.0 notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0) list only a Puya FM-1 as tested,
    and M-Flasher's dialog still warns. The earlier history:
    M-Flasher warns against installing m-bootloader on the newer flash
    (`B36014`; jvitkauskas/m-flasher#13), yet one UC25HQ80 unit reports m-bootloader `2.3-6bc8f45` installed and
    working (fm1-dub#14, 2026-11-22): installed with M-Flasher 0.12.1 (macOS), the unit rebooted normally, Device
    info "Firmware check: OK", and since then SHRUB v1.29, DUB-1 and stock V15 installed on it without trouble
    (M-Flasher and DUB-1's web installer). The owner calls it one unit, not proof: the failure in m-flasher#13
    (a UC25HQ80 that didn't boot after the same kind of install) used M-Flasher 0.7.0. Is the warning still needed
    with 0.12.1, and did something change between the versions? *How:* more UC25HQ80 installs with 0.12.1;
    M-Flasher's changelog between 0.7.0 and 0.12.1.

## The SMK-37

34. Elite hardware behaviour is inferred from the Pro's (m-bootloader SPEC §13). Who has an Elite to test?
    m-bootloader 3.0 itself has not been tested on any SMK-37 yet (its v3.0 notes). fm1-to-smk37 builds Elite and
    MKE-P37 packages too, but checked its facts only on an SMK-37 Pro (its README, 54252ff, 2026-10-17).
41. **SMK37 Open and the calibration sector.** SMK37 Open keeps the FM-1's layout, so its USER region
    `[0x7C000, 0xFF000)` spans the SMK-37's calibration and Bluetooth sectors `0xC0000`–`0xC1FFF`. It doesn't read
    the factory calibration; does its firmware or updater ever write there? ([SMK-37](SMK-37.md)) *How:* dump
    `0xC0000`–`0xC1FFF` before and after an SMK37 Open install and use.
    *Since:* the region is wider than that. It starts inside the SMK's stock app area (which runs to `0x9CFFF`) and
    covers the stock VM, USRFLASH (presets and wheel values) and USR too; its VM and BTIF sit below `0x7C000`
    (`sdk/Package.cs`). fm1-to-smk37 reads stock 016's table and says an SMK package should carry that table, with
    the stock flags, so the official loader treats it the same way (02 §2.3). **[RE]** So also: does the stock
    loader or stock firmware act on SMK37 Open's table, and what does stock firmware do with USRFLASH afterwards
    (a blank USRFLASH has been seen to make it erase the calibration)? *How:* also dump `0xC2000`–`0xEAFFF`.
43. **Does stock SMK-37 016 answer the soft key?** m-bootloader's SPEC §4.2 says stock 016 on an SMK-37 Pro
    answered `F0 22 24 35 7D F7` on the first "… Private" port; fm1-to-smk37 (02 §2, §3.1) says only Felucca-family
    loaders and apps understand it, not the stock app, and stages its own loader first for a backup. Which port did
    each try? ([MIDI and USB](MIDI-and-USB.md)) *How:* send the soft key from stock 016 on each of the SMK's MIDI
    ports and note which, if any, restarts into the loader.

## Display

35. **The backlight off without losing the keys.** PA2 drives both. Can the ST7789's own brightness command
    (`0x51`) darken this panel, given that one experiment with `0x51`/`0x53` gave wrong colours? *How:* a minimal
    test sending only `0x51` after the normal start-up, colours checked.
    *Since:* M-VAVE's own panel set-up sends `51 FF` (Dinghy `docs/NEWER-FM1-SCREEN.md`; [Display](Display.md)), so
    the command is accepted by this panel; dimming with lower values is still untested.
    On the SMK-37 the question doesn't arise: it has no backlight pin, and the screen is on whenever the unit is
    (fm1-to-smk37 01 §3, verified).
42. **The newer panels at 60 MHz.** Dinghy's notes say the newer panels need the display clock at 12 MHz;
    fm1-base's `lcd_panel` option sends M-VAVE's set-up but keeps 60 MHz ([Display](Display.md)). Do newer panels
    work at 60 MHz once fully set up? *How:* run fm1-base with `lcd_panel = true` on a newer unit.
    *Since:* fm1-base's author: not tried on a newer unit, and the commit that added `lcd_panel` was not run on an
    FM-1 at all. fm1-base sends the whole set-up at the start-up clock and only then raises the clock to 60 MHz for
    the pixels (`device/lcd.c`), so the question is only whether a newer panel takes pixels at 60 MHz (Charles
    Vestal, [fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99), 2026-11-27). A report from a `B36014` unit with the newer screen would settle it.
    For the SMK-37's panel, fm1-to-smk37 disagrees with itself: `BAUD` 4, about 12 MHz, verified (01 §3), against
    "the SMK's 20 MHz LCD clock" (10 §5).

## MIDI

53. **Does the every-poll TRS driver replay stale bytes?** Felucca reads the UART count only on receive or timeout
    events; felucca-bsp, fm1-base and Dinghy read it on every poll ([MIDI and USB](MIDI-and-USB.md)). The fault was
    shown in a host model. *How:* send a long stream of known CCs into TRS while the render is heavy (polls late), and
    count wrong or repeated messages with each driver.
54. **MIDI in latency and jitter, per base.** Nobody has measured note-in to sound-out on USB or TRS. *How:* a computer
    or a hardware interface sends 1,000 notes; record the FM-1's output beside a reference click; compare Felucca,
    fm1-base with and without `midi_stamp`, Optimist and stock V15.
55. **Clock out and clock follow on hardware.** The 418 µs spread of clock out before `FELUCCA_MIDI_RT` has no "after"
    figure, and the clock-follow fixes (Felucca, OberVave's PI loop, Optimist's filter) were shown in simulation.
    *How:* timestamp 10,000 F8s from the FM-1 on a computer; for follow, a DAW sends clock at fixed tempos and ramps and
    the FM-1's steps are recorded against the DAW's click for 10 minutes.
56. **Which USB MIDI devices fail with the USB host?** Low-speed controllers, devices that wait for VBUS, multi-cable
    devices, two-chip hubs on felucca-bsp's one-level code. *How:* a list of devices with who, when and the result.

## The radios

See [Bluetooth and WiFi](Bluetooth-and-WiFi.md).

44. **BLE MIDI latency and jitter.** Only CRC rates and acknowledgements have been measured. *How:* loop a note from the
    FM-1 to a Mac and back, or compare with USB MIDI on the same host; a histogram over 1,000 notes at 7.5, 11.25 and
    15 ms intervals; stock, JieLi's stack and the own stack.
45. **A2DP drift.** The own stack's SBC stream is labelled 44.1 kHz but fed at 44,117.6 Hz with no resampler; a full
    queue drops a whole packet. Does it click every few seconds, or does the sink absorb it? *How:* stream 10 minutes
    to two sinks, log the queue depth and drops, listen for regular clicks.
46. **The SBC encoder's cost.** 3.1 % of a core in the emulator, "~31 % of a block, up to 54 %" on an FM-1. Which, and
    why the gap (flash waits)? *How:* time the encoder with TIMER4 on a unit, code in flash and in RAM.
47. **Does radio work cause audio dropouts?** No measurement on either radio, any stack. *How:* record a steady tone
    over USB while scanning, joining, streaming and with BLE MIDI under load; count discontinuities; read the BLE
    handlers' longest time on the unit.
48. **Battery cost of the radios.** Nothing measured. *How:* a USB power meter or a shunt: radio off, advertising,
    connected, A2DP streaming, WiFi joined, MODE AP, JANK; 10 minutes each.
49. **Link phase against Live, measured outside the FM-1**, and **WiFi beside Bluetooth audio over hours.** The only
    on-air figures are in commit messages. *How:* the FM-1's click and Live's into a two-channel recorder for 30
    minutes, alone and with BLE MIDI and A2DP on; then an 8-hour soak logging retries, drops and underruns.
50. **Range, TX power and the weak uplink.** One note has the access point hearing the FM-1 20 dB low at 3–4 m; TX
    power per level is unmeasured. *How:* RSSI both ways at 1, 3, 10 and 20 m; with the stored calibration and without.
51. **Does modd's driver join and get a lease on hardware?** The repo's last dated hardware record stops before
    association. *How:* ask DimmKirr for the build, unit and date of a join and the weather fetch, or flash
    `modd_defconfig` and read its network screen.
52. **Windows and Android with the own Bluetooth stack**, and **does it need the SDK's temperature retune?** (the SDK
    retunes every 3 s as the chip warms). *How:* connect from Windows 11 and an Android DAW and report CRC and drops;
    heat and cool a connected unit and watch the same.

## Resolved

- **32. A GPL firmware with Bluetooth MIDI** (2026-11-27, fm1-base's author). **Yes:** a GPL firmware may use all of
  fm1-base, its own Bluetooth and WiFi stacks included, with or without Link. None of fm1-base's MPL-2.0 files carries
  the "Incompatible With Secondary Licenses" notice (MPL-2.0 section 3.3); the stacks' own files are
  `GPL-3.0-only OR MPL-2.0`; their other parts (Apache-2.0, BSD-2-Clause) are GPL-3.0 compatible. What a GPL firmware
  can't simply do is link JieLi's binary radio libraries: that needs a GPL-3.0 section 7 permission from every
  copyright holder of its GPL code (Charles Vestal, [fm1-base#99](https://github.com/charlesvestal/fm1-base/issues/99); fm1-base LICENSING.md and DESIGN.md,
  [e4863fa](https://github.com/charlesvestal/fm1-base/commit/e4863fa)). Now in [fm1-base as a Base](FM1-Base.md).
- **20. The XIP offset in the stock image** (2026-11-26, this wiki's reading). The stock app is linked at
  `0x02000120` like the open firmwares; FM-1-RE's addresses of code and data in `app.bin` are 0x120 low. Evidence
  from FM-1-RE's own data: the package's `entry-point` is `0x02000120`, the start-up's jump and core 1's entry land
  0x120 above the functions FM-1-RE finds, its tables sit 0x120 below the code's pointers, and its `.data` copy fits
  the file only at `0x02000120`. fm1-to-smk37 says the same for stock 016. FM-1-RE itself does not conclude **[RE]**.
  Now in [Memory Map](Memory-Map.md#the-stock-app-runs-at-0x02000120).
- **25. USB IDs of the update loaders** (2026-09-04 capture). `4D4A:4155` is **M-VAVE's own loader**: it enumerated
  that way and answered `ota-FM-1_015` between the two steps of a stock V15 install (FM-1-RE
  `docs/io/12-v15-reflash-proof.md`, one unit) **[HW]**. ORBIT's and Felucca's loaders use `1209:0002` "Felucca
  Update" by their sources, and m-bootloader's MIDI mode `1209:0002` "FM-1 Update" (SPEC §11.6); mvave-fm1's skill,
  which put ORBIT's loader at `4D4A:4155`, has not been checked with `lsusb`. FM-1-RE's July notes say a shrunk test
  loader came back as `4C4A:C755` (`docs/ota-loader-shrinking.md`). Now in [MIDI and USB](MIDI-and-USB.md).

- **5. The cycle counter** (2026-11-21). `C0_TL_CKCNT` counts only when bits 0–2 of `DBG_CON` (`0x1EEE344`) are
  set: 360,031 counts per ms with them, 0 without (felucca-bsp
  [#18](https://github.com/hugelton/felucca-bsp/issues/18), [dfa1648](https://github.com/hugelton/felucca-bsp/commit/dfa1648)).
  Now in [Clocks and Power](Clocks-and-Power.md). Whether that is safe to leave on: Q38.
- **31. The clock after a V16 install** (2026-11-21). M-UPGRADE's V15→V16 update writes V16's boot area (JieLi SDK
  1.2.13's bootloader), which starts the app at 320 MHz with PLL2 off (m-bootloader `docs/STOCK-V15-V16.md`,
  [121cfbe](https://github.com/jvitkauskas/m-bootloader/commit/121cfbe); v3.0 notes). Now in
  [Clocks and Power](Clocks-and-Power.md) and [Boot Chain](Boot-Chain-and-Bootloaders.md). What V16's app then
  sets for itself is part of Q21.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
