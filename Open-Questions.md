# Open Questions

Things nobody has settled in public, or where sources disagree. If you can answer one, add the measurement or
the source, move the fact to its page with the right tag, and leave a line here saying it was resolved.

Each entry: **the question**, what we know, and how to find out.

## Hardware

1. **The audio converter.** Felucca's HAL says ALNK0 I2S feeds an "external codec"; FM-1-RE says an "internal
   audio DAC". Which part is it, and what do the control lines PC0, PC1, PC2 and PC6 do?
   *How:* a board photo with the part number; a datasheet; probe the lines.
2. **The true sample rate.** Open firmwares assume 44,100 Hz; FM-1-RE reports the stock audio server at 44,118 Hz.
   Is the rate the same under 360 and 320 MHz? *How:* record a long sine from the FM-1 and measure its frequency
   against the computer's clock, under each bootloader.
3. **The instruction cache.** Size, ways, line size, replacement policy, and whether lines can be locked. DUB-1's
   notes say 32 KB, 8 ways × 4 KB, 32-byte lines; the SDK has `CACHE_LINE_COUNT 32` and `ICACHE_WAY`/`DCACHE_WAY`
   registers. *How:* time a loop over code of growing size from XIP; read the SDK's `cache.h` users.
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

## CPU

11. **The 64-bit multiply erratum.** Which exact instruction pairs fail? Is 320 MHz clear of it? Does it ever hit
    float code? (felucca-bsp `docs/CPU.md`.)
12. **The second core's misreads.** Why do some units' core 1 misread shared RAM? Is it the supply, the clock, the
    bus? Is it fixed by M-VAVE's voltages? (melodee#17, fm1-x0x#10.)
13. **Flag gains.** What do FM1 Open's flags (`-flto`, `-pi32v2-enable-simd=true`, `-pi32v2-large-program=true`,
    IPRA, `rep-memop`) gain on real DSP code, and do they change float results?

## Memory map

14. **The XIP offset in the stock image.** Felucca maps flash `0x4120` to `0x02000120`; FM-1-RE writes
    `flash_offset − 0x4120 + 0x02000000` and puts stock vectors at `0x020000A0`. Is the stock app linked
    differently, or is one note off by the 0x120 header?
15. **The stock clock in the app.** FM-1-RE reads a `pll_clock_init` → 240 MHz in V13's start-up; measurements under
    M-VAVE's bootloader, with an open firmware running, give 360 MHz. What does V15's own start-up set, and when?

## Installing

16. **Installs that stall half-way** ("the loader stopped answering" after ~half the requests). A USB drop, or the
    loader's poll falling behind? *How:* log the last request and its timing in the installer; a USB capture.
17. **m-bootloader and full packages.** Which installers keep m-bootloader and which put the stock one back?
    (SLOOP's kept it; M-VAVE's and Baud Girl's restore stock.)
18. **The loader's watchdog.** How long can the update loader go without a request before it gives up, per base?

## The SMK-37

19. Elite hardware behaviour is inferred from the Pro's (m-bootloader SPEC §13). Who has an Elite to test?

## Resolved

(Move questions here with a link to where the answer now lives.)
