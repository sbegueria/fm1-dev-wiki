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
   *How:* a board photo with the part number; a datasheet; probe the lines.
2. **The true sample rate.** Open firmwares assume 44,100 Hz; FM-1-RE reports the stock audio server at 44,118 Hz.
   Is the rate the same under 360 and 320 MHz? *How:* record a long sine from the FM-1 and measure its frequency
   against the computer's clock, under each bootloader.
3. **The instruction cache.** Size, ways, line size, replacement policy, and whether lines can be locked. DUB-1's
   notes say 32 KB, 8 ways × 4 KB, 32-byte lines; the SDK has `CACHE_LINE_COUNT 32` and `ICACHE_WAY`/`DCACHE_WAY`
   registers. *How:* time a loop over code of growing size from XIP; read the SDK's `cache.h` users.
   *Known so far:* running DUB-1's 55 KB audio path from SRAM instead of XIP cut its peaks by up to 35% and stopped
   its dropouts ([Performance](Performance.md)), so misses are costly; the size itself is still unmeasured.
4. **The D-cache and DMA.** Must DMA buffers be flushed or kept out of the D-cache? Felucca-family firmware keeps DMA
   buffers in plain RAM and has worked, but no one has written down why.
5. **The cycle counter.** `C0_TL_CKCNT` (`0x01EEE218`) did not move on at least one FM-1 under the stock
   bootloader (DUB-1, October 2026). fm1-base offers cycle-counter calls (`plat_cycles*`). Does it need enabling
   (e.g. a bit in `DBG_CON`), or does it run only on some units or bootloaders?
6. **The stall counters** (`C0_IF_UACNT`, `C0_RD_UACNT`, `C0_WR_UACNT`): the SDK's `cpu_effic_calc` reads them.
   Do they count on the FM-1, and do they show I-cache misses?
7. **SRAM in total.** The SDK says 578 KB; firmwares use 512 KB at `0x01C00000`. Where is the rest, and can a
   firmware use it?
8. **Subnormal floats.** Does the FPU slow down on subnormal numbers, or flush them? *How:* time a loop on
   subnormal and normal operands.
9. **Flash timing.** Sector erase and page program times of the stock part (`856014`), and whether XIP reads stall
   the other core during them.
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

## CPU

15. **Is there an FPU?** JieLi's SDK and Felucca, DUB-1 and FM1 Open compile with `-mfprev1` and use `float` on
    hardware. Dinghy's `docs/RULES.md` (the SLOOP line), and mvave-fm1's skills after it, say "there is no FPU and
    no libgcc", so floats fail to link. The likely reading is that Dinghy's build links no float support, not that
    the chip lacks an FPU, but no one has written that down. *How:* build a float test with `-mfprev1`, run it on
    a unit and time it against the same code built soft-float.

16. **The 64-bit multiply erratum.** Which exact instruction pairs fail? Is 320 MHz clear of it? Does it ever hit
    float code? (felucca-bsp `docs/CPU.md`.)
17. **The second core's misreads.** Why do some units' core 1 misread shared RAM? Is it the supply, the clock, the
    bus? Is it fixed by M-VAVE's voltages? (melodee#17, fm1-x0x#10.)
18. **Flag gains.** What do FM1 Open's flags (`-flto`, `-pi32v2-enable-simd=true`, `-pi32v2-large-program=true`,
    IPRA, `rep-memop`) gain on real DSP code, and do they change float results?

## Memory map

19. **The XIP offset in the stock image.** Felucca maps flash `0x4120` to `0x02000120`; FM-1-RE writes
    `flash_offset − 0x4120 + 0x02000000` and puts stock vectors at `0x020000A0`. Is the stock app linked
    differently, or is one note off by the 0x120 header?
20. **The stock clock in the app.** FM-1-RE reads a `pll_clock_init` → 240 MHz in V13's start-up; measurements under
    M-VAVE's bootloader, with an open firmware running, give 360 MHz. What does V15's own start-up set, and when?

## Installing

21. **Installs that stall half-way** ("the loader stopped answering" after ~half the requests). A USB drop, or the
    loader's poll falling behind? *How:* log the last request and its timing in the installer; a USB capture.
22. **m-bootloader and full packages.** Which installers keep m-bootloader and which put the stock one back?
    (SLOOP's kept it; M-VAVE's and Baud Girl's restore stock.)
23. **The loader's watchdog.** How long can the update loader go without a request before it gives up, per base?

24. **USB IDs of the update loaders.** The hub reports `4D4A:4155` "ota-FM-1" during an update; mvave-fm1's skill
    puts ORBIT's loader there too. ORBIT's and Felucca's loader sources say `1209:0002` "Felucca Update". Is
    `4D4A:4155` M-VAVE's own loader? *How:* `lsusb` during an M-UPGRADE update and during an ORBIT update.
25. **V16.** Since 2026-10-21 M-VAVE's download address serves V16. What identity does V16 answer (`FM-1_016`?),
    does it answer the soft key `F0 22 24 35 7D F7`, and does it accept the community's packages the way V15 does?
    Where does V16 stage its update loader (V15: `0xE6000`)? *How:* `fm1_install.py --info` and M-Flasher's
    DOWNLOAD MODE on a V16 unit; a flash dump after a V16 install.
26. **The V15 region hash on real units.** One unit reporting `FM-1_015` had a firmware-region hash
    (`0x4000…0x92FFF`) of `8c1d13a5…`, not the official file's `6edf3c37…` (mvave-fm1, 2026-10-22). Was it an
    earlier V15 build, or does stock V15 write inside its own app area? *How:* compare dumps from several V15
    units sector by sector.
27. **Download-mode sessions that end.** On one Linux run, the chip reset back to normal after one tool run in
    download mode (`info`, then `backup`, failed). A timeout in the ROM, or the host re-probing the disk?
    *How:* time how long the ROM stays in download mode idle, on Linux and Windows.
28. **The watchdog's timeout codes.** Dinghy documents `0xA` = 1 s to `0xF` = 32 s for `P3_WDT_CON`; 8 s for
    `0x0D` assumes a doubling per step. *How:* time a reset with no feed.
29. **Can anything run stock firmware off the device?** RemoteFM1's README says no emulator exists for the FM-1's
    CPU, so stock firmware can't run on a computer; fm1-emu and fm1-emulator run community firmware images
    unchanged ([Simulators and Emulators](Simulators-and-Emulators.md)). Do they run stock V15?

## The SMK-37

30. Elite hardware behaviour is inferred from the Pro's (m-bootloader SPEC §13). Who has an Elite to test?

## Resolved

(Move questions here with a link to where the answer now lives.)

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
