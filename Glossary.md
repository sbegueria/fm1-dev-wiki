# Glossary

| Term | Meaning |
| --- | --- |
| **AC791N / AC79 / WL82** | JieLi's chip family in the FM-1 and SMK-37. "AC79" in SDK names, "WL82" in the CPU directory and the ROM's USB name. |
| **ALNK0** | The chip's audio link (I2S-style) block; feeds the audio output by DMA. |
| **App slot** | The flash space for the application image: `0x8DFBC` bytes from flash `0x4120`. |
| **BTIF** | A small flash region for Bluetooth pairing data. |
| **Chip key** | A 16-bit key used to encrypt the app area in flash (`0x980F` on known FM-1s). |
| **Download mode / UBOOT** | The mask ROM's USB mode (`4C4A:8057`, "WL80UBOOT1.00"), for reading and writing flash with no firmware running. |
| **ENC** | JieLi's simple stream cipher (CRC16-based key schedule). |
| **`.fwsc`** | The firmware package file. See [[Firmware Package Format]]. |
| **HSB / LSB** | The high-speed and low-speed bus clocks (180 and 60 MHz at stock). Peripheral dividers are set against LSB. |
| **isd_config** | A config blob in the boot area that encodes the chip key. |
| **JLFS** | JieLi's flash directory format (32-byte entries with CRC16). |
| **M-UPGRADE** | M-VAVE's updater application, and the USB-MIDI SysEx update protocol it uses. |
| **Mask ROM** | Code fixed in the chip. It loads the bootloader, and offers download mode. |
| **NOINIT** | A RAM area not cleared at start-up, so it survives a reset (crash records, boot counters). |
| **P33** | The always-on power domain (power, reset, watchdog, analog settings), reached through a serial bridge. |
| **pi32v2** | JieLi's CPU architecture. Compiler target `-mcpu=r3`, FPU `-mfprev1`. |
| **PLL1** | The PLL whose 540 MHz × 2/3 gives the stock 360 MHz system clock. |
| **SFC** | The serial flash controller that maps flash into the XIP window and decrypts it. |
| **Soft key** | SysEx `F0 22 24 35 7D F7`: asks a running firmware to enter download mode. |
| **SPL / bootloader / `uboot.boot`** | The second stage after the mask ROM. It sets clocks and starts the app or the update loader. |
| **SYSVDD / VDC14** | The core-and-SRAM supply and the rail it is regulated from. See [[Clocks and Power]]. |
| **UFW** | JieLi's update container inside a `.fwsc`. |
| **UPDATA_PARM** | The 112-byte update record (RAM `0x01C7FD88` and flash) that makes the bootloader start the staged loader. |
| **USB_KEY** | A 16-bit signal on USB D+/D− at power-on that forces the mask ROM into download mode. |
| **USR / USRTRIM / USRFLASH** | User data regions in flash; on the SMK-37 USRTRIM holds factory calibration. |
| **VM** | JieLi's key/value config store region in flash. |
| **XIP** | Execute in place: running code directly from flash through the cache. |
| **Xrun / dropout** | A block of audio not ready in time; heard as a click or a gap. |
