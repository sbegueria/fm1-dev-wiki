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
| `0x00011900`–`0x0001197C` | Radio analog block shared by WiFi and Bluetooth: PLL, filters, PA, DC and IQ trims; the crystal trim at `0x11930` must be ramped from RAM with interrupts off | [RE] [HW] |
| `0x00014000`–`0x0001405C` | Wireless control: bit 0 the Bluetooth RF path, bit 1 WiFi (`0x14000`); the coexistence controller's port `0x14004` (565 words, needed for WiFi receive); the channel in `0x1405C` | [SDK] [HW] |
| `0x00020000`–`0x000201FF` | Bluetooth Classic (BR/EDR) baseband and its 625 µs clock | [RE] |
| `0x00028000`–`0x00028054` | BLE baseband registers. The per-link registers are a block in SRAM that the engine reaches by DMA. | [RE] [CODE] |
| `0x00030000`–`0x00031FFF` | WiFi MAC and BBP window: JieLi's wl30 core, with the register layout of Ralink's RT3070. The frame DMA is at `0x30F00`–`0x30F1C` and can't read buffers above `0x01C20000`. Receive interrupt 52 never latched in probes; the open stacks poll | [RE] [CODE] [HW] |
| `0x00041200` | AES-128 block (stock firmware uses it for Bluetooth; the own stack encrypts in software) | [RE] |
| `0x00040000`… | High-speed peripherals, including the SFC (flash controller) at `0x40200` and XIP decryption at `0x40300` | [SDK] [CODE] |
| `0x00050000`… | GPIO (`0x50000 + 0x40 × port`, A=0 … H=7) and pin mux (`0x51000`…) | [SDK] [CODE] |
| `0x01C00000`–`0x01C7FFFF` | Internal SRAM used by firmware (512 KB). Code can run from it. | [CODE] [HW] |
| `0x01C02000` | Where the mask ROM loads and starts the bootloader | [RE] |
| `0x01C0A800` | Where the staged update loader is decompressed and started | [RE] [CODE] |
| `0x01C30000`… | Where m-bootloader's RAM apps are linked and run (up to below `0x01C7FD80`) | [CODE] |
| `0x01C7FD50`–`0x01C7FFFF` | Reserved: boot info from the bootloader, the boot/update mailbox, the interrupt vector table at `0x01C7FE00` | [CODE] |
| `0x01EEE000`… | CPU control block ("corex2"): cache control (`0x01EEE008`), debug/guard limits, performance counters | [SDK] |
| `0x01EEF000`… | Interrupt controller (`ICFG`, pending), exception and trace registers | [CODE] |
| `0x01F00000`… | Cache internal arrays (the bootloader clears some ranges here to invalidate) | [RE] |
| `0x01F28000`… | The data cache's 8 ways of 4 KiB, one after another. 1 to 7 of them can be made plain RAM ([Performance](Performance.md)). | [HW] |
| `0x02000000`… | Flash XIP window, decrypted and cached. Flash offset `off` (≥ `0x4000`) reads at `0x02000000 + off − 0x4000`. | [CODE] |
| `0x02000120` | Entry point of every known application image | [CODE] [RE] |
| `0x04000000` | External RAM window. **Absent on the FM-1** (but see the SDRAM question in [Open Questions](Open-Questions.md)). | [SDK] [RE] |
| `0xFFC00000`–`0xFFCFFFFF` | The chip's ROM. Dinghy's build refuses any call to, or constant loaded from, this range: a constant that happens to fall in it (`-32767 << 7` = `0xFFC00080`) trips the check too. | [CODE] |

