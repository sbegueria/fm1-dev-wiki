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

## mvave-fm1

The thread ["Claude skill"](https://www.reddit.com/r/MVaveFM1/comments/1xh6o7m/claude_skill/), posted on
r/MVaveFM1 by u/LordApophis99 (October 2026): "Did my best to make you guys a claude skill for
interacting/building firmwares". It links the repo
[github.com/maxfridbe/mvave-fm1](https://github.com/maxfridbe/mvave-fm1) (MIT, first published 2026-10-22): two
Claude Code skills and a podman build container.

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

### Suggestions from the thread

- **Add fm1-base.** A commenter suggested adding [fm1-base](https://github.com/charlesvestal/fm1-base) to the
  development skill, as a base with "BLE, WIFI, Velocity, USB Audio, Dual Core, USB Midi, Aftertouch", "much
  better than using Felucca as base", but not GPL, so GPL engines from other firmwares can't easily be brought in.
  As of commit `5607a60` the skill's five paths don't include it. What fm1-base offers and what its licence
  allows in each direction: [fm1-base as a Base](FM1-Base.md).
- **Make the firmware skill generic.** The same commenter noted that `mvave-fm1-firmware` is written around ORBIT
  (its install example, rescue keys, identities and file formats). Most of it holds for any Felucca-family
  firmware (the update protocol, MvaveFM1Unbricker, the safety rules), but the identity `FM-1_900`, the loader name
  `ota-FM-1_9XX`, OCT− rescue and ORP7/OPB2 files are ORBIT's. If you use it for another firmware, swap those for
  your firmware's ([Update Protocol and Installers](Update-Protocol-and-Installers.md), "Package identities").
- u/kmplngj replied that he would check what he could use, and pointed to the skills and tooling in his Felucca
  fork (below).

## Other skills and agent kits

| Project | Licence | What |
| --- | --- | --- |
| [fm1-base](https://github.com/charlesvestal/fm1-base) `felucca-to-fm1-base` | MPL-2.0 | Moves a Felucca-based firmware onto fm1-base, keeping its GPL ([Choosing a Base](Choosing-a-Base.md)). |
| [fm1-to-smk37](https://github.com/genkigenki/fm1-to-smk37) | per its README | A prompt and kit for porting an FM-1 firmware to the SMK-37 with a coding agent ([SMK-37](SMK-37.md)). |
| [Felucca Editor](https://github.com/hunterxiong2026/fm1-felucca-editor-2026) | MIT | Edits Felucca 1.5 sounds, tracks and songs over SysEx; also packaged as a Claude skill. Chinese README. |
| [SLOOP Editor Skill](https://github.com/hunterxiong2026/fm1-sloop-editor-2026) | MIT | Drives SLOOP 2.4.1 over SysEx (a DeepSeek harness skill). Chinese README. |
| [kmplngj/Felucca](https://github.com/kmplngj/Felucca) `.agents/skills/` | GPL-3.0-only | A Felucca fork run largely by agents (below). |
| Dinghy's [docs/RULES.md](https://github.com/zvenson/dinghy/blob/main/docs/RULES.md) | GPL-3.0 | Not a skill, but written to be read by agents and people; the source of most of mvave-fm1's "chip rules". |

(Descriptions from each README and [awesome-fm-1](https://github.com/cicloid/awesome-fm-1), October 2026.)

## kmplngj's Felucca fork: skills and tooling

[kmplngj/Felucca](https://github.com/kmplngj/Felucca) (Jan Kampling, GPL-3.0-only) is a personal fork of
Felucca that collects features from upstream and other forks and works on CPU, RAM and size. Its agent set-up is
the most complete public one for FM-1 work we know of (commit `c543e24`, 2026-10-29) **[CODE]**:

- **One set of instructions for every agent.** `AGENTS.md` holds the commands, the architecture, the invariants
  ("three builds": firmware, host tests and browser emulator compile the same sources; every resource budget;
  older saved data still loads) and a table "before you … read …". `CLAUDE.md` only imports it, so Claude Code and
  Codex read the same file. Skills live in `.agents/skills/`.
- **Six skills:** `issue-delivery` (an issue or PR through validation, review and hand-off, or an explicitly
  requested backlog), `full-stop` and `full-send` (the same, ending at a verified PR, or merging once every gate
  passes), `pr-testing` (writing FM-1 hardware test steps and expected results for the user), `firmware-watch`
  (reviewing other firmwares' commits for ideas worth adopting, with a JSON/JSONL ledger of what was assessed) and
  `sol-loop` (a bounded code review by another model). `docs/ai-workflows.md` explains them for other projects.
- **No merge without hardware.** Tagged releases need an FM-1 test report and the maintainer's approval; agents
  write the test steps, the owner runs them.
- **Measurement tools** (in `tools/` and `tests/`): a read-only device diagnostics collector over the USB serial
  console (`tools/fm1_diagnostics.py`; boot clock registers, crash context, render-stage timing as opt-in build
  flags), an offline boot-area identifier for dumps and packages (`tools/fm1_bootarea.py`: M-VAVE V15/V16 or
  m-bootloader, CRCs, chip key, the clock declared in `isd_config.ini`), a static scan of the disassembly for the
  64-bit-multiply erratum pattern (`tests/mul64_hload_check.py`), golden-hash and CPU-baseline regression tests,
  a per-function target budget check on the disassembly (`tests/target_budget.py`), and screenshots over USB.
- **Dated investigation notes** in `docs/` (audio deadline misses on note attacks, the second core's idle polling,
  display power), each with the exact images tested. The hardware findings are on [Second Core](Second-Core.md),
  [Display](Display.md) and [Clocks and Power](Clocks-and-Power.md).

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
- The r/MVaveFM1 thread ["Claude skill"](https://www.reddit.com/r/MVaveFM1/comments/1xh6o7m/claude_skill/),
  posted by u/LordApophis99; a commenter's reply (name not recorded here); u/kmplngj's reply.
- [kmplngj/Felucca](https://github.com/kmplngj/Felucca) `AGENTS.md`, `CLAUDE.md`, `.agents/skills/*/SKILL.md`,
  `docs/ai-workflows.md`, `docs/diagnostics.md`, `docs/boot-area-survey.md`, `docs/mul64-hload-audit.md`
  (GPL-3.0-only; commit `c543e24`, 2026-10-29).

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
