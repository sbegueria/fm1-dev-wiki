---
title: "Clocks and Power"
parent: Hardware
nav_order: 3
---

# Clocks and Power

## The CPU clock depends on the bootloader

**The FM-1 runs at whatever clock the bootloader left, unless your firmware sets it.** This is the most
important fact on this page.

| Started by | System clock | Bus clocks (HSB / LSB) | Tag |
| --- | --- | --- | --- |
| M-VAVE's stock bootloader, **015** boot area (V15) | **360 MHz** (PLL1 540 MHz × 2/3) | 180 / 60 MHz | [HW] |
| M-VAVE's stock bootloader, **016** boot area (V16) | **320 MHz** (the 480 MHz main PLL; PLL1 left off) | 160 / 53.3 MHz | [HW] |
| [m-bootloader](https://github.com/jvitkauskas/m-bootloader) 1.2 | **320 MHz**, PLL1 off (felucca-bsp's reading) | 160 / 53 MHz | [HW], disputed below |
| m-bootloader 2.3 | **360 MHz** (as 015) | 180 / 60 MHz | [HW] |
| JieLi SDK tables | "up to 320 MHz" (SDK's stated maximum) | | [SDK] |

The 015, 016 and m-bootloader 2.3 rows come from a boot probe: a program installed as the application, which
records the clock registers it finds at entry. It ran on one FM-1 (flash `0x856014`) on 2026-11-14 and
2026-11-21 (m-bootloader [`docs/STOCK-V15-V16.md`](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md),
[121cfbe](https://github.com/jvitkauskas/m-bootloader/commit/121cfbe)). Under 016 the flash controller's
chip-select-high field is 8, not 9, to suit HSB 160 MHz. The 016 bootloader is the file of JieLi's AC79 SDK 1.2.13;
015's is SDK 1.1.9 (same source).

Under a 320 MHz start, "every firmware was 11 % slower", and every divider set for a 60 MHz peripheral clock
(the LCD's SPI, the UARTs) ran slow too (felucca-bsp `chip/fm1_clock.h`, measured with fm1-x0x's debug build,
2026-10-23). A firmware tuned near its CPU limit will drop out under such a bootloader and its author may never
know why.

**A disagreement on m-bootloader 1.2.** felucca-bsp read 320 MHz with PLL1 off "under m-bootloader 1.2"
(2026-10-23). m-bootloader's own sources say every version sets 360 MHz: its SPEC §3.2 ("the bootloader always
establishes the 360 MHz configuration"), the 1.3 release notes (2026-11-04: "still runs at 360 MHz as before"), and
1.2's source, which starts PLL1 (`src/platform.c` `hal_clock_a` at tag v1.2). The registers felucca-bsp read are the
ones the 016 bootloader leaves. Why the two readings differ is not known **[?]**. Either way, current
m-bootloader (2.3 measured, 3.0 by its SPEC) hands the application 360 MHz.

**What to do:** at start-up, before the audio and the timers, check the clock and put it back to the stock
360 MHz if it differs. felucca-bsp's `chip/fm1_clock.h` (MPL-2.0) does this with m-bootloader's own sequence,
from RAM, with interrupts off. fm1-base calls it at every start ("whichever bootloader started us").

### V16's boot area starts the app at 320 MHz

**[HW]** M-UPGRADE writes the 016 boot area when it installs V16 (m-bootloader
[1.3 release notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v1.3), 2026-11-04; the
[3.0 notes](https://github.com/jvitkauskas/m-bootloader/releases/tag/v3.0) list "M-VAVE's own update from V15 to
V16 through M-UPGRADE, which put the stock bootloader back" as tested). Its `isd_config.ini` says
`SYS_CLK = 320,000,000`, and the boot probe above saw that bootloader hand the app 320 MHz.

So a unit that has had V16 installed starts every later firmware at 320 MHz, unless that firmware resets the clock
or an install puts another boot area back. Community update loaders write only the app area, so they leave the 016
boot area in place (fm1-base [#14](https://github.com/charlesvestal/fm1-base/issues/14): "the loader writes only
0x4000..0x92FFF"). **Not known:** the clock V16's own firmware sets for itself. Neither probe sees it.
(kmplngj's boot-area tool had already read the 320 MHz key, `docs/boot-area-survey.md`, 2026-10.)

**[RE]** M-VAVE's FM1 Open kit ships the same choice: its `sdk/tools/pack/boot-config.bin` (at
[4c7496d](https://github.com/m-vave/fm1_open/tree/4c7496d)) says `SYS_CLK` 320 MHz, `VDDIO` 4, `DVDD` 11,
`DCDC14` 3, as V16's does. SMK37 Open's says 360 MHz, `DVDD` 15, `DCDC14` 7. Both kits carry byte-identical
`uboot.boot` and `ota.bin`. Whether an FM1 Open install writes this boot area on a unit has not been checked.

### The registers

**[HW]** Read off FM-1s under M-VAVE's 015 bootloader (felucca-bsp `chip/fm1_clock.h`; the same three words were read
independently on another unit by DUB-1's Setup page in October 2026, and by m-bootloader's boot probe in November 2026):

| Register | Address | Stock value | Meaning |
| --- | --- | --- | --- |
| System clock source | `0x10014` [7:0] | `6` | 6 = PLL1 × 2/3 (the 016 bootloader leaves `2`: the 480 MHz main PLL, 320 MHz) |
| PLL1 control 0 | `0x119A8` | `0x0940022B` | enable/run, reference divider (`0` under 016) |
| PLL1 control 1 | `0x119AC` | `0x0750310C` | multiplier: 24 MHz / 12 × 270 = 540 MHz (`0` under 016) |
| Main PLL | `0x119A0` / `0x119A4` | `0x45400203` / `0x3F503026` | the same under 015, 016 and m-bootloader 2.3 |
| HSB / LSB dividers | `0x10008` | `0x00010200` | HSB /2 = 180 MHz, LSB /3 = 60 MHz |
| Clock mux | `0x1000C` | `0x1C1` | bit 8: the system clock from the PLL mux |

### Measuring the clock

- **[HW]** A one-instruction loop runs 51.43 times per µs at 360 MHz and 45.71 at 320 MHz (felucca-bsp).
  Timing a fixed loop against TIMER4 (24 MHz, from the crystal) gives the clock without trusting any register.
- **[HW]** The core's cycle counter (`C0_TL_CKCNT`, `0x01EEE218`) counts **only when bits 0–2 of `DBG_CON`
  (`0x01EEE344`) are set**. They are 0 at boot, which is why it did not move on DUB-1's unit (October 2026).
  `DBG_CON` is written behind the debug unlock (`DBG_WR_EN`). Measured on one FM-1 (stock bootloader, V15 board):
  360,031 counts per ms with the bits set, 0 before and after (hellsend,
  [felucca-bsp#18](https://github.com/hugelton/felucca-bsp/issues/18), 2026-10-30). The SDK's `cpu_effic_init`
  sets the same bits. felucca-bsp's `fm1_perf_stalls_enable(1)` sets them
  ([dfa1648](https://github.com/hugelton/felucca-bsp/commit/dfa1648), 2026-11-21). **Not known:** whether they are
  safe to leave on while an app runs (they may also arm fault exceptions). felucca-bsp sets them only when asked.

### Going faster

**[CODE]** felucca-bsp's `fm1_clock_pll1(mul)` sets another PLL1 multiplier by the stock steps: PLL1 =
24 / 12 × mul MHz, the system clock 2/3 of it. `mul` 297 gives **396 MHz**, "JieLi's top, which its SDK marks not
for mass production" (the file's comment). HSB and LSB follow (198 / 66 MHz), so the LCD's SPI and the UARTs run 10%
faster. The audio PLL is not touched. A soft reset keeps the PLL, so a firmware that raises it must call
`fm1_clock_restore()` before its resets and before the update loader
([d61bfcb](https://github.com/hugelton/felucca-bsp/commit/d61bfcb), Charles Vestal, 2026-11-10).

**[HW]** fm1-base offers it as `cpu_clock = 396` in `fm1.toml`, off by default. "Tried on one FM-1: no dropouts on
a heavy synth that needed it, installs both ways clean" ([a6630fe](https://github.com/charlesvestal/fm1-base/commit/a6630fe),
2026-11-10). Two hung starts fall back to 360 MHz. The build warns on every build that uses it. One trap: with the
restore called while interrupts were on, an update or download-mode request from a 396 MHz firmware hung and the
watchdog reset the unit into the same firmware (one FM-1 with flash `0xB36014`). The fix turns interrupts off
around every restore (tokumine, [fm1-base#87](https://github.com/charlesvestal/fm1-base/pull/87), merged
2026-11-22).

felucca-bsp's README and `docs/EXPERIMENTAL.md` still call a higher clock a "concept" (at
[fdd709f](https://github.com/hugelton/felucca-bsp/tree/fdd709f), 2026-11-26). Raising the clock moves the flash
timing, SPI and UART dividers and USB's 48 MHz with it. The thread's advice still holds: use M-VAVE's settings
(360 MHz, its voltages) unless you have a reason not to, and try anything faster on several units.

**[?]** One user's Skiff unit crashed with both cores on at 360 MHz under m-bootloader. It ran fine at 360 MHz
under the stock 015 and 016 bootloaders, and fine at 320 MHz (Skiff's own setting) under m-bootloader. Another
user's units were not affected (Audio-C, [m-flasher#55](https://github.com/jvitkauskas/m-flasher/issues/55),
2026-11-14 and 15). The cause is not known: m-bootloader leaves the same core voltages as the stock bootloaders
(STOCK-V15-V16.md).

## Time bases

| Timer | Use in Felucca-family firmware | Tag |
| --- | --- | --- |
| TIMER4 (`0x10800`) | Free-running from the 24 MHz crystal, period `0xFFFFFFFF`: the time base. Wraps every 178.9 s; use unsigned differences. Not used by the ROM or the first-stage loader. | [CODE] |
| TIMER5 (`0x10900`) | System tick: key-matrix scan, USB poll (2 kHz), ms counter. IRQ 63. 10 kHz by default. Since 2026-11-10 felucca-bsp also takes 5, 2 or 1 kHz (`FM1_INPUT_HZ`; period 6 MHz / rate), and fm1-base's `input_hz`. fm1-base estimates the scan at ~7.5% of core 0 at 10 kHz. | [CODE] |
| ALNK0 (I2S) | The audio interrupt: one per half buffer. See [Audio Output](Audio-Output.md). | [CODE] |

TIMER4 runs from the crystal, so it keeps true time whatever the CPU clock is. That makes it the right reference
for a CPU-load meter.

## Supply voltages

**Two sources disagree on what M-VAVE's firmware sets.** Both are shown here; see [Open Questions](Open-Questions.md).

| Rail | Register (P33) | Steps | Bootloader leaves [HW] | V15's code sets, read statically [RE] | Read while V15 or V16 ran, by a probe [HW] |
| --- | --- | --- | --- | --- | --- |
| VDC14 ("domain") | `P3_ANA_CON6` (`0x06`) [2:0] | 1.25 V + 50 mV a step | 3 (1.40 V) | 7 (1.60 V) | 3 (1.40 V) |
| SYSVDD ("core", core and SRAM) | `P3_ANA_CON9` (`0x09`) [3:0] | 0.93 V + 30 mV a step | 11 (1.26 V) | 15 (1.38 V) | 9 (1.20 V) |
| VDDIO (I/O) | `P3_ANA_CON5` (`0x05`) | | 015: not written; 016: 4 (3.2 V) | 4 (3.2 V) | 7 (3.5 V) |

How each was taken:
- **[RE]** fc79 read V15's start-up code: its `app_core` task runs the SDK's `__hw_power_init` with
  `P33_CON_SET(6,0,3,7)` (VDC14 7) and `P33_CON_SET(9,0,4,15)` (SYSVDD 15), and VDDIOM 4. "A second independent
  pass confirmed each of them" ([fm1-x0x#21](https://github.com/charlesvestal/fm1-x0x/issues/21), 2026-10-14).
  felucca-bsp `chip/fm1_sys.h` and fm1-base set these levels from that reading. X0X 1.0.9 runs at them.
- **[HW]** m-bootloader's voltage probe is a RAM app that reads the P33 registers. It runs after the ROM's
  download mode, entered from the running firmware by a soft reset. P33 settings survive that reset, so the probe
  sees what the firmware left. One FM-1 (`0x856014`), 2026-11-21: under V15 and V16 firmware, with any bootloader,
  I/O code 7, core 9, domain 3 ([0403adc](https://github.com/jvitkauskas/m-bootloader/commit/0403adc);
  STOCK-V15-V16.md, "The voltages while firmware runs"). The same document says the chip powers up with the I/O rail
  at 3.5 V, and that 016's 3.2 V "lasts only from start-up until its firmware starts".

The two may describe different moments (static code versus registers after a reset into download mode), or one may
be wrong. Nobody has yet read the registers from inside a running V15.

**[HW]** "At the boot loader's, under load (both cores, or WiFi in fm1-radio), some units read flash and SRAM wrong"
(felucca-bsp `chip/fm1_sys.h`, from fm1-x0x#21). felucca-bsp raises
them a step every 100 µs, VDC14 first, because SYSVDD is regulated from it.

**[RE]** By fc79's reading, M-VAVE's firmware also moves the 1.3 V supply from its linear regulator onto the DC-DC,
tunes the DC-DC, sets VDDIO to 3.2 V and puts the WiFi regulator on automatic. felucca-bsp's `fm1_power_stock()`
reproduces the sequence. **[HW]** The always-on P33 domain keeps these across a reset, so a unit's state depends on
what ran last. On one FM-1, WiFi heard far less without them (felucca-bsp, 2026-10-23).

**I/O voltage and the bootloaders.** **[CODE]** m-bootloader 2.0 to 2.4 set I/O code 4 (3.2 V) on the FM-1, copying
016's `VDDIO` key ([5a2a57e](https://github.com/jvitkauskas/m-bootloader/commit/5a2a57e), 2026-11-04). The write was
removed in [551f76e](https://github.com/jvitkauskas/m-bootloader/commit/551f76e) ([PR #38](https://github.com/jvitkauskas/m-bootloader/pull/38), 2026-11-21), so 3.0 leaves the
I/O voltage as it finds it. **[?]** One user heard louder USB noise after installing the bootloader
([m-flasher#70](https://github.com/jvitkauskas/m-flasher/issues/70), 2026-11-20); no link to the voltage has been
shown.

The cost of the higher voltages is battery life (the thread's author; not measured here).

### The P33 bridge

Power, reset, the watchdog and the analog settings live in the always-on "P33" domain, reached through a serial
bridge at `0x13E00`. Access it with interrupts off so nothing else uses the bridge at the same time. **[CODE]**

## Battery

The battery's divider is on ADC channel 3 (PB1). **Never drive or pull PB1 low:** held low for 8 s it resets the
chip (felucca-bsp `boards/fm1/fm1_adc.h`, from `isd_config`). **[CODE]** fm1-base has `plat_battery` for charge bars.
FM-1-RE decodes the same stock setting, `RESET=PB01_08_0`: PB1, 8 s, active low
([`TODO_aug2.md`](https://github.com/AL-255/FM-1-RE/blob/main/TODO_aug2.md), "PB01 is a reset input"). **[RE]**

**How stock V13 reads it.** **[RE]** It sums ADC channel 3 over 256 ticks of its input timer and turns the average into
four levels, with cuts at raw values of about 520, 531, 550 and 591. Around each reading it drives GPIO 149
high, then low, and reads bit 1 of `0x51004`, which FM-1-RE takes for a charger or USB-present flag; those two roles are
its guess (FM-1-RE [`io/07-input.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/07-input.md) §4.2). **[?]**
What those raw values mean in volts is still open ([Open Questions](Open-Questions.md)).

## Sources

- [felucca-bsp](https://github.com/hugelton/felucca-bsp) `chip/fm1_clock.h` (MPL-2.0, © Charles Vestal),
  `chip/fm1_sys.h`, `chip/fm1_perf.h`, `chip/fm1_time.h`, `chip/fm1_timer.h`, `boards/fm1/fm1_adc.h`,
  `docs/EXPERIMENTAL.md` (at fdd709f); [issue #18](https://github.com/hugelton/felucca-bsp/issues/18).
  Felucca has taken these files from felucca-bsp (its `firmware/bsp` submodule) since 1.7.0; its old `hal/` copies
  are gone.
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §3.2;
  [docs/STOCK-V15-V16.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/STOCK-V15-V16.md) (121cfbe);
  [release notes](https://github.com/jvitkauskas/m-bootloader/releases).
- [fm1-base](https://github.com/charlesvestal/fm1-base) `tools/build.py` (`cpu_clock`, `input_hz`), `docs/SECOND-CORE.md`;
  [PR #87](https://github.com/charlesvestal/fm1-base/pull/87); [issue #14](https://github.com/charlesvestal/fm1-base/issues/14).
- [FM1 Open](https://github.com/m-vave/fm1_open) and [SMK37 Open](https://github.com/m-vave/smk37_open) `sdk/tools/pack/boot-config.bin`.
- JieLi AC79 SDK `README.md`.
- [fm1-x0x#21](https://github.com/charlesvestal/fm1-x0x/issues/21) (voltages; fc79's reading of V15).
- [m-flasher#55](https://github.com/jvitkauskas/m-flasher/issues/55), [#70](https://github.com/jvitkauskas/m-flasher/issues/70).
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) (WTFPL; stock V13) `docs/io/07-input.md` §4.2, `TODO_aug2.md`.
- [kmplngj/Felucca `docs/boot-area-survey.md`](https://github.com/kmplngj/Felucca/blob/main/docs/boot-area-survey.md) (GPL-3.0-only).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
