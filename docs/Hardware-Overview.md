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
| SoC | JieLi **AC791N**, WL82 family ("AC79" in JieLi's SDK). The r/MVaveFM1 thread names the part **AC79118B**; the [FM-1 resource hub](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) reads **AC7911B**, QFN-48, marking `JL C156211-11A-B`, from board photos. Neither is checked here **[?]** ([Open Questions](Open-Questions.md)). Strings like `JL-BR22` in the stock firmware come from linked libraries and do not identify the chip. | [RE] [SDK] |
| CPU | JieLi **pi32v2**, 32-bit, little endian, 16-bit instruction words. Toolchain target `-mcpu=r3`. | [SDK] [CODE] |
| Cores | **Two.** The SDK defines `CPU_CORE_NUM 2`. Several firmwares run work on the second core. See [Second Core](Second-Core.md). | [SDK] [HW] |
| FPU | **Single-precision** hardware floating point, enabled with `-mfprev1`. Doubles are done in software (slow). | [SDK] [CODE] |
| Caches | I-cache and D-cache (SDK). Size and layout: see [Performance](Performance.md) and [Open Questions](Open-Questions.md). | [SDK] [?] |
| Clock | **360 MHz** under M-VAVE's bootloader (measured); 320 MHz under m-bootloader 1.2 unless the firmware resets it. The SDK calls 320 MHz its maximum. See [Clocks and Power](Clocks-and-Power.md). | [HW] |
| SRAM | The SDK says **578 KB** of on-chip SRAM in total. Firmwares use the window `0x01C00000`–`0x01C80000` (512 KB). There is **no external RAM** on the FM-1. | [SDK] [CODE] |
| Accelerators | Hardware FFT/IFFT and matrix maths, AES-128/256, SHA, random numbers, CRC16 (SDK). Only CRC16 is known to be used by community firmware. | [SDK] |
| Radio | WiFi 802.11b/g/n and Bluetooth 5.0 (dual mode), on JieLi's closed libraries. fm1-base supports BLE MIDI or WiFi (not both at once). | [SDK] [HW] |

## The board

| | | Tag |
| --- | --- | --- |
| Flash | 1 MiB SPI NOR, executed in place (XIP) through the SFC. JEDEC ID `856014`: the only ID `fm1_rescue` accepts, so at least the units it was tested on have it. Chip key `0x980F` on known units. | [HW] |
| Flash part | JEDEC manufacturer `0x85` is Puya; the hub names the **Puya P25Q80H** (8 Mbit) and says the die is **inside the SoC's package** (SiP), wired to the SFC on PD0, PD1, PD4, PD5, PD6. Other notes speak of a SOIC-8 part on the board ([Flash Storage](Flash-Storage.md)). | [?] |
| Screen | 240×240 colour TFT, ST7789-class, RGB565, on SPI1. Backlight on PA2 (active low). See [Display](Display.md). | [HW] |
| Keys | 27 note keys (F3 to G5) and 14 buttons in one diode matrix behind two 74HC595 shift registers. | [HW] |
| Encoders | 7 quadrature encoders in the same matrix. | [HW] |
| Pots | MASTER on the ADC (PB6). The battery's divider on PB1. | [HW] |
| LEDs | Key and button LEDs, multiplexed with the matrix columns. | [HW] |
| Audio | Stereo out through the chip's ALNK0 (I2S) block, with four control lines to the output stage on PC0, PC1, PC2, PC6. Whether the converter is inside the chip or a separate codec is disputed: see [Audio Output](Audio-Output.md). | [CODE] [?] |
| MIDI | USB-MIDI device (USB 1.1 full speed); TRS MIDI in on UART1 (PH8). | [CODE] |
| USB | One USB port: device mode normally (stock firmware enumerates as `4C4A:C755`); the chip's mask-ROM download mode for recovery. USB host is possible with external power. | [HW] |
| Power | Battery with a charger; the core and SRAM supply (SYSVDD) and VDC14 set by firmware. See [Clocks and Power](Clocks-and-Power.md). | [HW] |
| Debug | **No debug header** on the board. Recovery is over USB only. | [HW] |

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
storage stacks ([FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md)).
None of that is needed by a custom firmware: the open firmwares replace the whole application.

## Sources

- JieLi AC79 SDK: `README.md` (chip summary), `include_lib/driver/cpu/wl82/asm/cpu.h` (`CPU_CORE_NUM`).
- [felucca-bsp](https://github.com/hugelton/felucca-bsp): `chip/fm1_clock.h`, `chip/fm1_sys.h`, `boards/fm1/`.
- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §2, §8.
- [FM-1-RE](https://github.com/AL-255/FM-1-RE).
- Felucca-derived HAL headers (`hal/fm1_input.h`, `fm1_lcd_hw.h`, `fm1_adc.h`, `fm1_audio.h`, `fm1_uart.h`;
  GPL-3.0-only, © Leo Kuroshita), as in DUB-1 and other Felucca forks.
- `fm1_rescue.py` in Felucca-family trees (flash ID and chip key checks).
- [FM-1 resource hub REVERSE_ENGINEERING.md](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) §1.1, §1.2
  (part marking, package, flash part, from board photos and JieLi's AC7911B datasheet; no licence declared, so
  summarised here, not copied).
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) firmware skill §10 (USB ID `4C4A:C755` seen on a stock unit,
  2026-10-22).
- [awesome-fm-1](https://github.com/cicloid/awesome-fm-1) "Official" (V16, October 2026).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
