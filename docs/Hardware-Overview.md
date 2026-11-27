---
title: "Hardware Overview"
parent: Hardware
nav_order: 1
---

# Hardware Overview

What is inside an FM-1, as far as the community knows. Each line has its tag (see [Home](Home.md)).

## The chip

| | | Tag |
| --- | --- | --- |
| SoC | JieLi **AC791N**, WL82 family ("AC79" in JieLi's SDK). The r/MVaveFM1 thread names the part **AC79118B**; the [FM-1 resource hub](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) reads **AC7911B**, QFN-48, marking `JL C156211-11A-B`, from board photos. FM-1-RE reads the marking on its unit as `C156211-11B8`, **LQFP48** ([`docs/01-firmware-analysis.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/01-firmware-analysis.md), "SoC / CPU"). None is checked here **[?]** ([Open Questions](Open-Questions.md)). Strings like `JL-BR22` in the stock firmware come from linked libraries and do not identify the chip. | [RE] [SDK] |
| CPU | JieLi **pi32v2**, 32-bit, little endian, 16-bit instruction words. Toolchain target `-mcpu=r3`. | [SDK] [CODE] |
| Cores | **Two.** The SDK defines `CPU_CORE_NUM 2`. Several firmwares run work on the second core. See [Second Core](Second-Core.md). | [SDK] [HW] |
| FPU | **Single-precision** hardware floating point, enabled with `-mfprev1`. Doubles are done in software (slow). M-VAVE's V13 links software helpers for doubles only, none for single floats ([Performance](Performance.md)). | [SDK] [CODE] [RE] |
| Caches | I-cache and D-cache (SDK). The data cache has **8 ways of 4 KiB**; 1 to 7 of them can be used as plain RAM at `0x01F28000` + way × 4 KiB, as M-VAVE's V15 does with 7. On one FM-1 the ways passed a memory test and read as fast as main RAM; with 7, flash data reads over 16 KiB were about 57× slower (Optimist, 2026-10-18; felucca-bsp `chip/fm1_cache_ram.h`; hdavid, [optimist#295](https://github.com/w0ts/optimist/issues/295)). No long run is on record. I-cache size and layout: see [Performance](Performance.md) and [Open Questions](Open-Questions.md). | [SDK] [HW] [?] |
| Clock | **360 MHz** under M-VAVE's 015 (V15) bootloader (measured). M-VAVE's **016** (V16) bootloader hands the firmware **320 MHz** (LSB 53.3 MHz), measured with a boot probe on one FM-1 (m-bootloader [`docs/STOCK-V15-V16.md`](https://github.com/jvitkauskas/m-bootloader/blob/121cfbe/docs/STOCK-V15-V16.md), 2026-11-14 and 21). m-bootloader: its SPEC sets 360 MHz, and the same probe read 360 MHz under 2.3; [Clocks and Power](Clocks-and-Power.md) reports 320 MHz under 1.2. The SDK calls 320 MHz its maximum. | [HW] |
| SRAM | The SDK says **578 KB** of on-chip SRAM in total. Firmwares use the window `0x01C00000`–`0x01C80000` (512 KB). There is **no external RAM** on the FM-1. | [SDK] [CODE] |
| Accelerators | Hardware FFT/IFFT and matrix maths, AES-128/256, SHA, random numbers, CRC16 (SDK). Only CRC16 is known to be used by community firmware. | [SDK] |
| Radio | WiFi 802.11b/g/n and Bluetooth 5.0 (dual mode). On **JieLi's closed libraries** a firmware has BLE or WiFi, not both: together they don't fit in RAM. **fm1-base's own stacks** (no JieLi code) run BLE and WiFi together, since 2026-11-25: a WPA2-PSK station with ARP, DHCP, IGMP and UDP only (no TCP, no DNS), an access-point mode, and opt-in Ableton Link (GPL-3.0-only) ([fm1-base README](https://github.com/charlesvestal/fm1-base/blob/3039770/README.md), merge [c734409](https://github.com/charlesvestal/fm1-base/commit/c734409)). fm1-base's notes: the WiFi MAC has Ralink RT3070's register layout at `0x30000` (JieLi's wl30 core), the BLE baseband registers are at `0x28000` (`docs/wifi/README.md`, `docs/bt/BLE-HW-FACTS.md`). See [fm1-base as a Base](FM1-Base.md). | [SDK] [HW] [RE] |

## The board

| | | Tag |
| --- | --- | --- |
| Flash | 1 MiB SPI NOR, executed in place (XIP) through the SFC. Two JEDEC IDs seen: `856014` (Puya, the earlier units) and `B36014` (**UC25HQ80**, on some units sold since about autumn 2026; same size, same chip key). Not every new unit has it: one received in late November 2026 still carried the Puya `856014`. Chip key `0x980F` on known units. A firmware that checks the ID must accept both: DUB-1 1.4.3 accepted only `856014`, so on a UC25HQ80 nothing saved ("NOT SAVED · FLASH ERROR") and it refused installs over itself; fixed in 1.4.4 (fm1-dub#14, PR #15), `fm1_rescue` too. M-Flasher's device info shows the ID. | [HW] |
| Flash part | JEDEC manufacturer `0x85` is Puya; the hub names the **Puya P25Q80H** (8 Mbit) and says the die is **inside the SoC's package** (SiP), wired to the SFC on PD0, PD1, PD4, PD5, PD6. Other notes speak of a SOIC-8 part on the board ([Flash Storage](Flash-Storage.md)). | [?] |
| Screen | 240×240 colour TFT, ST7789-class, RGB565, on SPI1. Backlight on PA2 (active low). See [Display](Display.md). | [HW] |
| Keys | 27 note keys (F3 to G5) and 14 buttons in one diode matrix behind two 74HC595 shift registers. | [HW] |
| Encoders | 7 quadrature encoders in the same matrix. | [HW] |
| Pots | MASTER on the ADC (PB6). The battery's divider on PB1. | [HW] |
| LEDs | Key and button LEDs, multiplexed with the matrix columns. | [HW] |
| Audio | Stereo out through the chip's ALNK0 (I2S) block: PC0 MCLK, PC1 BCLK, PC2 LRCK, PC6 data (felucca-bsp and M-VAVE's FM1 Open). About **44,117.6 Hz**, from the 24 MHz crystal. Whether the converter is inside the chip or a separate codec is disputed: see [Audio Output](Audio-Output.md). | [CODE] [?] |
| Headphone jack | A switch on PB5: low with headphones in. | [HW] |
| MIDI | USB-MIDI device (USB 1.1 full speed); TRS MIDI in on UART1 (PH8). | [CODE] |
| USB | One USB port: device mode normally (stock firmware enumerates as `4C4A:C755`); the chip's mask-ROM download mode for recovery. USB host is possible with external power. | [HW] |
| Power | Battery with a charger; the core and SRAM supply (SYSVDD) and VDC14 set by firmware. See [Clocks and Power](Clocks-and-Power.md). | [HW] |
| Debug | **No debug header** on the board. Recovery is over USB only. | [HW] |
| Card slot | None. A mod ([m-card](https://github.com/jvitkauskas/m-card)) puts a microSD card on the screen's SPI1: see [Pinout](Pinout.md). | [HW] [CODE] |

Pin details: [Pinout](Pinout.md). Addresses: [Memory Map](Memory-Map.md).

## The SMK-37

The SMK-37 Pro / Elite uses the same chip family with more keys, pads, faders and a different panel wiring.
See [SMK-37](SMK-37.md).

## What runs on it from the factory

M-VAVE's stock firmware is a DX7-compatible six-operator FM synth. **V16** came out on 2026-10-21 (eight parts,
20-note polyphony, 64-step patterns, per awesome-fm-1 and the hub); before it, **V15**. M-VAVE replaces the file at
its download address with each release, so that address now gives V16. Most custom-firmware installers and
recovery tools were written against V15 ([Recovery](Recovery.md)). **[CODE]** Reverse
engineering of V13 found an msfa/Dexed-derived engine, a FreeRTOS-derived SMP kernel, JieLi's UI, USB, BLE and
storage stacks ([FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md); its code
addresses need `0x120` added, see [Memory Map](Memory-Map.md#the-stock-app-runs-at-0x02000120)). Its notes find the
stock synth on the same ALNK0 output as the open firmwares ([Audio Output](Audio-Output.md)).
None of that is needed by a custom firmware: the open firmwares replace the whole application.

## Sources

- JieLi AC79 SDK: `README.md` (chip summary), `include_lib/driver/cpu/wl82/asm/cpu.h` (`CPU_CORE_NUM`).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp): `chip/fm1_clock.h`, `chip/fm1_sys.h`, `boards/fm1/`.
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §2, §8.
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) (WTFPL; stock V13): `docs/architecture.md`, `docs/01-firmware-analysis.md`.
- Felucca-derived HAL headers (`hal/fm1_input.h`, `fm1_lcd_hw.h`, `fm1_adc.h`, `fm1_audio.h`, `fm1_uart.h`;
  GPL-3.0-only, © Leo Kuroshita), as in DUB-1 and other Felucca forks. Since Felucca 1.7.0 they are felucca-bsp's
  `boards/fm1/` and `chip/` files (MPL-2.0); felucca-bsp `chip/fm1_cache_ram.h`, `boards/fm1/fm1_jack.h`,
  `boards/fm1/fm1_audio.h` and `docs/EXPERIMENTAL.md` at `fdd709f`.
- [m-bootloader `docs/STOCK-V15-V16.md`](https://github.com/jvitkauskas/m-bootloader/blob/121cfbe/docs/STOCK-V15-V16.md) (the clock each bootloader hands over).
- [fm1-base](https://github.com/charlesvestal/fm1-base) at `3039770`: `README.md`, `docs/wifi/README.md`, `docs/bt/BLE-HW-FACTS.md` (MPL-2.0).
- [M-VAVE FM1 Open](https://github.com/m-vave/fm1_open) `app/board/fm1_audio.h` at `4c7496d`.
- `fm1_rescue.py` in Felucca-family trees (flash ID and chip key checks).
- [fm1-dub#14](https://github.com/sbegueria/fm1-dub/issues/14), [#15](https://github.com/sbegueria/fm1-dub/pull/15): the UC25HQ80 (`B36014`) on new units, M-Flasher's device info from two owners (2026-11).
- [FM-1 resource hub REVERSE_ENGINEERING.md](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) §1.1, §1.2
  (part marking, package, flash part, from board photos and JieLi's AC7911B datasheet; no licence declared, so
  summarised here, not copied).
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) firmware skill §10 (USB ID `4C4A:C755` seen on a stock unit,
  2026-10-22).
- [awesome-fm-1](https://github.com/cicloid/awesome-fm-1) "Official" (V16, October 2026).
- [optimist#295](https://github.com/w0ts/optimist/issues/295): hdavid's answers to this wiki (2026-11-27); Optimist [`docs/INSTALL-PATHS.md`](https://github.com/w0ts/optimist/blob/bfa17d5e/docs/INSTALL-PATHS.md) (2026-11-26).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
