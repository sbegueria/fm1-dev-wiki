# Clocks and Power

## The CPU clock depends on the bootloader

**The FM-1 runs at whatever clock the bootloader left, unless your firmware sets it.** This is the most
important fact on this page.

| Started by | System clock | Peripheral clock (LSB) | Tag |
| --- | --- | --- | --- |
| M-VAVE's stock bootloader | **360 MHz** (PLL1 540 MHz × 2/3) | 60 MHz | [HW] |
| [m-bootloader](https://github.com/jvitkauskas/m-bootloader) 1.2 | **320 MHz**, PLL1 off | 53 MHz | [HW] |
| JieLi SDK tables | "up to 320 MHz" (SDK's stated maximum) | | [SDK] |

Under m-bootloader 1.2, "every firmware was 11 % slower", and every divider set for a 60 MHz peripheral clock
(the LCD's SPI, the UARTs) ran slow too (felucca-bsp `chip/fm1_clock.h`, measured with fm1-x0x's debug build,
2026-10-23). A firmware tuned near its CPU limit will drop out on m-bootloader and its author may never know why.

**What to do:** at start-up, before the audio and the timers, check the clock and put it back to the stock
360 MHz if it differs. felucca-bsp's `chip/fm1_clock.h` (MPL-2.0) does this with m-bootloader's own sequence,
from RAM, with interrupts off. fm1-base calls it at every start ("whichever bootloader started us").

### The registers

**[HW]** Read off FM-1s under M-VAVE's bootloader (felucca-bsp `chip/fm1_clock.h`; the same three words were read
independently on another unit by DUB-1's Setup page in October 2026):

| Register | Address | Stock value | Meaning |
| --- | --- | --- | --- |
| System clock source | `0x10014` [7:0] | `6` | 6 = PLL1 × 2/3 (m-bootloader leaves `2`) |
| PLL1 control 0 | `0x119A8` | `0x0940022B` | enable/run, reference divider |
| PLL1 control 1 | `0x119AC` | `0x0750310C` | multiplier: 24 MHz / 12 × 270 = 540 MHz |
| HSB / LSB dividers | `0x10008` | `0x00010200` | HSB /2 = 180 MHz, LSB /3 = 60 MHz |
| Clock mux | `0x1000C` | `0x1C1` | bit 8: the system clock from the PLL mux |

### Measuring the clock

- **[HW]** A one-instruction loop runs 51.43 times per µs at 360 MHz and 45.71 at 320 MHz (felucca-bsp).
  Timing a fixed loop against TIMER4 (24 MHz, from the crystal) gives the clock without trusting any register.
- **[HW] [?]** The core's cycle counter (`C0_TL_CKCNT`, `0x01EEE218`) did **not** move on at least one FM-1 under
  the stock bootloader (DUB-1, October 2026), so a measurement based on it returned nothing. fm1-base offers
  cycle-counter calls for profiling. Whether the counter needs enabling first is in [[Open Questions]].

### Going faster

Above 360 MHz is a **concept** in felucca-bsp, not something that runs. Raising the clock moves the flash
timing, SPI and UART dividers, USB's 48 MHz and the audio clock with it. The thread's advice: use M-VAVE's
settings (360 MHz, its voltages) and assume they're fine until you have a reason to doubt them.

## Time bases

| Timer | Use in Felucca-family firmware | Tag |
| --- | --- | --- |
| TIMER4 (`0x10800`) | Free-running from the 24 MHz crystal, period `0xFFFFFFFF`: the time base. Wraps every 178.9 s; use unsigned differences. Not used by the ROM or the first-stage loader. | [CODE] |
| TIMER5 (`0x10900`) | 10 kHz system tick: key-matrix scan, USB poll (2 kHz), ms counter. IRQ 63. | [CODE] |
| ALNK0 (I2S) | The audio interrupt: one per half buffer. See [[Audio Output]]. | [CODE] |

TIMER4 runs from the crystal, so it keeps true time whatever the CPU clock is. That makes it the right reference
for a CPU-load meter.

## Supply voltages

**[HW]** The bootloaders leave the core rails lower than M-VAVE's firmware sets them:

| Rail | Register (P33) | Steps | Bootloader leaves | M-VAVE V15 sets |
| --- | --- | --- | --- | --- |
| VDC14 | `P3_ANA_CON6` [2:0] | 1.25 V + 50 mV a step | 3 (1.40 V) | 7 (1.60 V) |
| SYSVDD (core and SRAM) | `P3_ANA_CON9` [3:0] | 0.93 V + 30 mV a step | 11 (1.26 V) | 15 (1.38 V) |

"At the boot loader's, under load (both cores, or WiFi in fm1-radio), some units read flash and SRAM wrong"
(felucca-bsp `chip/fm1_sys.h`, from fm1-x0x#21). felucca-bsp raises them a step every 100 µs, VDC14 first,
because SYSVDD is regulated from it.

**[HW]** M-VAVE's firmware also moves the 1.3 V supply from its linear regulator onto the DC-DC, tunes the DC-DC,
sets VDDIO to 3.2 V and puts the WiFi regulator on automatic. The always-on P33 domain keeps these across a
reset, so **a unit that last ran V15 has them, and one that never did lacks them.** On one FM-1, WiFi heard far
less without them. felucca-bsp's `fm1_power_stock()` reproduces the sequence.

The cost of the higher voltages is battery life (the thread's author; not measured here).

### The P33 bridge

Power, reset, the watchdog and the analog settings live in the always-on "P33" domain, reached through a serial
bridge at `0x13E00`. Access it with interrupts off so nothing else uses the bridge at the same time. **[CODE]**

## Battery

The battery's divider is on ADC channel 3 (PB1). **Never drive or pull PB1 low:** held low for 8 s it resets the
chip (Felucca's `hal/fm1_adc.h`, from `isd_config`). **[CODE]** fm1-base has `plat_battery` for charge bars.

## Sources

- [felucca-bsp](https://github.com/hugelton/felucca-bsp) `chip/fm1_clock.h` (MPL-2.0, © Charles Vestal),
  `chip/fm1_sys.h`, `docs/EXPERIMENTAL.md`.
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §3.2.
- JieLi AC79 SDK `README.md`.
- Felucca's `hal/fm1_time.h`, `fm1_timer.h`, `fm1_adc.h` (GPL-3.0-only).
- [fm1-x0x#21](https://github.com/charlesvestal/fm1-x0x/issues/21) (voltages, cited by felucca-bsp).
