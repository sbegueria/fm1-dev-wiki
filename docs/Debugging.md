---
title: "Debugging"
parent: Writing firmware
nav_order: 4
---

# Debugging

The FM-1 has no debug header. Everything has to come out through the screen, USB, or memory that survives a reset.

## A crash screen with the PC

**[CODE]** Felucca-family firmware points all 128 interrupt vectors at a fatal stub, enables the CPU-exception
vector, and on any fault writes a **crash record** (PC, cause registers, a count) into a `NOINIT` area that
survives the reset. It shows the record on screen, then resets. On the next start the firmware can show "crashed
at PC …" for a moment.

To find the line: look up the PC in your build's disassembly (`objdump -d app.elf`) or symbol map. Keep the ELF
of every build you hand out, and show a **build stamp** (commit and time) on a settings or info page so you know
which ELF a report belongs to.

## Hardware guards

**[CODE]** The CPU control block (`0x01EEE000`…) has limits that raise the CPU exception. felucca-bsp's
`chip/fm1_guard.h` (MPL-2.0; Felucca's `hal/fm1_guard.h` until 1.7.0) uses:

| Guard | Catches |
| --- | --- |
| EMU stack limit | the stack pointer leaving the stacks' range |
| Write limits | core 0 writing into the 256-byte guard bands under the stacks, or into `0x01C7FD50`–`0x01C7FFFF` (boot info, mailbox, vectors) once the vectors are set |
| Bus-invalid | fetch, read or write of unmapped space (the NULL page, writes into XIP) |
| PC limit | instruction fetch outside the code's range (two windows: flash, and RAM code) |

If you add code in RAM, widen the PC-limit window, or the guard faults on the first fetch ([Performance](Performance.md)).

**[RE]** M-VAVE's V13 uses the stack guard per task: at each context switch it writes the new task's stack range to
`0x01EEF0E0` and `0x01EEF0DC`, plus `0x200` for core 1 (FM-1-RE
[`io/02-rtos.md`](https://github.com/AL-255/FM-1-RE/blob/main/docs/io/02-rtos.md) §2.1, marked high).

## Survive a crash loop

**[CODE]** Count starts in `NOINIT`. If the firmware crashes or hangs (watchdog) in its first ~30 s twice in a
row, start in a **safe mode**: no audio, no engine, just the screen and USB with the update path, so the user can
install something else. Clear the count once the firmware has run long enough.

**Make the `NOINIT` record `volatile`.** **[HW]** felucca-bsp's `examples/hwtest` kept its boot guard as a plain
`static` struct in `.noinit`, only ever given its magic number. The compiler treated it as zero at start and shrank
the magic and a counter to 1-bit flags, so the 32-bit check was gone. Whatever RAM stock V15 left behind then passed
as a record: the first boot after an install landed in download mode (UBOOT) 3 times out of 3 (hellsend,
[felucca-bsp#17](https://github.com/hugelton/felucca-bsp/issues/17), 2026-10-28), and 2 of 2 over V16
(charlesvestal). Declared `volatile`, and with counters above 1 refused as well, it came back on its own 2 of 2
times ([14610bd](https://github.com/hugelton/felucca-bsp/commit/14610bd), 2026-10-31). fm1-base's and
Felucca-based firmwares' guards are globals and were checked whole in their built images. If you write your own,
make it `volatile` and check the emitted code.

Arm the **watchdog** early in start-up and feed it from the main loop. The reset reason (P33) tells you next time
whether it was the watchdog.

**[CODE]** The watchdog is in P33 (`P3_WDT_CON`, P33 register `0x80`). Dinghy's `hal/fm1_sys.h` arms it in reset
mode with a timeout code from `0xA` (1 s) to `0xF` (32 s), and its platform uses `0x0D`; mvave-fm1's skill calls
that 8 s, which fits a doubling per step. Feed it from the main loop at least that often.

## Divide by zero: the trap is off in the Felucca family

**[CODE]** The CPU can trap a division by zero (Dinghy clears bit 2 of `EMU_CON` to turn the trap off). SLOOP 2.4
and Felucca 1.0.3.1 turned it **off**, and Dinghy follows: JieLi's clang treats a divide as harmless and may hoist
a loop-invariant division *ahead of* the test that guards it, so a guarded divide could trap. With the trap off,
such a hoisted divide gives a value nobody uses; the price is that a real divide by zero gives a **wrong value**
instead of a crash report. Test divisors yourself, and build the host simulator with
`-fsanitize=integer-divide-by-zero` to catch real ones (Dinghy `hal/fm1_irq.h`, whose comment cites
"#61, after PR #111 by spinkham"; `docs/RULES.md`).

## On-screen meters

Show, on a debug or settings page:
- the audio load: now and the peak of the last ~2 s ([Performance](Performance.md));
- the dropout count;
- the clock (from its registers, or the cycle counter against TIMER4 once `DBG_CON` bits 0–2 are set:
  [Clocks and Power](Clocks-and-Power.md)) and the build stamp;
- the supply rails, if you set them (fm1-base's `tools/fm1_remote.py log` prints them as the bootloader left them
  and as they are now);
- per-stage timing of the heaviest block, if you have it;
- second-core job errors, if you use it ([Second Core](Second-Core.md)).

Put a dark, semi-transparent panel behind debug text if it sits over an animation: you want to keep the
animation running (it is part of the load you are measuring) and still read the numbers.

## Nothing hidden in the stock firmware

**[RE]** FM-1-RE searched stock V13 and V14 for a way in and found none
([`TODO_aug2.md`](https://github.com/AL-255/FM-1-RE/blob/main/TODO_aug2.md), "No hidden console or recovery chord has
been found"; `analysis/device/debug-surfaces.md`):
- no serial console: the stock `isd_config.ini` sets a debug baud rate (`UTBD`) but no debug pin (`UTTX`) and no UART
  update pin (`UTRX`); the application's UART is the MIDI input;
- no USB serial port, and none of the SDK's factory, test or AT command frames;
- no trace of the SDK's online configuration tool protocol (frames starting `5A AA A5`), which can read and write flash;
- the two-key chord in the key scanner only resets octave and semitone ([Controls](Controls.md)).

So a crash in stock firmware tells you nothing over USB; the community's own tools are below.

## Over USB

| Tool | What | Tag |
| --- | --- | --- |
| A CDC-ACM serial port | felucca-bsp's `chip/usb.c` adds one with `FELUCCA_CDC=1`; Felucca builds it in by default for its text console (`BUILDING.md`, MENU > USB SERIAL) | [CODE] |
| `fm1_debug.py` (X0X debug builds) | peek and poke registers and RAM in a few allowed ranges, time a loop against TIMER4 | [CODE] |
| `tools/fm1_remote.py` (fm1-base) | press buttons and keys, turn encoders, read back what the screen says, the last crash and the radio's log | [CODE] |
| "Report a problem" (fm1-base's installer page) | reads the crash record and log in the browser, ready to copy | [CODE] |
| MvaveFM1Unbricker in download mode | saves 512 bytes of `NOINIT` (`0x01C7C000`) as `retained-ram.bin` before its backup: a crash record survives a trip into download mode | [CODE] |
| `fm1t memr` (FM-1 Transporter) | read memory, e.g. a debug struct in `NOINIT` (named in a Felucca-family code comment; how it reaches RAM not checked here) | [?] |

Restrict register peeks to known ranges: a read of a gated peripheral block can fault.

## Keep hardware facts honest

- Keep register addresses in one place (a HAL) with a comment saying **where each fact comes from** and whether it
  was verified on hardware. A header that says "Not verified on hardware" is more useful than one that is silent.
- When a fact is in doubt, compare with [felucca-bsp](https://github.com/hugelton/felucca-bsp) and
  [m-bootloader's SPEC](https://github.com/jvitkauskas/m-bootloader/blob/main/docs/SPEC.md), which more people read.
- If a build check fails, make sure no stale package is left where you'd pick it up.

## Sources

- [felucca-bsp](https://github.com/hugelton/felucca-bsp) `chip/fm1_irq.h`, `fm1_guard.h`, `fm1_sys.h`, `fm1_debug.h`,
  `fm1_perf.h`, `usb.c` (MPL-2.0), which Felucca has used as its `firmware/bsp` submodule since 1.7.0 (its `hal/`
  copies are gone); [issue #17](https://github.com/hugelton/felucca-bsp/issues/17).
- Felucca's `BUILDING.md` and its derivatives' main loops (GPL-3.0-only).
- [fm1-base README](https://github.com/charlesvestal/fm1-base) ("Development over USB"), `docs/SECOND-CORE.md` ("The supplies").
- [FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter).
- [Dinghy](https://github.com/zvenson/dinghy) `firmware/hal/fm1_sys.h`, `fm1_irq.h`, `firmware/src/platform.c`, `docs/RULES.md` (GPL-3.0).
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) development skill §3, §7; [MvaveFM1Unbricker README](https://github.com/Quixotic7/MvaveFM1Unbricker).
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) (WTFPL; stock V13, V14): `docs/io/02-rtos.md` §2.1, `TODO_aug2.md`,
  [`analysis/device/debug-surfaces.md`](https://github.com/AL-255/FM-1-RE/blob/main/analysis/device/debug-surfaces.md).
  Its code addresses need `0x120` added ([Memory Map](Memory-Map.md#the-stock-app-runs-at-0x02000120)).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
