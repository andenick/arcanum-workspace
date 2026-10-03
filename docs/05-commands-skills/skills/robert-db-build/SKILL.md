---
name: robert-db-build
version: "1.0"
description: "Orchestrator for the Robert Database Framework: runs init → harvest → enrich → organize → audit → publish in order for a project, resumable from BUILD_STATE.json, with continuousness-first cadence and explicit yield triggers."
when-to-use: '"User wants to build a project''s Robert database end-to-end, run the full pipeline, or resume an interrupted build."'
search-hints: "robert db build orchestrator pipeline end-to-end stages resume BUILD_STATE yield triggers continuous init harvest enrich organize audit publish"
argument-hint: "[project]"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit, Agent
requires: robert-db-init
part-of: Robert Database Framework v1.0
---

# robert-db-build — Orchestrator (all stages)

Drive a project from a config to a published database by running the six stages in
order, recording progress to `BUILD_STATE.json`, and resuming cleanly after any
interruption. This is the default entry point for "build the Robert database for
project P".

## Purpose

Sequence INIT → HARVEST → ENRICH → ORGANIZE → AUDIT → PUBLISH, delegating each stage
to its sub-skill, with canonical on-disk state so a forced session boundary never
loses progress.

## Preconditions

- `robertdb_config.json` exists and validates (see `robert-db-init`). The
  orchestrator runs init itself if the DB is absent.

## Stage order and semantics

| # | Stage | Sub-skill | Gate / note |
|---|---|---|---|
| 1 | INIT | robert-db-init | scaffold + DB (skipped if DB present + stamped) |
| 2 | HARVEST | robert-db-harvest | chains quality + views |
| 3 | ENRICH | robert-db-enrich | agent batches → patches → merge; multi-round |
| 4 | ORGANIZE | robert-db-organize | **pauses for user taxonomy ratification** |
| 5 | AUDIT | robert-db-audit | `--gate` must exit 0 before stage 6 |
| 6 | PUBLISH | robert-db-publish | gated on stage 5; mirrors with grep-verify |

ENRICH and ORGANIZE both depend only on HARVEST; the orchestrator runs ENRICH then
ORGANIZE, but they may be interleaved. AUDIT strictly gates PUBLISH.

## Procedure

1. **Load state.** Read `Technical/RobertDB/state/BUILD_STATE.json`
   (`load_build_state` seeds it if absent). It records per-stage status under
   `stages.{init,harvest,enrich,organize,audit,publish}`.
2. **Run stages in order**, invoking each sub-skill. After each stage completes,
   write its status (`done` + run_id + timestamp) into BUILD_STATE via
   `save_build_state` (atomic). For ENRICH, drive **many rounds inline per turn**
   per `orchestration-cadence.md` — never `/loop` once per round.
3. **Ratification pause (ORGANIZE).** When the taxonomy proposal is ready, stop and
   present it to the user. Do not classify or advance to AUDIT until the user
   ratifies. Record the pause in BUILD_STATE so a resume re-enters at ratification.
4. **Gate before PUBLISH.** Run AUDIT `--gate`; advance to PUBLISH only on exit 0
   (or explicit user `--allow-warn`).
5. **Resume.** On re-entry with `--resume`, read BUILD_STATE and skip stages marked
   `done`; re-enter the first incomplete stage (for ENRICH, resume the queue from
   the DB — pending Tier A/B tables — not from conversation).

## Yield triggers (per orchestration-cadence)

Stay continuous and in-turn. End the turn / hand back ONLY when:
1. **Disk guard trips** (system drive < 20 GB free) — pause and report.
2. **A user-blocking decision is required** — e.g. taxonomy ratification, an
   ambiguous scope, or a publish leak that needs a license/scope call.
3. **The build is done** — published and mirror grep-verified.
4. **Context is about to overflow** and checkpointing now (BUILD_STATE is canonical)
   loses less than after a forced compaction.
5. **A hard rate/billing wall** that 30s→5min retry did not clear.

Transient errors (API overload, socket close, 1M-context billing gate, timeout) are
**not** yield triggers — retry in-loop. Never fabricate content on a transient.

## Outputs

- A fully built, audited, and (if gated green) published project database.
- `state/BUILD_STATE.json` reflecting per-stage completion (resumable).
- Per-stage run-ledger rows; the published bundle + verified mirror.

## Failure handling

- **A stage fails** → record the failure in BUILD_STATE, fix at the stage's
  sub-skill, re-run `robert-db-build --resume`. Do not skip a failed stage.
- **AUDIT gate red** → loop back to ENRICH/ORGANIZE to fix the flagged items; the
  orchestrator will not advance to PUBLISH on a red gate.
- **Resume ambiguity** → BUILD_STATE on disk is canonical; trust it over
  conversation history after any compaction.
