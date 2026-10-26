# FM-1 Firmware Developers' Wiki

An unofficial, community-written knowledge base for people who write their own firmware for the
**M-VAVE FM-1** pocket synthesizer, and for its sibling, the **SMK-37 Pro / Elite** keyboard. Both are built
around a JieLi AC791N (WL82 family) chip.

> **Unofficial.** This wiki is not affiliated with, endorsed by or supported by M-VAVE or JieLi. "M-VAVE",
> "FM-1" and "SMK-37" are names of their owners, used here only to say which hardware we mean.
> **Installing firmware is at your own risk.** Read [[Recovery]] before you install anything you built yourself.

## Who this is for

- You want to write or change firmware for the FM-1, and want to know what the hardware really does.
- You already have a firmware, and want to know why it drops out, crashes, or installs badly.
- You use an AI assistant to write code, and want facts that you (and it) can check.

You don't need to be an embedded developer. You do need to read the sources each page cites.

## Why a wiki

The FM-1 scene grew very fast in 2026: dozens of firmwares, several bases, many tools. Much of what people know
lives in READMEs, code comments, issues and Reddit threads. Some of it has been copied from project to project
without anyone checking it again. A wrong "fact" in a comment can end up shaping a whole design.

This wiki tries to collect what is known, **say where each fact comes from, and say how sure we are.**
See [[Contributing]] for the rules.

## How sure we are: the tags

Every fact on a technical page carries one of these tags:

| Tag | Meaning |
| --- | --- |
| **[HW]** | Measured or seen on a real unit. The page says by whom, or which project. |
| **[SDK]** | Read in JieLi's AC79 SDK (headers, linker scripts, documentation). True for the chip family; not always checked on an FM-1. |
| **[RE]** | From reverse engineering of M-VAVE's stock firmware or tools. |
| **[CODE]** | Used by working community firmware, so it works in practice, but nobody wrote down a measurement. |
| **[?]** | Reported, guessed or disputed. Listed in [[Open Questions]]. |

When two sources disagree, both are shown and the point goes to [[Open Questions]].

## Start here

1. [[Getting Started]]: the toolchain, the SDK files, a first build, a first install.
2. [[Choosing a Base]]: which existing code to start from.
3. [[Recovery]]: how to get a unit back when an install goes wrong. **Read it before your first install.**
4. [[Hardware Overview]]: what is in the box.
5. [[Programming Guide]]: the audio deadline, interrupts, the screen, and the traps.

## All pages

**Hardware:** [[Hardware Overview]] · [[Memory Map]] · [[Clocks and Power]] · [[Pinout]] · [[Audio Output]] ·
[[Display]] · [[Controls]] · [[MIDI and USB]] · [[SMK-37]]

**Boot, packages and installing:** [[Boot Chain and Bootloaders]] · [[Firmware Package Format]] ·
[[Update Protocol and Installers]] · [[Recovery]] · [[Flash Storage]]

**Writing firmware:** [[Programming Guide]] · [[Performance]] · [[Second Core]] · [[Debugging]] ·
[[Simulators and Emulators]]

**Community:** [[Known Projects]] · [[Tools]] · [[Open Questions]] · [[Glossary]] · [[Contributing]] · [[License]]

## Related lists

This wiki is about *how things work*. For a list of every firmware, editor and tool, see
[awesome-fm-1](https://github.com/cicloid/awesome-fm-1) (CC0) and the
[FM-1 resource hub](https://github.com/aroum/fm1-custom-fw). We link to them rather than copy them.

## How this wiki started

It grew out of a post on r/MVaveFM1,
["All right, stop, collaborate and listen"](https://www.reddit.com/r/MVaveFM1/comments/1xe8c9x/all_right_stop_collaborate_and_listen/)
(October 2026), which asked for a shared place for FM-1 firmware knowledge, so people stop repeating the same
mistakes. Much of the first content comes from that thread, from the projects it links, and from the notes of
the firmwares listed in [[Known Projects]].
