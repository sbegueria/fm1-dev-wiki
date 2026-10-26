# Boot Chain and Bootloaders

## From power-on to your code

```
mask ROM ──> bootloader ("uboot.boot", flash 0xA0, runs from RAM at 0x01C02000)
               │  sets clocks, reads the flash directory, sets up XIP decryption and the cache
               ├──> your app at 0x02000120 (XIP)                 normal start
               └──> staged update loader at 0x01C0A800 (RAM)     if an update record is pending
```

1. **The mask ROM** reads the flash header at offset 0, finds the boot container at `0xA0` (decoded with the fixed
   key `0xFFFF`), loads its payload to `0x01C02000` and jumps to it. **[RE] [HW]**
2. **The bootloader** (JieLi calls it the SPL or `uboot.boot`) sets the clock, reads the app directory, programs
   the SFC and the XIP key, invalidates the cache and starts the app at `0x02000120`. If an update record is
   pending, it starts the staged update loader instead. **[RE] [CODE]**
3. **Your app** starts with the clock, cache and XIP as the bootloader left them. Everything else (stacks, `.data`,
   `.bss`, vectors, peripherals) is yours to set up. **[CODE]**

The full contract, with every field, is in
[m-bootloader's SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §4–§6. It is the most
complete public description of the FM-1's boot path, written for a clean-room bootloader.

## The bootloaders you will meet

| Bootloader | Licence | Notes |
| --- | --- | --- |
| M-VAVE's stock (JieLi's SDK `uboot.boot`) | JieLi SDK, Apache-2.0 | Leaves the clock at 360 MHz. No keys, no menu. Most packages, including Felucca's, carry this same file. |
| [m-bootloader](https://github.com/jvitkauskas/m-bootloader) | MIT, clean room | REC at power-on: download mode. PLAY at power-on (or a firmware that fails to start): a recovery menu (boot, download mode, boot anyway once, wipe user data). Checks every firmware file's checksum before starting it. Tested on FM-1 and SMK-37 Pro. **Version 1.2 leaves the clock at 320 MHz** ([[Clocks and Power]]). |
| [fm1-bootloader](https://github.com/jvitkauskas/fm1-bootloader) | Apache-2.0 | A readable C restructuring of the SDK's stock bootloader, from a byte-identical reconstruction. |

**Installing an official M-VAVE update, or a firmware whose package carries the SDK bootloader, puts the stock
bootloader back** (m-bootloader README). If you rely on m-bootloader's recovery menu, check after each install.
SLOOP's installer was seen to keep m-bootloader (SPEC §13).

## What your app should do at start

From Felucca-family firmware and felucca-bsp. **[CODE]**

1. Mask interrupts, set the stack pointers, copy `.data` and any RAM code from flash, clear `.bss`.
2. Read and keep the **reset reason** first thing (P33's reset source: power-on, watchdog, VDDIO low, long press,
   soft reset).
3. Arm the **watchdog** early, and feed it from the main loop.
4. Put the **clock** and the **supply voltages** back to stock ([[Clocks and Power]]).
5. Set up the interrupt table (all 128 vectors to a fatal stub first) and the hardware guards ([[Debugging]]).
6. Count boots in a `NOINIT` area: if the firmware crashes twice during start-up, come up in a **safe mode** with
   no audio and USB on, so the user can install something else. Felucca-family firmwares do this.
7. Only then: the screen, the inputs, the audio, USB.

## Leaving your app

| To | How | Tag |
| --- | --- | --- |
| Reboot | chip reset through P33 (`P3_PR_PWR` bit 4) | [CODE] |
| ROM download mode | write `usb_update_mode\0` (16 bytes) at `0x01C7FD80`, then a software reset (power control `0x10000` bit 4). Don't jump into the ROM directly. | [HW] |
| The update loader | write the 112-byte `UPDATA_PARM` record at `0x01C7FD88` (and its copy in flash), then reset: the bootloader starts the staged loader | [CODE] |

Stop the audio DMA before any of these ([[Audio Output]]).

## Sources

- [m-bootloader](https://github.com/jvitkauskas/m-bootloader) README and SPEC.md §4, §6, §7, §13.
- [FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §3.
- Felucca's `hal/fm1_sys.h`, `crt0.S`, and its derivatives' start-up (GPL-3.0-only).
