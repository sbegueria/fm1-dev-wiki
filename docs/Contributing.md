---
title: "Contributing"
parent: Community
nav_order: 6
---

# Contributing

Anyone can improve this wiki. The rules exist for one reason: **a wrong fact, once written down, gets copied.**
The r/MVaveFM1 thread that started this wiki put it this way: a small hallucination or oversight leads to a
workaround that didn't need to be there, and "once it's there in code, comments, markdown, etc. it becomes almost
impossible for future agents to question it".

## The rules

1. **Every technical fact has a source and a tag** ([HW], [SDK], [RE], [CODE], [?]; see [Home](Home.md)).
   - [HW]: say who measured it, on what (FM-1 or SMK-37, which bootloader, which firmware), and when.
   - [CODE]: link the file and, ideally, the commit.
   - [SDK]: give the SDK path.
   - Can't source it? Tag it [?] and add it to [Open Questions](Open-Questions.md).
2. **Don't upgrade a tag without evidence.** "Everyone's code does it" is [CODE], not [HW].
3. **Disagreements stay visible.** If your source contradicts the page, add yours beside it and open an entry in
   [Open Questions](Open-Questions.md). Don't delete the other.
4. **Write the date** on measurements and on anything about a project's status. Projects move fast.
5. **Plain English, short sentences.** Many readers use translation tools. Define terms in [Glossary](Glossary.md).
6. **No secrets, no binaries.** Don't paste M-VAVE's or JieLi's binaries or large parts of their code. Short
   register facts, addresses and protocol descriptions are fine.
7. **Credit people and projects.** Link the repository, issue or post.
8. **Unofficial and neutral.** No logos. Don't speak for M-VAVE or JieLi.

## Using AI assistants

AI tools write a lot of FM-1 code. They are welcome here too, with these extra rules:

- **An AI's claim is [?] until someone checks it** on hardware, in the SDK, or in working code.
- Say in your edit summary that an AI helped, and which parts you checked yourself.
- Don't let an assistant "fill in" a table. Missing data stays missing.
- When you point an assistant at this wiki, point it at the tags too: [?] facts are not facts.

## Code excerpts

- Keep excerpts short: the lines that show the point.
- Say where each excerpt comes from and its licence, in a comment or the line above it.
- Your own new snippets are under the wiki's licence ([License](License.md)); mark them "a sketch".

## Page layout

Each technical page ends with a **Sources** section. Use `[Page Name](Page-Name.md)` for links inside the wiki. Use tables
for registers and pins: address, value, meaning, tag.

## How to edit

The pages are Markdown files in `docs/`. Fork the repository, edit, and open a pull request: each change is
reviewed before it's merged, so a second pair of eyes checks every new fact. Small fixes can be edited in the
browser (the pencil icon makes the fork and the pull request for you). New pages or big changes: open an issue
first, so others can comment.

---

Unofficial community wiki. Not affiliated with M-VAVE or JieLi. Text under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/); code excerpts keep their own licences ([License](License.md)). Facts are tagged **[HW] [SDK] [RE] [CODE] [?]**: see [Home](Home.md). Found an error? See [Contributing](Contributing.md). [Home](Home.md)
