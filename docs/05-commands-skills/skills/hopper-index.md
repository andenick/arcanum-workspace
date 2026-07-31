---
name: hopper-index
version: "1.0"
description: "Pointer skill — read INDEX.md to become conversant in every Hopper protocol, the V2 multi-model renamer, the unified rename ledger, operational discipline rules, and the Hopper-vs-HDARP boundary in ~5 minutes. Use when invoked on any Hopper-related task with no prior context."
when-to-use: "User starts a session with a Hopper-adjacent task (extract / rename / coordinate / debug GPU coexistence / audit naming actions). Read INDEX.md first so the rest of the session is informed."
search-hints: "hopper index orientation conversant pdf naming rename multi-model consensus unified ledger v2 hl2 onboarding"
allowed-tools: Read, Glob, Grep
requires: ""
part-of: "Hopper Line v2 (HL2) — Offline Local VLM Extraction"
---

# /hopper-index — Become conversant in Hopper in 5 minutes

This skill is a single redirect: **read `INDEX.md`** before doing any Hopper-related work.

`INDEX.md` is structured for fast read:

1. **TL;DR** — what Hopper IS and IS NOT (hard naming rule: Hopper ≠ HDARP)
2. **Decision tree** — user task → which skill
3. **File map** — every Hopper-related artifact (~60 files) categorized
4. **Frozen champions matrix** — engine OCR tier + renamer compose tier
5. **Operational discipline** — single-launch, GPU coexistence, port discipline (8088 vs 8090), naming
6. **Live state pointers** — work queue, build ledger, unified rename ledger
7. **Conversant-agent knowledge graph** — "if user asks X, read Y first" table
8. **Quick-start invocations** — copy-paste preflight + commands

After reading INDEX.md, you'll know which other skill to invoke and which files to read for the user's specific task.

## Invocation

```
/hopper-index
```

(No arguments. Just opens the index.)

## What it does

```python
# Effectively just:
Read("INDEX.md")
# Then summarize relevant sections for the task at hand.
```

## When NOT to use

- If the user has already specified a concrete Hopper action (e.g. "extract this PDF") → go straight to `/hopper` or `/pdf-naming-protocol`. INDEX.md is for orientation, not execution.
- If you're already deep in a Hopper session with context loaded — skip the index and use the relevant skill directly.

## Related

- `/hopper` — engine front door (execute extraction)
- `/pdf-naming-protocol` — multi-model renamer
- `/handoff_preparation` — session closeout
- All `/hdarp-*` skills — DISTINCT cloud Claude Read-tool pipeline (NEVER conflate with Hopper)
