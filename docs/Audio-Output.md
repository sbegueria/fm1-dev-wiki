---
title: "Audio Output"
parent: Hardware
nav_order: 5
---

# Audio Output

## The path

**[CODE]** Open firmwares in the Felucca family output audio through the chip's **ALNK0** block (I2S-style audio
link, registers at `0x12E00`), fed by **double-buffered DMA**:

- The firmware owns one buffer split into two halves. Each half holds `N` stereo frames as `int32_t` pairs
  (L, R).
- **The sample format** (felucca-bsp's driver contract, since 2026-10-25): signed **Q23**, the 24 significant bits in
  bits 23:0, sign-extended into the 32-bit word, **not** shifted left by 8. Positive full scale is `0x007FFFFF`,
  negative full scale `0xFF800000`, silence `0`. A 16-bit PCM sample becomes this format when multiplied by 256.
  The driver does not clip: clip before you store. **[CODE]** (`boards/fm1/fm1_audio.h`, `docs/BOARD_IO.md`;
  [e7a22e3](https://github.com/hugelton/felucca-bsp/commit/e7a22e3)). M-VAVE's FM1 Open SDK uses the same
  format: stereo signed Q23 (`sdk/include/fm1_block.h`, `fm1_engine_control.h`). **[CODE]**
- When the DMA finishes one half, ALNK0 raises its interrupt (`FM1_IRQ_ALNK0`). The interrupt handler fills the
  half the DMA just left, while the DMA plays the other one.
- **Felucca uses 128-frame halves: 2.9 ms** per half. That is your hard deadline per block. Felucca changed from
  256 frames (5.8 ms) to 128 in **1.0** ([727f272](https://github.com/hugelton/Felucca/commit/727f272),
  2026-10-05). Forks taken from Felucca before 1.0 still use 256 (Dinghy's `platform.c`, for example). FM1 Open
  also renders 128-frame blocks (`FM1_BLOCK_FRAMES 128`). **[CODE]**

```c
/* the shape of the ALNK0 handler in Felucca-family firmware (GPL-3.0-only) */
void fm1_alnk0_irq(void)
{
    uint8_t p = fm1_audio_pending();
    fm1_audio_ack_aux(p);
    if (p & FM1_AUDIO_HALF) {
        uint32_t half = fm1_audio_free_half();
        render_block(&abuf[half * HALF_WORDS]);    /* must finish inside one half: 2.9 ms at 128 frames */
        fm1_audio_ack_half();
        if (fm1_audio_free_half() != half)
            xruns++;                               /* the DMA moved on while we rendered: a dropout */
    }
}
```

The last check is cheap, but it **does not catch every dropout**. felucca-bsp's `docs/BOARD_IO.md` says pending
bits can coalesce, and the before/after compare "misses an even number of transitions; it cannot prove that no
underrun occurred". To be sure, also time each render against the frame period and track the gaps between
interrupts. **[CODE]** Show both on screen during development ([Debugging](Debugging.md)).

Felucca also sheds load on its own: when two halves in a row take more than **85%** of a half (the render plus the
nested TIMER5 work), it fades out one voice, then one more each half while it lasts (`SHED_TICKS` in
`firmware/src/audio.c`). **[CODE]**

### Bring-up

**[CODE]** felucca-bsp's `fm1_audio_init()` drives four GPIOs on port C around the ALNK0 set-up, with 5 ms waits.
Since 2026-10-25 its source names them **PC0 MCLK, PC1 BCLK, PC2 LRCK, PC6 data**, and says: "PC6 is the I2S data
line, not a codec reset or mute pin" ([e7a22e3](https://github.com/hugelton/felucca-bsp/commit/e7a22e3)). Before
that, Felucca called them "codec control lines". `fm1_audio_stop()` drives the data line low.

**[CODE]** The checked start, `fm1_audio_init_checked()`, refuses to start (`FM1_AUDIO_ERR_CLOCK`) unless the audio
PLL reads as the stock set-up: `0x119A0 == 0x45400203` and `0x119A4 & 0x07FFFFFF == 0x07503026`. It never retunes
that PLL. A refused start is silence with nothing logged, so check its return value
([4d39207](https://github.com/hugelton/felucca-bsp/commit/4d39207), 2026-11-05).

Stop the DMA (`fm1_audio_stop()`) before a crash screen or before entering the ROM's download mode; otherwise the
last buffer loops as a buzz. Zero the buffer before flash erases, which run with interrupts off. **[CODE]**

## What is on the other end: disputed

| Source | Says | Tag |
| --- | --- | --- |
| felucca-bsp `boards/fm1/fm1_audio.h` (fdd709f) | first line still "ALNK0 I2S → external codec"; the pins PC0 MCLK, PC1 BCLK, PC2 LRCK, PC6 data | [CODE] |
| M-VAVE's [FM1 Open](https://github.com/m-vave/fm1_open/blob/4c7496d/app/board/fm1_audio.h) `app/board/fm1_audio.h` (2026-10) | "ALNK0: 24-bit stereo, two DMA halves, 128 frames"; PC0 MCLK, PC1 SCLK, PC2 LRCLK, **PC3 clock/RX**, **PC6 DAC**; "RX supplies the interrupt clock; FM1 has no connected analogue input" | [CODE] |
| [FM-1-RE](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) | "internal audio DAC", the stock firmware running its audio server at **44118 Hz** with 64-sample blocks | [RE] |
| [FM-1 resource hub](https://github.com/aroum/fm1-custom-fw/blob/main/REVERSE_ENGINEERING.md) §1.2 | an **external codec** on I2S: PC0 MCLK, PC1 BCLK, PC2 LRCK, PC6 the codec's power/reset; the chip's own DAC (DACL pin) as an auxiliary line. From board photos and JieLi's reference schematics; no codec part number given. | [?] |
| The hub's `HARDWARE_MODS.md` (2026-11-18), on the TNFM mod | cutting the output filter capacitors C98, C99, C100, C102 "to disconnect the internal DAC" | [?] |

Two code sources (felucca-bsp and M-VAVE's own) now agree that **PC6 carries the audio data**, not a codec reset. The
hub's "PC6 = power/reset" disagrees with both. Whether the converter is on-chip or a separate part is still open:
no one has given a part number. See [Open Questions](Open-Questions.md). If you know, please add a photo or a
measurement.

## Sample rate

**[CODE]** The rate is not exactly 44,100 Hz. felucca-bsp's contract: on the stock audio PLL, one frame is **544
ticks of the 24 MHz crystal**, so the rate is 24,000,000 / 544 ≈ **44,117.647 Hz**, about +400 ppm above 44,100. The
crystal's own tolerance comes on top. Use this rate for deadline sums; use 44,100 only where a protocol asks for the
advertised rate. `docs/BOARD_IO.md` says this ratio was "corroborated by prior FM-1 DMA-event measurements against
TIMER4 and USB frame timing". M-VAVE's FM1 Open uses the same figure (`FM1_AUDIO_HARDWARE_RATE 44118U`), and
FM-1-RE reported 44,118 Hz for the stock firmware.

**[HW]** A USB measurement agrees: with the USB audio input following the crystal, macOS's clock for the FM-1 read
**44,122 Hz** (Charles Vestal, felucca-bsp [d18879b](https://github.com/hugelton/felucca-bsp/commit/d18879b),
2026-11-11, one FM-1).

Under 360 and 320 MHz: m-bootloader's measurements found the same main PLL values (`0x45400203` / `0x3F503026`)
under M-VAVE's 015 and 016 bootloaders and under m-bootloader ([`docs/STOCK-V15-V16.md`](https://github.com/jvitkauskas/m-bootloader/blob/121cfbe/docs/STOCK-V15-V16.md),
2026-11-21). So the audio rate probably does not change with the system clock, but no one has recorded it under
each. **[?]**

## USB audio

**[CODE]** Felucca 1.0 and fm1-base can expose the master output as a USB Audio Class 1 input (16-bit stereo,
44.1 kHz, isochronous asynchronous IN), so a computer can record the FM-1 over the cable. fm1-base also supports
the other direction (the computer plays through the FM-1). The USB packet service must run every 1 ms frame, so in
Felucca-family firmware it runs nested inside the audio render from the 10 kHz timer interrupt.

**The packets follow the crystal.** **[HW]** felucca-bsp's USB audio input sends 44 frames a millisecond and one
more every ~8.5 ms, the pattern of the 44,117.6 Hz crystal rate. Two older ways of steering the packet size by how
full the ring was caused trouble on hardware:
- runs of long or short packets: iOS's Camera filled the short runs with silence, ~5 samples of dropout about once
  a second ([5b0879b](https://github.com/hugelton/felucca-bsp/commit/5b0879b), 2026-11-06);
- with long renders, a frame too many every render (+0.39%): macOS took the FM-1's rate as **44,292 Hz**, and
  recordings that resample on the host's clock warbled. A test tone was off by 6,400 ppm; after the fix, within
  7 ppm ([d18879b](https://github.com/hugelton/felucca-bsp/commit/d18879b), 2026-11-11; in fm1-base
  [2d2aa16](https://github.com/charlesvestal/fm1-base/commit/2d2aa16)).

**[HW] [?]** Forks with their own `usb.c` may still have a ring sized for the wrong block. In Dinghy (256-frame
renders, ring band set for 128), ~1.7% of the frames sent were repeats and recordings came out ~30 cents flat. A
larger band and ring fixed it on the reporter's unit ([zvenson/dinghy#13](https://github.com/zvenson/dinghy/issues/13),
kurzhaar168, 2026-11-14, still open on 2026-11-26).

**More channels.** **[HW]** `FELUCCA_UAC_CH` (fm1-base: `usb_audio_channels`) gives a USB input of 4, 6, 8 or 10
channels of 16 bits at 44.1 kHz: the master on 1 and 2, the app's own on the rest. A 10-channel packet is 920 bytes,
under full speed's 1023. LOOM-8 recorded 10 inputs on macOS with no gaps
([91583ec](https://github.com/hugelton/felucca-bsp/commit/91583ec), 2026-11-06). At 10 channels a packet takes
~0.6 ms on the bus. Serviced at 2 kHz, it crackled and the pitch dropped in Ableton Live; it now runs at every
0.1 ms tick ([fm1-base b2bb9b7](https://github.com/charlesvestal/fm1-base/commit/b2bb9b7)).

M-VAVE's view, for FM1 Open: "Eight separate audio tracks over USB are currently not supported because the
available USB bandwidth is insufficient for that configuration" (cream100,
[m-vave/fm1_open#6](https://github.com/m-vave/fm1_open/issues/6), 2026-10-21). Their channel width is not stated.
Both statements stand: the 10-channel input above is 16-bit.

**Other options.** felucca-bsp also has, each off by default: 24-bit input, an asynchronous 24-bit output, the
input on the computer's clock, and input and output as two named functions
([PR #49](https://github.com/hugelton/felucca-bsp/pull/49), Henri David, merged 2026-11-21). On one FM-1, 60 s of
playback and 3 × 60 s of recording at 44.1 kHz ran with no underruns (Charles Vestal, same PR). The new options'
own device runs were still to be reported. **[CODE]**

**FM1 Open** mixes USB playback (Q23, with the host's left/right gain) and Bluetooth A2DP audio (16-bit) into the
output, and sends the final mix to the USB input (`app/audio_output.c`). **[CODE]** M-VAVE tested stereo USB
playback on an FM-1 under Windows (fo#6, 2026-10-21). **[HW]**

**48 kHz.** kmplngj's Felucca fork offers the USB audio input at **44.1 or 48 kHz** (`FELUCCA_UAC_48K`, on by default;
44.1 kHz only up to 1.0.5.2), per its `BUILDING.md`. **[CODE]** The 44.1 → 48 kHz resampler in felucca-bsp's
`usb.c` (`uac_tap48`) had the 64-bit multiply pattern that fails on this CPU
([felucca-bsp#16](https://github.com/hugelton/felucca-bsp/issues/16), dsablic). The fix,
[14610bd](https://github.com/hugelton/felucca-bsp/commit/14610bd), moves each multiply out of line. It ran 30
minutes at 48 kHz on an FM-1 with no fault (dsablic, 2026-11-18). It costs 1.67× the instructions (in an emulator),
and on the device that firmware sent ~10% too few frames, which dsablic put down to its own scheduling and code
running from flash. **[HW]**

**macOS 13–15** show no USB audio device while MIDI is USB interface 1. Felucca **1.8.3** (2026-11-22) moves MIDI
after the audio streaming interface (`FELUCCA_UAC_MIDI_LAST`). Also, quit Chrome while testing: Chrome holding the
FM-1 open keeps macOS's audio service away from it. See [MIDI and USB](MIDI-and-USB.md). **[HW]**

## Choosing a block size

Shorter halves give lower latency and less time per block. Longer halves give more slack against an occasional
heavy block (a note-on, a parameter change), at the cost of latency. Felucca since 1.0 and FM1 Open use 128 frames
(2.9 ms); older Felucca forks use 256 (5.8 ms). Whatever you pick, measure the **worst** block, not the average:
see [Performance](Performance.md). If your firmware has a USB audio input, size its ring for your block (see
above).

## Sources

- [felucca-bsp](https://github.com/hugelton/felucca-bsp) at `fdd709f` (MPL-2.0): `boards/fm1/fm1_audio.h`,
  `docs/BOARD_IO.md` (Kerem Kilic, [e7a22e3](https://github.com/hugelton/felucca-bsp/commit/e7a22e3), 2026-10-25;
  [4d39207](https://github.com/hugelton/felucca-bsp/commit/4d39207)), `chip/usb.c` (commits
  [5b0879b](https://github.com/hugelton/felucca-bsp/commit/5b0879b),
  [d18879b](https://github.com/hugelton/felucca-bsp/commit/d18879b),
  [91583ec](https://github.com/hugelton/felucca-bsp/commit/91583ec),
  [14610bd](https://github.com/hugelton/felucca-bsp/commit/14610bd)); issue
  [#16](https://github.com/hugelton/felucca-bsp/issues/16); PR [#49](https://github.com/hugelton/felucca-bsp/pull/49).
  Since Felucca 1.7.0 these files are Felucca's `firmware/bsp` submodule; they used to be Felucca's `hal/fm1_audio.h`
  and `src/usb.c`.
- [Felucca](https://github.com/hugelton/Felucca) at `ac8cc0e` (GPL-3.0-only): `firmware/src/core.h` (`HALF_FRAMES`),
  `firmware/src/audio.c`; [727f272](https://github.com/hugelton/Felucca/commit/727f272) (1.0); the
  [1.8.3 release notes](https://github.com/hugelton/Felucca/releases/tag/v1.8.3); the main loop of its derivatives
  (`main_fm1.c` in X0X, AMB-1, DUB-1).
- [fm1-base](https://github.com/charlesvestal/fm1-base) commits [2d2aa16](https://github.com/charlesvestal/fm1-base/commit/2d2aa16),
  [b2bb9b7](https://github.com/charlesvestal/fm1-base/commit/b2bb9b7) (MPL-2.0).
- [M-VAVE FM1 Open](https://github.com/m-vave/fm1_open) at `4c7496d`: `app/board/fm1_audio.h`, `app/audio_output.c`,
  `sdk/include/fm1_block.h`; issue [#6](https://github.com/m-vave/fm1_open/issues/6).
- [m-bootloader `docs/STOCK-V15-V16.md`](https://github.com/jvitkauskas/m-bootloader/blob/121cfbe/docs/STOCK-V15-V16.md) (PLL values under each bootloader).
- [zvenson/dinghy#13](https://github.com/zvenson/dinghy/issues/13).
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) `docs/architecture.md` §1, §4.
- [kmplngj/Felucca BUILDING.md](https://github.com/kmplngj/Felucca/blob/main/BUILDING.md) (build options).
- [FM-1 resource hub](https://github.com/aroum/fm1-custom-fw) `REVERSE_ENGINEERING.md` §1.2, item 3; `HARDWARE_MODS.md`
  item 8 ([7ad4fe4](https://github.com/aroum/fm1-custom-fw/commit/7ad4fe4), 2026-11-18).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
