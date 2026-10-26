# Hardware Overview

What is inside an FM-1, as far as the community knows. Each line has its tag (see [[Home]]).

## The chip

| | | Tag |
| --- | --- | --- |
| SoC | JieLi **AC791N**, WL82 family ("AC79" in JieLi's SDK). The r/MVaveFM1 thread names the part **AC79118B** (not checked against a chip marking here **[?]**). Strings like `JL-BR22` in the stock firmware come from linked libraries and do not identify the chip. | [RE] [SDK] |
| CPU | JieLi **pi32v2**, 32-bit, little endian, 16-bit instruction words. Toolchain target `-mcpu=r3`. | [SDK] [CODE] |
| Cores | **Two.** The SDK defines `CPU_CORE_NUM 2`. Several firmwares run work on the second core. See [[Second Core]]. | [SDK] [HW] |
| FPU | **Single-precision** hardware floating point, enabled with `-mfprev1`. Doubles are done in software (slow). | [SDK] [CODE] |
| Caches | I-cache and D-cache (SDK). Size and layout: see [[Performance]] and [[Open Questions]]. | [SDK] [?] |
| Clock | **360 MHz** under M-VAVE's bootloader (measured); 320 MHz under m-bootloader 1.2 unless the firmware resets it. The SDK calls 320 MHz its maximum. See [[Clocks and Power]]. | [HW] |
| SRAM | The SDK says **578 KB** of on-chip SRAM in total. Firmwares use the window `0x01C00000`–`0x01C80000` (512 KB). There is **no external RAM** on the FM-1. | [SDK] [CODE] |
| Accelerators | Hardware FFT/IFFT and matrix maths, AES-128/256, SHA, random numbers, CRC16 (SDK). Only CRC16 is known to be used by community firmware. | [SDK] |
| Radio | WiFi 802.11b/g/n and Bluetooth 5.0 (dual mode), on JieLi's closed libraries. fm1-base supports BLE MIDI or WiFi (not both at once). | [SDK] [HW] |

## The board

| | | Tag |
| --- | --- | --- |
| Flash | 1 MiB SPI NOR, executed in place (XIP) through the SFC. JEDEC ID `856014`: the only ID `fm1_rescue` accepts, so at least the units it was tested on have it. Chip key `0x980F` on known units. | [HW] |
| Screen | 240×240 colour TFT, ST7789-class, RGB565, on SPI1. Backlight on PA2 (active low). See [[Display]]. | [HW] |
| Keys | 27 note keys (F3 to G5) and 14 buttons in one diode matrix behind two 74HC595 shift registers. | [HW] |
| Encoders | 7 quadrature encoders in the same matrix. | [HW] |
| Pots | MASTER on the ADC (PB6). The battery's divider on PB1. | [HW] |
| LEDs | Key and button LEDs, multiplexed with the matrix columns. | [HW] |
| Audio | Stereo out through the chip's ALNK0 (I2S) block, with four control lines to the output stage on PC0, PC1, PC2, PC6. Whether the converter is inside the chip or a separate codec is disputed: see [[Audio Output]]. | [CODE] [?] |
| MIDI | USB-MIDI device (USB 1.1 full speed); TRS MIDI in on UART1 (PH8). | [CODE] |
| USB | One USB port: device mode normally; the chip's mask-ROM download mode for recovery. USB host is possible with external power. | [HW] |
| Power | Battery with a charger; the core and SRAM supply (SYSVDD) and VDC14 set by firmware. See [[Clocks and Power]]. | [HW] |
| Debug | **No debug header** on the board. Recovery is over USB only. | [HW] |

Pin details: [[Pinout]]. Addresses: [[Memory Map]].

## The SMK-37

The SMK-37 Pro / Elite uses the same chip family with more keys, pads, faders and a different panel wiring.
See [[SMK-37]].

## What runs on it from the factory

M-VAVE's stock firmware (V15 at the time of writing) is a DX7-compatible six-operator FM synth. Reverse
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
