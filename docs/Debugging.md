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

**[CODE]** The CPU control block (`0x01EEE000`…) has limits that raise the CPU exception. Felucca's
`hal/fm1_guard.h` (GPL-3.0-only) uses:

| Guard | Catches |
| --- | --- |
| EMU stack limit | the stack pointer leaving the stacks' range |
| Write limits | core 0 writing into the 256-byte guard bands under the stacks, or into `0x01C7FD50`–`0x01C7FFFF` (boot info, mailbox, vectors) once the vectors are set |
| Bus-invalid | fetch, read or write of unmapped space (the NULL page, writes into XIP) |
| PC limit | instruction fetch outside the code's range (two windows: flash, and RAM code) |

If you add code in RAM, widen the PC-limit window, or the guard faults on the first fetch ([Performance](Performance.md)).

## Survive a crash loop

**[CODE]** Count starts in `NOINIT`. If the firmware crashes or hangs (watchdog) in its first ~30 s twice in a
row, start in a **safe mode**: no audio, no engine, just the screen and USB with the update path, so the user can
install something else. Clear the count once the firmware has run long enough.

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
- the clock (from its registers, [Clocks and Power](Clocks-and-Power.md)) and the build stamp;
- per-stage timing of the heaviest block, if you have it;
- second-core job errors, if you use it ([Second Core](Second-Core.md)).

Put a dark, semi-transparent panel behind debug text if it sits over an animation: you want to keep the
animation running (it is part of the load you are measuring) and still read the numbers.

## Over USB

| Tool | What | Tag |
| --- | --- | --- |
| A CDC-ACM serial port | Felucca's `usb.c` can add one (`FELUCCA_CDC=1`) for a text console | [CODE] |
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

- Felucca's `hal/fm1_irq.h`, `fm1_guard.h`, `fm1_sys.h`, `fm1_debug.h`, `src/usb.c`, and its derivatives' main loops (GPL-3.0-only).
- [fm1-base README](https://github.com/charlesvestal/fm1-base) ("Development over USB").
- [FM-1 Transporter](https://github.com/kurogedelic/FM-1-transporter).
- [Dinghy](https://github.com/zvenson/dinghy) `firmware/hal/fm1_sys.h`, `fm1_irq.h`, `firmware/src/platform.c`, `docs/RULES.md` (GPL-3.0).
- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) development skill §3, §7; [MvaveFM1Unbricker README](https://github.com/Quixotic7/MvaveFM1Unbricker).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
