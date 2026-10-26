---
title: "Memory Map"
parent: Hardware
nav_order: 2
---

# Memory Map

Addresses as the CPU sees them. See [Flash Storage](Flash-Storage.md) for the flash's own layout.

## CPU address space

| Range | What | Tag |
| --- | --- | --- |
| `0x00010000`… | Low-speed peripherals (clock, power, timers, UART, SPI, ADC, P33 bridge…) | [SDK] [CODE] |
| `0x00040000`… | High-speed peripherals, including the SFC (flash controller) at `0x40200` and XIP decryption at `0x40300` | [SDK] [CODE] |
| `0x00050000`… | GPIO (`0x50000 + 0x40 × port`, A=0 … H=7) and pin mux (`0x51000`…) | [SDK] [CODE] |
| `0x01C00000`–`0x01C7FFFF` | Internal SRAM used by firmware (512 KB). Code can run from it. | [CODE] [HW] |
| `0x01C02000` | Where the mask ROM loads and starts the bootloader | [RE] |
| `0x01C0A800` | Where the staged update loader is decompressed and started | [RE] [CODE] |
| `0x01C7FD50`–`0x01C7FFFF` | Reserved: boot info from the bootloader, the boot/update mailbox, the interrupt vector table at `0x01C7FE00` | [CODE] |
| `0x01EEE000`… | CPU control block ("corex2"): cache control (`0x01EEE008`), debug/guard limits, performance counters | [SDK] |
| `0x01EEF000`… | Interrupt controller (`ICFG`, pending), exception and trace registers | [CODE] |
| `0x01F00000`… | Cache internal arrays (the bootloader clears some ranges here to invalidate) | [RE] |
| `0x02000000`… | Flash XIP window, decrypted and cached. Flash offset `off` (≥ `0x4000`) reads at `0x02000000 + off − 0x4000`. | [CODE] |
| `0x02000120` | Entry point of every known application image | [CODE] [RE] |

**A disagreement on the XIP offset.** Felucca-family code maps flash offset `0x4120` (where `app.bin` starts) to
`0x02000120`. FM-1-RE's architecture notes write the mapping as `flash_offset − 0x4120 + 0x02000000` and put the
stock app's vectors at `0x020000A0`. Working firmware proves the first for open firmwares. Whether the stock
image is linked differently, or the note is off by the 0x120-byte header, is in [Open Questions](Open-Questions.md). **[?]**
| `0x04000000` | External RAM window. **Absent on the FM-1.** | [SDK] [RE] |

## One firmware's RAM layout (example)

This is the layout of Felucca's linker script as used in DUB-1 (GPL-3.0-only, © Leo Kuroshita). Other firmwares
divide RAM differently; the reserved top is the same for all.

```ld
MEMORY
{
    XIP    (rx)  : ORIGIN = 0x02000120, LENGTH = 0x8DFBC   /* the app slot: 581,564 bytes */
    RAMTEXT (rwx): ORIGIN = 0x01C00000, LENGTH = 0x6000    /* code that runs while the flash is busy */
    RAM    (rwx) : ORIGIN = 0x01C08000, LENGTH = 96K       /* .data, .bss */
    POOL   (rw)  : ORIGIN = 0x01C20000, LENGTH = 0x54000   /* big buffers: delay lines, canvases */
    NOINIT (rw)  : ORIGIN = 0x01C7C000, LENGTH = 0x3D50    /* survives a reset: crash record, boot counters */
}
```

Points to know:
- **The app slot is `0x8DFBC` bytes** (581,564). The package builder refuses a bigger image. **[CODE]**
- **Any SRAM can hold code.** The "(rw)" on the pool is only a linker label. M-VAVE's own open firmware marks its
  render functions to run from SRAM. See [Performance](Performance.md). **[CODE] [HW]**
- **Stacks** sit high in RAM with guard bands under them; Felucca's hardware guards fault on a write into a band.
  See [Debugging](Debugging.md). **[CODE]**
- **Leave `0x01C7FD50` and up alone** until your vectors are set up. The bootloader passes boot info there, and
  the update path writes its record at `0x01C7FD88` (112 bytes). Writing `usb_update_mode\0` at `0x01C7FD80`
  and resetting enters the mask ROM's download mode. **[CODE] [HW]**
- fm1-base's figures for its template: 96 KB of RAM for `.data`/`.bss` and a 336 KB pool (128 KB and 304 KB in a
  WiFi build). **[CODE]**

## Calling between flash and RAM

**[CODE]** A call from XIP code (`0x02…`) to RAM code (`0x01C…`) is out of the CPU's direct call range (23 bits).
Felucca calls RAM code through a pointer the compiler can't fold back:

```c
/* from Felucca's hal/fm1_flash.h (GPL-3.0-only, © Leo Kuroshita) */
static inline void *fl_far(void *p) { void *volatile q = p; return q; }
#define FL_FAR(fn) ((__typeof__(&fn))fl_far((void *)&fn))
```

Code that runs while the flash is switched off (erasing, programming) must touch only registers, the stack and
RAM: no constants in flash, no library calls, no helpers in flash. Felucca's build checks that its `.ram_text`
section contains no `call` instruction.

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §2.1, §2.2, §4.2, §6.
- [FM-1-RE architecture](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §2.
- Felucca's `firmware/app.ld`, `hal/fm1_xip.h`, `hal/fm1_flash.h`, `hal/fm1_sys.h` (GPL-3.0-only).
- [fm1-base README](https://github.com/charlesvestal/fm1-base), "What each costs".

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
