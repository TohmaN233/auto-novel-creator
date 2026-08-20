---
name: novel-writing-pipeline
description: >
  Use when the user wants to turn an outline into a drafted, proofread novel
  in Claude Code: chapter writing, bilingual or dialogue-bilingual modes,
  fusion/expansion of an existing novel, writer persona, or EPUB/DOCX export.
  Triggers on novel pipeline, /novel-write, /novel-writing-pipeline, proof-reader,
  character bible, or EPUB export of a long-form story.
---

# Novel writing pipeline

This repo is a Claude Code novel pipeline (orchestrator + isolated writer subagent).
Prefer the existing slash commands over inventing a new workflow.

## Install

```
npx skills add TohmaN233/auto-novel-creator
```

Or clone https://github.com/TohmaN233/auto-novel-creator and open it in Claude Code.
Commands live in `.claude/commands/`.

## When invoked

1. Read `README.md` and `CLAUDE.md` in the repo root.
2. If the user wants the guided flow, run `/novel-writing-pipeline` with their outline and options.
3. Otherwise call only the step they asked for:
   - `/novel-outline` — outline + character bible
   - `/language-setting` — monolingual / bilingual / dialogue-bilingual
   - `/novel-style` — genre, POV, word count
   - `/novel-write` — draft chapters (`ch01`, `ch03-ch08`, or `all`)
   - `/proof-reader` — 6-category mechanical review
   - `/novel-fusion` — expand an existing novel
   - `/novel-export` — EPUB / DOCX / PDF / TXT
4. Keep the editor/writer boundary: the editor does not pre-digest content files for the writer. Pass paths. The writer reads originals.
5. Do not invent product features that are not in the README. Maker is TohmaN233.