**ROM services.** **[RE]** M-VAVE's V13 calls the ROM at fixed addresses: `0xFFC00404` flash read, `0xFFC00476`
flash write, `0xFFC05C4C` delay, `0xFFC00EEA` / `0xFFC00F2E` P33 read / write (FM-1-RE
[`io/09-storage.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/09-storage.md) §1,
[`io/07-input.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/07-input.md) §1,
[`io/01-boot.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/01-boot.md) §5, marked high).

### The stock app runs at `0x02000120`

**[RE]** Felucca-family code maps flash `0x4120` (where `app.bin` starts) to `0x02000120`, and working firmware
proves it for open firmwares. FM-1-RE reads M-VAVE's V13 `app.bin` as if its first byte were at `0x02000000`
(`docs/architecture.md` §2: `flash_offset − 0x4120 + 0x02000000`, vectors at `0x020000A0`). This wiki's reading of
FM-1-RE's own data (2026-11-26) is that **the stock app is linked at `0x02000120` too**, and FM-1-RE's app addresses
are 0x120 low:

- The stock package's metadata, as FM-1-RE unpacked it, gives `entry-point: 33554720`, which is `0x02000120`
  (`firmware-images/v13/raw_fw/FM-1.fwsc_unpack/jlfw.yaml`). `docs/01-firmware-analysis.md` converts it as `0x020000A0`.
- The start-up code jumps to the literal `0x02001C24`, but FM-1-RE finds that function starting at its `0x02001B04`,
  0x120 lower. The entry handed to core 1, `0x020001B8`, is FM-1-RE's core-1 vector `0x02000098` + 0x120
  (`io/01-boot.md` §4, §5.2 "Anomaly note").
- Tables sit 0x120 below the addresses the code loads for them (`io/04-synth-engine.md` §1 "Reading note";
  `io/05-midi.md` §3.3).
- The start-up code copies `.data`, `0x9E7C` bytes, from `0x02084820`, then 32 bytes from `0x0208E69C`
  (`io/01-boot.md` §2, §2.1). `app.bin` is `0x8E59C` bytes. Placed at `0x02000000`, the copies would run 0x100 bytes past
  the end of the file; placed at `0x02000120`, they end exactly at its last byte (this wiki's arithmetic).
- fm1-to-smk37 says independently that stock `app.bin` runs at `0x02000120`, checked with tables of string pointers
  ([`docs/05-reverse-engineering.md`](https://github.com/genkigenki/fm1-to-smk37/blob/main/docs/05-reverse-engineering.md)
  §3, stock 016). felucca-bsp's package builder writes `0x02000120` into the `app_area_head` entry (`tools/fm1pkg.py`).

FM-1-RE calls the offsets "anomalies" or a "shard artifact" and does not conclude; its `io/01-boot.md` header says
the plain mapping was verified. Both readings are shown here. **When you use an FM-1-RE address of code or data in
`app.bin`, add `0x120`** to get the address the CPU uses. Its RAM, register and ROM addresses are not affected, nor
an address it computed from a literal in the code (that one is already the CPU's).

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

**Why RAM and POOL? A convention, not a hardware limit.**
- **It's one SRAM.** Felucca's linker script says so itself: RAM holds `.data`/`.bss`, "what the audio and key-scan
  interrupts touch, small variables"; POOL holds big zero-initialised buffers (section `.pool`, not loaded from flash),
  and "the RAM / POOL boundary only splits the budget" (`firmware/app.ld` header, Leo Kuroshita; 96/336 KB until 1.4,
  72/360 KB since 1.5). Felucca's build reports the two budgets apart and checks the pool's headroom; no memory guard
  sits on the boundary. **[CODE]**
- **Same speed.** Timed on one FM-1 (Optimist, 2026-10-25): a 32-bit read took 35.2 ns in RAM and in POOL alike, a
  write 35.1 ns; code ran at the same speed from RAM code, RAM and cached flash (Optimist
  [`docs/MEMORY-MAP.md`](https://github.com/w0ts/optimist/blob/2b559c1d/docs/MEMORY-MAP.md) §2.3). Not measured:
  contention with the screen's DMA, and the second core's access. **[HW]**
- **What it buys** is bookkeeping: the interrupt side's memory stays small and reported on its own, and a buffer tagged
  `.pool` reads as "main loop only" (Felucca's comments: "main loop only: off the audio code's .bss"). **[CODE]**
- **Packed layouts** drop the boundary: Optimist packs RAM code, data and big buffers into one region in every build
  since 2026-10-25 (`RAM_PACKED`), and fm1-base offers `ram_layout = "packed"` (below). How buffers are handed out
  (fixed sections, or an arena reset as a whole, as Optimist uses for effects and engines) is a separate question from
  where a boundary sits. **[CODE]**
- **One real limit lies nearby:** the radio's frame DMA can't read above `0x01C20000` (below), which in the layout
  above is exactly where POOL starts. Radio buffers must stay low, whatever the layout. **[HW]**

(Asked and answered on r/MVaveFM1, "Why the POOL/RAM split?", J-MW, November 2026, with replies from deamonSLINKY and
keremimo; checked here against the sources above.)

Points to know:
- **The app slot is `0x8DFBC` bytes** (581,564) in the stock layout. The package builder refuses a bigger image.
  fm1-base can move the app area's end with `app_end` (up to `0xE0000`; `0xCE000` gives an `app.bin` of
  823,228 bytes), and its update loader writes the bigger area ([Flash Storage](Flash-Storage.md)). **[CODE]**
- **Any SRAM can hold code.** The "(rw)" on the pool is only a linker label. M-VAVE's own open firmware marks its
  render functions to run from SRAM. See [Performance](Performance.md). **[CODE] [HW]**
- **Stacks** sit high in RAM with guard bands under them; Felucca's hardware guards fault on a write into a band.
  See [Debugging](Debugging.md). **[CODE]**
- **Leave `0x01C7FD50` and up alone** until your vectors are set up. The bootloader passes boot info there, and
  the update path writes its record at `0x01C7FD88` (112 bytes). Writing `usb_update_mode\0` at `0x01C7FD80`
  and resetting enters the mask ROM's download mode. **[CODE] [HW]**
- Dinghy's layout differs again: a 360 KB pool from `0x01C1A000`, stacks in `0x01C74000`–`0x01C7C000` with
  256-byte guard bands (mvave-fm1 development skill §3). **[CODE]**
- `NOINIT` survives a reset **and** an entry into download mode: MvaveFM1Unbricker reads 512 bytes at
  `0x01C7C000` through the ROM's loader to recover Melodee's crash record. **[CODE]**
- fm1-base's figures for its template: 96 KB of RAM for `.data`/`.bss` and a 336 KB pool (128 KB and 304 KB in a
  WiFi build). With `ram_layout = "packed"` the RAM code, `.data`/`.bss` and the pool share **one budget of 464 KB**
  instead, 8 KB kept spare. `PLAT_SPARE` puts a buffer in NOINIT's room past the records that must survive a reset
  (about 13.8 KB of its 15.7 KB), zeroed at boot (fm1-base README "What each costs", `include/plat.h`, at
  [3039770](https://github.com/charlesvestal/fm1-base/tree/3039770)). **[CODE]**
- **The radio's frame DMA does not reach above `0x01C20000`.** fm1-base's own WiFi stack keeps its frame buffers
  below that address, and its build checks it (README). **[CODE]**
- Felucca itself has since changed its layout: at [ac8cc0e](https://github.com/hugelton/Felucca/tree/ac8cc0e)
  (2026-11-24) `firmware/app.ld` has 72 KB of RAM and a pool of `0x5A000` from `0x01C1A000`. **[CODE]**
- M-VAVE's V13, for comparison: `.data` (`0x9E7C` bytes) at `0x01C00000`, `.bss` after it up to `0x01C211FB`, core 0's
  stacks topped at `0x01C14BB4` and `0x01C15BB4`, core 1's at `0x01C15EB4` and `0x01C16EB4`, and the same vector table
  at `0x01C7FE00` (FM-1-RE [`io/01-boot.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/01-boot.md) §1,
  marked high). **[RE]**

