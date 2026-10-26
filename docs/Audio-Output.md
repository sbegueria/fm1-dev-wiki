# Audio Output

## The path

**[CODE]** Open firmwares in the Felucca family output audio through the chip's **ALNK0** block (I2S-style audio
link, registers at `0x12E00`), fed by **double-buffered DMA**:

- The firmware owns one buffer split into two halves. Each half holds `N` stereo frames as `int32_t` pairs
  (L, R), **24-bit, left-justified**.
- When the DMA finishes one half, ALNK0 raises its interrupt (`FM1_IRQ_ALNK0`). The interrupt handler fills the
  half the DMA just left, while the DMA plays the other one.
- Felucca uses **256-frame halves at 44.1 kHz: 5.8 ms** per half. That is your hard deadline per block.

```c
/* the shape of the ALNK0 handler in Felucca-family firmware (GPL-3.0-only) */
void fm1_alnk0_irq(void)
{
    uint8_t p = fm1_audio_pending();
    fm1_audio_ack_aux(p);
    if (p & FM1_AUDIO_HALF) {
        uint32_t half = fm1_audio_free_half();
        render_block(&abuf[half * HALF_WORDS]);    /* must finish inside one half: 5.8 ms */
        fm1_audio_ack_half();
        if (fm1_audio_free_half() != half)
            xruns++;                               /* the DMA moved on while we rendered: a dropout */
    }
}
```

The last check is a cheap, reliable dropout counter: if the free half has changed by the time you finish, you were
late. Show it on screen during development ([Debugging](Debugging.md)).

### Bring-up

Felucca's `fm1_audio_init()` drives four GPIOs on port C (PC0, PC1, PC2, PC6) around the ALNK0 set-up, with 5 ms
waits: these switch the output stage on (PC6 low first, then high once the link runs). The exact role of each line
is not written down anywhere we know of. **[CODE] [?]**

Stop the DMA (`fm1_audio_stop()`) before a crash screen or before entering the ROM's download mode; otherwise the
last buffer loops as a buzz. Zero the buffer before flash erases, which run with interrupts off. **[CODE]**

## What is on the other end: disputed

| Source | Says | Tag |
| --- | --- | --- |
| Felucca's `hal/fm1_audio.h` | "ALNK0 I2S → external codec" | [CODE] |
| [FM-1-RE](https://github.com/AL-255/FM-1-RE/blob/main/docs/architecture.md) | "internal audio DAC", the stock firmware running its audio server at **44118 Hz** with 64-sample blocks | [RE] |

Both can be partly true (an I2S link to an on-chip or off-chip converter). The part number, whether there is a
separate codec, and the true sample rate under each clock setting are in [Open Questions](Open-Questions.md). If you know, please
add a photo or a measurement.

## Sample rate

Felucca-family firmware assumes 44,100 Hz. The stock firmware's 44,118 Hz suggests the rate is derived from a
clock that doesn't divide exactly. A measured rate (e.g. record a long sine and compare with the computer's
clock) would settle it. **[?]**

## USB audio

**[CODE]** Felucca 1.0 and fm1-base can expose the master output as a USB Audio Class 1 input (16-bit stereo,
44.1 kHz, isochronous asynchronous IN), so a computer can record the FM-1 over the cable. fm1-base also supports
the other direction (the computer plays through the FM-1). The USB packet service must run every 1 ms frame, so in
Felucca-family firmware it runs nested inside the audio render from the 10 kHz timer interrupt.

## Choosing a block size

Shorter halves give lower latency and less time per block. Longer halves give more slack against an occasional
heavy block (a note-on, a parameter change), at the cost of latency. Felucca's 256 frames (5.8 ms) is the common
choice. Whatever you pick, measure the **worst** block, not the average: see [Performance](Performance.md).

## Sources

- Felucca's `hal/fm1_audio.h`, `src/usb.c`, and the main loop of its derivatives (`main_fm1.c` in X0X, AMB-1,
  DUB-1); GPL-3.0-only.
- [FM-1-RE](https://github.com/AL-255/FM-1-RE) `docs/architecture.md` §1, §4.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Contents](../README.md)
