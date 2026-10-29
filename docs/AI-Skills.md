---
title: "AI Skills"
parent: Community
nav_order: 3
---

# AI Skills

Many FM-1 firmwares are written with an AI coding assistant. A **skill** is a folder of instructions (a
`SKILL.md` file, plain Markdown) that an assistant such as Claude Code loads when a task matches it. A good skill
saves the assistant from re-learning the FM-1 every session. A bad one spreads a wrong "fact" into every project
that uses it. This page lists the FM-1 skills we know of, what they teach, and where they disagree with this wiki.

> **Read a skill like any other source.** A skill is a summary of other people's READMEs, written (often by an AI)
> at one date. Where it states a fact, look for the fact's tag on this wiki. Where the two disagree, the point is
> in [Open Questions](Open-Questions.md). See also "Using AI assistants" in [Contributing](Contributing.md).

## mvave-fm1 (maxfridbe)

[mvave-fm1](https://github.com/maxfridbe/mvave-fm1) (GitHub user maxfridbe, MIT, first published 2026-10-22) is
two Claude Code skills and a podman build container. The r/MVaveFM1 thread
["Claude skill"](https://www.reddit.com/r/MVaveFM1/comments/1xh6o7m/claude_skill/) is, as far as we could tell,
about this repository (see Sources).

| Part | What it covers |
| --- | --- |
| `skills/mvave-fm1-firmware/` | Back up, install (ORBIT as the worked example), go back to stock, unbrick: device states and USB IDs, the SysEx update protocol, the flash map, the tools, eight hard safety rules, and "lessons from a real run" on Linux. |
| `skills/mvave-fm1-software-development/` | Write firmware: five development paths (Felucca family, FM1 Open, bare metal, binary patches of stock, modular builders), the hardware API of Dinghy's `platform.c`, the memory and flash maps, the chip's rules as Dinghy's build enforces them, build and package steps, package identities, a catalogue of ~130 firmwares by lineage, and a list of non-negotiables for shipping. |
| `docker/Containerfile`, `mvave-build.sh` | A podman image with JieLi's Linux toolchain and the three SDK files, all pinned by SHA-256, that builds Felucca-family firmwares (Dinghy, Felucca, ORBIT, SLOOP, Optimist) unchanged. See [Getting Started](Getting-Started.md). |

**How to use it.** Copy or symlink the two folders in `skills/` into `~/.claude/skills/` (or a project's
`.claude/skills/`); the assistant then loads them when you ask it to back up, flash or write FM-1 firmware. They
are also plain Markdown, so you can read them yourself. The skills' own order of work: **back up first**, then
flash, then develop.

**What it says it verified** (its README, 2026-10-22), which is a good model for any skill:
- **[HW]** on one FM-1 under Linux: the identity read, a full-flash backup read twice and compared (with
  [MvaveFM1Unbricker](https://github.com/Quixotic7/MvaveFM1Unbricker)), and an install of the ORBIT 0.7.1 release
  package, after which the unit answered `FM-1_900`.
- **[CODE]** In CI (GitHub Actions): the image built with the hash-checked toolchain, and Dinghy, Felucca and
  ORBIT built with no failing step. No container-built package had been flashed.
- **Not verified:** flashing a container-built package, rolling back from a custom firmware to stock, and the
  claims taken from other repositories' docs. The firmware catalogue comes from the
  [FM-1 resource hub](https://github.com/aroum/fm1-custom-fw) README; only the platform bases were read in depth.

**Useful things in it that this wiki did not have** are now on their pages, each with its source:
the container build ([Getting Started](Getting-Started.md)); the Linux recovery lessons and the ways into ROM boot
without hardware ([Recovery](Recovery.md)); the package identities ([Update Protocol and
Installers](Update-Protocol-and-Installers.md)); the Felucca-family flash map ([Flash Storage](Flash-Storage.md));
Dinghy's build checks ([Memory Map](Memory-Map.md), [Debugging](Debugging.md)).

### Where the skills and this wiki disagree

As of 2026-10-29. Each is in [Open Questions](Open-Questions.md).

| The skill says | This wiki says | Notes |
| --- | --- | --- |
| "no FPU", "integers only" | single-precision FPU with `-mfprev1` **[SDK] [CODE]** | The skill repeats Dinghy's `docs/RULES.md`. Dinghy's build has no float support linked; the chip's FPU is another matter. |
| SoC "AC7911B" | "AC79118B" from the first r/MVaveFM1 thread **[?]** | The hub reads the marking from board photos. |
| Flash "inside the package (Puya P25Q80H)" | a part soldered as SOIC-8 (felucca-bsp's notes on a bigger flash) | |
| OTA loader on USB `4D4A:4155`, ORBIT's included | Felucca's loader, and ORBIT's, are `1209:0002` "Felucca Update" **[CODE]** | `4D4A:4155` is from the hub's notes on M-VAVE's loader. |
| Audio "into an external codec" | disputed: external codec or internal DAC | |
| RemoteFM1 and TransitFM1 answer as `FM-1_015` | RemoteFM1 0.2 answers as `FM-1_52XXYYZZ` (its README, 2026-10-28) | the skill was written at 0.1; projects move fast |

None of these is a reason to avoid the skill. They are the reason to keep the tags.

## Other skills and agent kits

| Project | Licence | What |
| --- | --- | --- |
| [fm1-base](https://github.com/charlesvestal/fm1-base) `felucca-to-fm1-base` | MPL-2.0 | Moves a Felucca-based firmware onto fm1-base, keeping its GPL ([Choosing a Base](Choosing-a-Base.md)). |
| [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) | per its README | A prompt and kit for porting an FM-1 firmware to the SMK-37 with a coding agent ([SMK-37](SMK-37.md)). |
| [Felucca Editor](https://github.com/hunterxiong2026/fm1-felucca-editor-2026) | MIT | Edits Felucca 1.5 sounds, tracks and songs over SysEx; also packaged as a Claude skill. Chinese README. |
| [SLOOP Editor Skill](https://github.com/hunterxiong2026/fm1-sloop-editor-2026) | MIT | Drives SLOOP 2.4.1 over SysEx (a DeepSeek harness skill). Chinese README. |
| Dinghy's [docs/RULES.md](https://github.com/zvenson/dinghy/blob/main/docs/RULES.md) | GPL-3.0 | Not a skill, but written to be read by agents and people; the source of most of mvave-fm1's "chip rules". |

(Descriptions from each README and [awesome-fm-1](https://github.com/cicloid/awesome-fm-1), October 2026.)

## Writing a skill for the FM-1

What the good ones do:
- **Say what was verified and how**, and what was only read. mvave-fm1's "Status" section and ORBIT's release
  notes are examples.
- **Link sources** for each fact, and date them.
- **Put safety first**: a verified full-flash backup before any write; never write below `0x4000`; keep an update
  path in every firmware ([Recovery](Recovery.md)).
- **Point at this wiki's tags** rather than copying tables that will go stale.

## Sources

- [mvave-fm1](https://github.com/maxfridbe/mvave-fm1) README, `readme/build-container.md`, both `SKILL.md` files
  (MIT, © 2026 maxfridbe), commit `5607a60` (2026-10-22).
- [awesome-fm-1](https://github.com/cicloid/awesome-fm-1) (CC0), "Tools" and "Editors" entries.
- [RemoteFM1 README](https://github.com/Quixotic7/RemoteFM1) (0.2.3, 2026-10-28).
- [Dinghy docs/RULES.md](https://github.com/zvenson/dinghy/blob/main/docs/RULES.md).
- The r/MVaveFM1 thread ["Claude skill"](https://www.reddit.com/r/MVaveFM1/comments/1xh6o7m/claude_skill/): its
  text could not be fetched when this page was written; the repository above is the project the thread is about
  as far as we could tell. If the thread adds facts, add them here with the posters' usernames.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