## Calling between flash and RAM

**[CODE]** A call from XIP code (`0x02…`) to RAM code (`0x01C…`) is out of the CPU's direct call range (23 bits).
Felucca calls RAM code through a pointer the compiler can't fold back:

```c
/* from felucca-bsp chip/fm1_flash.h (MPL-2.0, © Leo Kuroshita); Felucca's own copy was hal/fm1_flash.h before 1.7.0 */
static inline void *fl_far(void *p) { void *volatile q = p; return q; }
#define FL_FAR(fn) ((__typeof__(&fn))fl_far((void *)&fn))
```

Code that runs while the flash is switched off (erasing, programming) must touch only registers, the stack and
RAM: no constants in flash, no library calls, no helpers in flash. Felucca's build checks that its `.ram_text`
section contains no `call` instruction.

## Sources

- [m-bootloader SPEC.md](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md) §2.1, §2.2, §4.2, §6, §11.8 (RAM apps).
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) (WTFPL; stock V13; last commit 2026-09-08):
  [`docs/architecture.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) §2,
  [`docs/01-firmware-analysis.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/01-firmware-analysis.md),
  [`docs/io/01-boot.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/01-boot.md) §1, §2, §4, §5,
  [`docs/io/04-synth-engine.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/04-synth-engine.md) §1,
  [`docs/io/05-midi.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/05-midi.md) §3.3,
  [`docs/io/07-input.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/07-input.md) §1,
  [`docs/io/09-storage.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/09-storage.md) §1,
  [`firmware-images/v13/raw_fw/FM-1.fwsc_unpack/jlfw.yaml`](https://github.com/AL-255/FM-1-RE/blob/main/firmware-images/v13/raw_fw/FM-1.fwsc_unpack/jlfw.yaml).
- [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) `docs/05-reverse-engineering.md` §3 (CC BY-NC 4.0; cited, not copied).
- Felucca's `firmware/app.ld` (GPL-3.0-only). [felucca-bsp](https://github.com/hugelton/felucca-bsp) `chip/fm1_xip.h`,
  `chip/fm1_flash.h`, `chip/fm1_sys.h`, `docs/EXPERIMENTAL.md` (MPL-2.0); Felucca has used them as its
  `firmware/bsp` submodule since 1.7.0, and its `hal/` copies are gone.
- [fm1-base README](https://github.com/charlesvestal/fm1-base), "What each costs", "A bigger app slot";
  `docs/bt/BLE-HW-FACTS.md` §1 and `docs/wifi/README.md` (radio registers).
- [Dinghy](https://github.com/zvenson/dinghy) `docs/RULES.md`, `tools/build.py` (GPL-3.0); [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) development skill §3.
- [MvaveFM1Unbricker README](https://github.com/Quixotic7/MvaveFM1Unbricker) (`retained-ram.bin`).
- The radio blocks: Optimist `docs/WIFI-HW-FACTS.md` §2 and fm1-base `docs/bt/BLE-HW-FACTS.md` §1 (from Optimist, 2026-10); felucca-bsp `chip/fm1_wifi.h`; modd `docs/hardware/wifi/SPECSHEET.md` §4 (`a833219`). More on [Bluetooth and WiFi](Bluetooth-and-WiFi.md).
- Felucca `firmware/app.ld` header (GPL-3.0-only); Optimist `docs/MEMORY-MAP.md` §2.3 at `2b559c1d`; r/MVaveFM1, J-MW, "Why the POOL/RAM split?" (November 2026).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
