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

11. **The chip's exact part.** The first r/MVaveFM1 thread says AC79118B; the resource hub reads AC7911B, QFN-48,
    marking `JL C156211-11A-B`, from board photos; mvave-fm1 repeats AC7911B. *How:* a sharp photo of the marking.
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

16. **The 64-bit multiply erratum.** Which exact instruction pairs fail? Is 320 MHz clear of it? Does it ever hit
    float code? (felucca-bsp `docs/CPU.md`.) A new case: the 48 kHz USB resampler `uac_tap48` had the pattern
    (felucca-bsp [#16](https://github.com/hugelton/felucca-bsp/issues/16), 2026-11-18). Is 396 MHz
    ([Clocks and Power](Clocks-and-Power.md)) worse for it?
17. **The second core's misreads.** Why do some units' core 1 misread shared RAM? Is it the supply, the clock, the
    bus? Is it fixed by M-VAVE's voltages? (melodee#17, fm1-x0x#10.)
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

## Memory map

20. **The XIP offset in the stock image.** Felucca maps flash `0x4120` to `0x02000120`; FM-1-RE writes
    `flash_offset − 0x4120 + 0x02000000` and puts stock vectors at `0x020000A0`. Is the stock app linked
    differently, or is one note off by the 0x120 header?
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
23. **m-bootloader and full packages.** Which installers keep m-bootloader and which put the stock one back?
    (SLOOP's kept it; M-VAVE's and Baud Girl's restore stock.) *Since:* M-UPGRADE's V15→V16 update puts the stock
    016 bootloader back (m-bootloader [v3.0 notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0));
    m-bootloader's own MIDI mode writes only the app, never a bootloader ([Boot Chain](Boot-Chain-and-Bootloaders.md)).
    Other installers are still to be listed.
24. **The loader's watchdog.** How long can the update loader go without a request before it gives up, per base?

25. **USB IDs of the update loaders.** The hub reports `4D4A:4155` "ota-FM-1" during an update; mvave-fm1's skill
    puts ORBIT's loader there too. ORBIT's and Felucca's loader sources say `1209:0002` "Felucca Update". Is
    `4D4A:4155` M-VAVE's own loader? *How:* `lsusb` during an M-UPGRADE update and during an ORBIT update.
    Note: m-bootloader's MIDI mode also uses `1209:0002` ("FM-1 Update"; SPEC §11.6).
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
32. **A GPL firmware with Bluetooth MIDI.** fm1-base's own BLE stack links nothing of JieLi's and its parts are
    MPL-2.0, Apache-2.0 and BSD-2-Clause. Can a GPL-3.0 firmware ship it, as this wiki reads the licences?
    fm1-base states the JieLi-library case, not this one ([fm1-base as a Base](FM1-Base.md)). *How:* ask the
    fm1-base maintainers to state it.
    *Since:* fm1-base's Ableton Link option is GPL-3.0-only and needs the own BLE and WiFi stacks; the build refuses
    it unless the firmware declares `licence = "GPL-3.0-only"`, and fm1-base ships `examples/link` that way
    (its README and LICENSING.md, November 2026). So fm1-base itself builds GPL firmware on its own stacks. An
    explicit statement for a GPL firmware without Link is still missing.
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
    m-bootloader 3.0 itself has not been tested on any SMK-37 yet (its v3.0 notes).
41. **SMK37 Open and the calibration sector.** SMK37 Open keeps the FM-1's layout, so its USER region
    `[0x7C000, 0xFF000)` spans the SMK-37's calibration and Bluetooth sectors `0xC0000`–`0xC1FFF`. It doesn't read
    the factory calibration; does its firmware or updater ever write there? ([SMK-37](SMK-37.md)) *How:* dump
    `0xC0000`–`0xC1FFF` before and after an SMK37 Open install and use.

## Display

35. **The backlight off without losing the keys.** PA2 drives both. Can the ST7789's own brightness command
    (`0x51`) darken this panel, given that one experiment with `0x51`/`0x53` gave wrong colours? *How:* a minimal
    test sending only `0x51` after the normal start-up, colours checked.
    *Since:* M-VAVE's own panel set-up sends `51 FF` (Dinghy `docs/NEWER-FM1-SCREEN.md`; [Display](Display.md)), so
    the command is accepted by this panel; dimming with lower values is still untested.
42. **The newer panels at 60 MHz.** Dinghy's notes say the newer panels need the display clock at 12 MHz;
    fm1-base's `lcd_panel` option sends M-VAVE's set-up but keeps 60 MHz ([Display](Display.md)). Do newer panels
    work at 60 MHz once fully set up? *How:* run fm1-base with `lcd_panel = true` on a newer unit.

## Resolved

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
