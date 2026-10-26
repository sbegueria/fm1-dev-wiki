# Performance

The FM-1 has plenty of CPU for one good synth voice set, and not much spare. Most audio dropouts come from a
few heavy blocks, not from a high average. This page collects what is known about where the time goes.

## Measure first

- **Time each block** with TIMER4 (24 MHz crystal, independent of the CPU clock): load = block time / block period.
- **Keep the peak** of a window (e.g. the heaviest block of each ~2 s), not only a running mean.
- **Count dropouts:** after rendering, check whether the DMA's free half changed ([[Audio Output]]).
- **Time stages inside the render** (voices, effects, mix) the same way, and show the heaviest block's breakdown.
  Show it on the device: a computer's profile does not predict the device's (below).
- **Show the build stamp** on the same screen, so you know which build you measured.

The core's own cycle counter may not run (see [[Clocks and Power]]); TIMER4 always does.

## Flash, XIP and the instruction cache

**[SDK]** The chip runs code from flash through an instruction cache (XIP). **[CODE]** Every open firmware does
by default.

**[HW]** What developers have seen:
- u/ZenLegume (r/MVaveFM1): with all DSP in SRAM, the DSP core's load "is fairly static for a given number of
  voices, with a little bit of jitter". After one change it became "heavier and much more erratic, bouncing up and
  down rapidly": some DSP code had ended up running from flash. Moving it back fixed it. His advice: **run the hot
  code from SRAM**; XiP looks fine "until your otherwise lightly loaded system starts having buffer underruns
  because a background process on the other core was fetching some data from flash".
- M-VAVE's FM1 Open marks about 60 functions (every engine's render, the mixers, the effects) as
  `sram_required` in its build script: **M-VAVE runs its whole DSP from SRAM**.
- DUB-1 (single core, October 2026): with the engine in flash, which pieces peaked highest on the device did
  not match a computer's profile of the same code at all, a sign of a code-layout (cache) effect. The engine's
  code had grown to ~93 KB with a ~55 KB hot path. An earlier test with ~62 KB of engine in RAM gave the same
  average load and **evener peaks**. A larger test was being measured when this page was written.

**[?]** The I-cache size and layout. Notes in DUB-1 give 32 KB, 8 ways × 4 KB, 32-byte lines; the SDK defines a
32-byte line (`CACHE_LINE_COUNT 32`) and has per-way registers (`ICACHE_WAY`, `DCACHE_WAY`); m-bootloader clears
cache arrays at `0x01F00000`… when it invalidates. Nobody has published a measurement. See [[Open Questions]].

### Putting hot code in RAM

**[CODE]** The recipe Felucca-family firmwares use for their flash driver, extended to DSP:

1. A linker region in SRAM (e.g. carve 64 KB out of the pool) and an output section for it, loaded from flash
   (`> FAST AT > XIP`).
2. Mark hot functions with `__attribute__((section(".fast_text")))`, or put whole object files' `.text` there in
   the linker script and mark the cold ones to stay in flash.
3. Copy the section at start-up, with `.data`, before anything calls it.
4. Open the PC-limit guard over the region, or the guard faults on the first fetch ([[Debugging]]).
5. Calls between flash and RAM need far calls ([[Memory Map]]). Calls inside the region are short.
6. Jump tables and constant tables stay in flash unless you move them too (`-fno-jump-tables`; tables `const` in
   a RAM data section).
7. Make the build check the region's size and the pool's headroom.

If all engines don't fit, load the current one into a reserved SRAM area when the user picks it (ZenLegume).

## Compiler and linker flags

**[CODE]** The FPU needs `-mcpu=r3 -mfprev1` (the SDK's own flags). Without them floats are done in software.

Common practice:
- `-O2` for the hot DSP files, `-Os` (or `-Oz`) for the rest. Code size matters for the cache too.
- `-ffunction-sections` and `--gc-sections`.
- `-ffp-contract=off` (no fused multiply-add) if you want the device and a host build to give the same samples.

**[CODE]** M-VAVE's FM1 Open (`sdk/Build.ps1`, at commit 4c7496d) compiles with
`-target pi32v2 -integrated-as -mcpu=r3 -mfprev1 -flto -ffunction-sections -fdata-sections -fno-common
-mllvm -pi32v2-large-program=true -O2 -fwrapv -fno-fast-math -ffp-contract=off`, `-Oz` for control-side files,
and links with LTO through the gold plugin with `-pi32v2-enable-simd=true`, `-enable-ipra=true`,
`-pi32v2-enable-rep-memop`, `-inline-threshold=5`, `-global-merge-on-const`, `--gc-sections`. What each of these
gains on the FM-1 has not been benchmarked publicly. **[?]**

## The CPU erratum: 64-bit multiplies

**[HW]** felucca-bsp `docs/CPU.md`: on the FM-1 at 360 MHz, code that inlines 32×32→64-bit multiplies among
16-bit loads now and then leaves a register **one bit off** (a loop index 0x12 read as 0x00800012), and the
firmware crashes. Found in fm1-radio's AAC decoder. Interrupts off, no instruction pairing, a `nop` after each
multiply, and a lower core voltage did not help. **Moving the multiplies into `noinline` functions did.**

```c
/* the workaround, from felucca-bsp docs/CPU.md (MPL-2.0) */
__attribute__((noinline)) int32_t mul_q31(int32_t a, int32_t b) { return (int32_t)(((int64_t)a * b) >> 31); }
```

It cost that decoder ~50% of its speed. Floating-point code (no 64-bit multiplies) never showed it. Not known:
the exact instruction pairs, whether 320 MHz avoids it. To find exposed code, look in the disassembly for
`rN_rM = rA * rB` beside `h[...]` loads.

## Integer maths

From u/ZenLegume's tips (r/MVaveFM1):
- Where a calculation needs 64 bits in the middle, don't keep the 64-bit value in a variable: cast and shift
  inline, so the compiler can drop the high word. (And mind the erratum above.)
- Multiply by a precomputed inverse instead of dividing.

## Small things that add up

- **memcpy/memset.** Byte-loop versions in early Felucca-family `libc.c` are slow; fm1-base made them 7–9× faster
  (its commit log). Matters if you clear canvases or copy buffers every frame.
- **Clearing the screen** (240×240×2 = 115 KB) every frame costs real time and evicts cache.
- **Fonts read from flash** while drawing evict audio code from the cache.

## Shed load instead of dropping out

A firmware can watch its own load and back off before it drops out: fewer voices, a cheaper filter stage, a
shorter tail, while the load is high. Users hear a thinner sound rather than a click.

## The second core

Two cores could roughly double the headroom, with caveats: [[Second Core]].

## Sources

- [r/MVaveFM1 thread](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/) (u/ZenLegume's post and replies).
- [FM1 Open `sdk/Build.ps1`](https://github.com/m-vave/fm1_open) (compile and link flags, `sram_required`).
- [felucca-bsp docs/CPU.md](https://github.com/hugelton/felucca-bsp/blob/main/docs/CPU.md).
- JieLi AC79 SDK `include_lib/driver/cpu/wl82/asm/cache.h`, `csfr.h`.
- DUB-1's CPU notes (October 2026) and Felucca's `tools/build.py` checks (GPL-3.0-only).
- [fm1-base](https://github.com/charlesvestal/fm1-base) commit history.
