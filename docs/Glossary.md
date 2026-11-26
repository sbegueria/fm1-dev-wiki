---
title: "Glossary"
parent: Community
nav_order: 5
---

# Glossary

| Term | Meaning |
| --- | --- |
| **AC791N / AC79 / WL82** | JieLi's chip family in the FM-1 and SMK-37. "AC79" in SDK names, "WL82" in the CPU directory and the ROM's USB name. |
| **ALNK0** | The chip's audio link (I2S-style) block; feeds the audio output by DMA. |
| **App slot** | The flash space for the application image: `0x8DFBC` bytes from flash `0x4120`. |
| **BTIF** | A small flash region for Bluetooth pairing data. |
| **Boot stub** | A small piece at the start of the app area (ChoralRoot's: flash `0x4120…0x4FFF`) that checks the firmware before starting it and falls back to update mode after a cut-off write. |
| **BSP** | Board support package: the code that drives one chip and board (clock, interrupts, flash, USB, pins). felucca-bsp is the FM-1's shared one. |
| **Cache RAM** | Data-cache ways (4 KiB each, at `0x01F28000`) switched to plain RAM. Fewer ways left as cache make big flash reads slower. |
| **Chip key** | A 16-bit key used to encrypt the app area in flash (`0x980F` on known FM-1s). |
| **Download mode / UBOOT** | The mask ROM's USB mode (`4C4A:8057`), for reading and writing flash with no firmware running. Its USB name reads "WL80UBOOT1.00"; as a disk it shows as "WL82 UBOOT1.00" (SCSI vendor `WL82`, product `UBOOT1.00`), seen so on an SMK-37 Pro. |
| **ENC** | JieLi's simple stream cipher (CRC16-based key schedule). |
| **Fast install** | felucca-bsp's update loader mode (2026-11): the installer sends a per-sector CRC table, and only the 4 KiB sectors that differ are sent and written; a cut-off install resumes. Loaders and hosts without it use the classic install. |
| **`.fwsc`** | The firmware package file. See [Firmware Package Format](Firmware-Package-Format.md). |
| **HSB / LSB** | The high-speed and low-speed bus clocks (180 and 60 MHz at stock). Peripheral dividers are set against LSB. |
| **Identity** | The `FM-1_nnn` string a firmware or loader answers in the update handshake; installers use it to tell firmwares apart. |
| **isd_config** | A config blob in the boot area that encodes the chip key. |
| **JLFS** | JieLi's flash directory format (32-byte entries with CRC16). |
| **M-UPGRADE** | M-VAVE's updater application, and the USB-MIDI SysEx update protocol it uses. |
| **Mask ROM** | Code fixed in the chip. It loads the bootloader, and offers download mode. |
| **MIDI mode** | m-bootloader's mode that takes a firmware package from a browser installer over USB MIDI, with no firmware running. |
| **NOINIT** | A RAM area not cleared at start-up, so it survives a reset (crash records, boot counters). |
| **Owner tag** | A firmware's own mark on its stored data (fm1-to-smk37's ports keep it at `0xAF000` on the SMK-37), so data without it is treated as another firmware's. |
| **P33** | The always-on power domain (power, reset, watchdog, analog settings), reached through a serial bridge. |
| **pi32v2** | JieLi's CPU architecture. Compiler target `-mcpu=r3`, FPU `-mfprev1`. |
| **PLL1** | The PLL whose 540 MHz × 2/3 gives the stock 360 MHz system clock. |
| **PRCT** | The region-table entry for the protected app area, starting at flash 0 (`0x9D000` long on the SMK-37). |
| **RAM app / RAM stage** | A program loaded into RAM and run there without writing the flash (m-bootloader SPEC §11.7–11.8): M-Unbricker, m-apps, and m-bootloader's helper started through a firmware's own update route. |
| **Skill** | A folder of instructions (`SKILL.md`) that an AI coding assistant loads for a task. See [AI Skills](AI-Skills.md). |
| **SFC** | The serial flash controller that maps flash into the XIP window and decrypts it. |
| **Sliced erase** | A flash sector erase split into windows of about 1 ms: the erase is suspended (`75h`), interrupts and XIP run for a moment, then it resumes (`7Ah`), so audio keeps playing. felucca-bsp `FL_ERASE_SLICED`, off by default. |
| **Soft key** | SysEx `F0 22 24 35 7D F7`: asks a running firmware to enter download mode. |
| **SPL / bootloader / `uboot.boot`** | The second stage after the mask ROM. It sets clocks and starts the app or the update loader. |
| **SYSVDD / VDC14** | The core-and-SRAM supply and the rail it is regulated from. See [Clocks and Power](Clocks-and-Power.md). |
| **TransitFM1** | A silent stopover firmware for moving between two custom firmwares. |
| **`trimdat`** | The marker at the end of the SMK-37's calibration record (flash `0xC1224`); a quick check that the calibration is there. |
| **UC25HQ80** | The flash chip in many newer FM-1s (JEDEC ID `0xB36014`), beside the Puya P25Q80 (`0x856014`). Firmware that checks the ID must accept both. |
| **UFW** | JieLi's update container inside a `.fwsc`. |
| **UPDATA_PARM** | The 112-byte update record (RAM `0x01C7FD88` and flash) that makes the bootloader start the staged loader. |
| **USB_KEY** | A 16-bit signal on USB D+/D− at power-on that forces the mask ROM into download mode. |
| **USR / USRTRIM / USRFLASH** | User data regions in flash. On the SMK-37, USRTRIM (`0xC1000`) holds the factory calibration and USRFLASH (`0xC2000`) the presets, with the unit's wheel values; stock firmware has been seen to erase the calibration when USRFLASH is blank ([SMK-37](SMK-37.md)). |
| **VM** | JieLi's key/value config store region in flash. |
| **Workbench** | M-VAVE's web page for FM1 Open and SMK37 Open: updates, an in-browser preview, project, sample and DX7 managers, and the source + SDK download. |
| **XIP** | Execute in place: running code directly from flash through the cache. |
| **Xrun / dropout** | A block of audio not ready in time; heard as a click or a gap. |

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
