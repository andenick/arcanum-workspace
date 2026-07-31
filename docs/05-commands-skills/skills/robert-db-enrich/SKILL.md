---
name: robert-db-enrich
version: "1.0"
description: "Agent-orchestrated recovery of honest table metadata (titles, pages, units, footnotes, two-axis quality) for a Robert database: doc-grouped Opus subagent batches emit JSONL patches per the patch contract; rdb_merge_patches.py validates and merges. Subagents never touch the DB."
when-to-use: '"User wants to enrich a project''s harvested tables with recovered metadata, run enrichment batches, or merge enrichment patches into the Robert database."'
search-hints: "robert db enrich enrichment patch jsonl subagent opus batch doc-grouped merge_patches field_basis not_captured obs_status transcription_status 32k cap chunk-range"
argument-hint: "[project]"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit, Agent
requires: robert-db-harvest
part-of: Robert Database Framework v1.0
---

# robert-db-enrich — Stage 3 (ENRICH)

Recover the metadata mechanical harvest could not: real titles, page refs, units,
footnotes, period/geography, and the two-axis quality assessment. This is an
**agent-orchestration skill**. Subagents read source context and emit JSONL patch
files; **only `rdb_merge_patches.py` writes the DB**. Subagents NEVER touch
`robertdb.sqlite`.

## Honesty contract (non-negotiable)

> **NOT CAPTURED is a first-class value. The source page is the authority.**
> A field is `from_source` ONLY if it is visible verbatim in the table's CSV,
> its `Tables/*.md`, or the `FULL_TEXT`/`Text` context for that chunk. A title,
> footnote, or unit that is not visibly present is either `agent_inferred`
> (defensible from context) or sent as `null` with basis `not_captured`. Never
> fabricate titles, footnotes, quotes, or page numbers. `from_source` is never
> downgraded by `agent_inferred` (the merge enforces this).

## Preconditions

- `robert-db-harvest` has run; `xtables` populated with mechanical provenance.
- Tier policy assigned (Tier A/B from `config.tiers`). Enrich prioritizes Tier A,
  then B; Tier C is enriched opportunistically or skipped per scope.

## Verification pass vs recovery pass (HDARP v6.3)

If the KB was extracted under HDARP v6.3, harvest already ingested the native
`RDB_METADATA*.jsonl` sidecars and set those tables to `enrichment_status='enriched'`
with real `from_source`/`agent_inferred` provenance. **ENRICH is then a thin
verification pass:** the pending queue below naturally targets only `pending` /
`mechanical_only` rows (legacy KBs, `fullread` output, `enrichhdarp` gaps), so
already-enriched tables are skipped; lift a sample to `transcription_status='V'`
**only** through the formal `robert-db-audit` spot-check, never here. For a legacy KB
(no sidecars) this is the full recovery pass exactly as written below.

## Optional: LOCAL-MODEL triage (cut Opus tokens — default OFF)

If `kb.enrich_triage.mode` is `draft` in `robertdb_config.json`, run the A4
local-model triage **before** spawning Opus subagents: the ALGRP A4 champion
(gemma-4-31B, GBNF-constrained) drafts each pending table's metadata on the 5090;
a confidence gate auto-commits the high-confidence grounded majority as JSONL
patches (merged by `rdb_merge_patches.py` unchanged) and writes only the
low-confidence tables to `enrichment/triage/<run_id>/review_queue.jsonl` — and
THOSE are the only tables you then hand to Opus subagents below. Harness:
`scripts/triage/rdb_triage_route.py`; full wiring + GPU smoke test (USER runs the
server, single-launch discipline): `A4_TRIAGE_HANDOFF.md`.
Default is **off** (this whole section is skipped) — the Opus-subagent path below
is the default and is unchanged.

## Procedure (doc-grouped, continuous, multi-round)

Follow `orchestration-cadence.md`: process **many rounds inline per
turn**. Do NOT `/loop` once per round and do NOT return control after a single
batch — select work, spawn agents, merge, select the next round, repeat in-turn.

1. **Build the queue from the DB.** Select Tier A/B documents that have pending
   tables:
   ```bash
   PYTHONIOENCODING=utf-8 python -c "import sqlite3,sys; c=sqlite3.connect(sys.argv[1]); \
[print(r[0],r[1]) for r in c.execute(\"select d.doc_id,count(*) n from xtables x join documents d on d.doc_id=x.doc_id \
where x.enrichment_tier in ('A','B') and x.enrichment_status='pending' and x.is_marker_file=0 \
group by d.doc_id order by n desc\").fetchall()]" \
       <P>/robertdb.sqlite
   ```
   Choose a `run_id` for the round, e.g. `ENR_20260611_A01`.

2. **Spawn Opus subagents in FOREGROUND, multiple per message.** Each subagent
   gets **ONE document** (or, for table-dense yearbooks, ONE chunk-range of one
   document — split by chunk-range to respect the 32K output cap, never by
   trimming fields). Each subagent:
   - reads the document's table CSVs (`CSV_Tables/`), the `Tables/*.md` (USSR
     layout) and the `FULL_TEXT*.md` / `Text/` chunk context;
   - recovers metadata it can honestly assess for each table;
   - emits **ONE JSONL patch file** to
     `<doc_id>.jsonl`
     (one line per table) per `PATCH_CONTRACT.md`;
   - **does not open the database**.

3. **Merge the round.**
   ```bash
   PYTHONIOENCODING=utf-8 python rdb_merge_patches.py \
       --config <P>/robertdb_config.json \
       --run-id <run_id> [--dry-run]
   ```
   Run `--dry-run` first to surface rejects, then merge for real. The merger
   enforces the patch contract: `table_uid` must exist, `doc_id` must match,
   `field_basis` must cover every metadata field present, no-downgrade is enforced,
   `null`+`not_captured` writes an honest negative, and on success
   `enrichment_status='enriched'`. **Rejected lines go to
   `patches/<run_id>/_rejected.jsonl` with a reason — never silently dropped.**
   Read the rejects, fix the agent's output, re-emit, re-merge (merges are
   idempotent).

4. **Next round, in-turn.** Re-run the queue query, spawn the next batch of
   subagents, merge. Continue until Tier A/B pending is drained or a yield trigger
   fires (disk guard, user-blocking ambiguity, scope done, context overflow).

5. **Recompute quality** (the merger updates rows; if you changed weights or want a
   full pass): `rdb_quality.py --config CFG [--recompute-all]`, then `rdb_views.py`.

## Subagent prompt TEMPLATE (use verbatim, fill the placeholders)

```
You are enriching table metadata for the Robert Database Framework, document
<DOC_ID> in project <PROJECT> (code <PJ>). Model: Opus. Run id: <RUN_ID>.

INPUTS (read-only — read all that exist for this document):
  - Tables CSVs:   <PROJECT_ROOT>/<DOC_ID>/CSV_Tables/*.csv
  - Table notes:   <PROJECT_ROOT>/<DOC_ID>/Tables/*.md      (if present)
  - Body context:  <PROJECT_ROOT>/<DOC_ID>/FULL_TEXT*.md
                   or .../Text/*.md   (find the chunk that contains each table)
TABLE LIST (table_uid -> source CSV relpath), enrich EXACTLY these:
  <TABLE_UID>  <SOURCE_RELPATH>
  ... (one per line; for a chunk-range assignment, only the listed uids)

YOUR JOB: for each listed table, recover only what you can honestly assess, and
write ONE JSON object per line to:
  <PROJECT_ROOT>/<RUN_ID>/<DOC_ID>.jsonl

HONESTY RULES (HARD):
  - NOT CAPTURED is a first-class value. The source page is the authority.
  - A field is "from_source" ONLY if it is visible verbatim in this table's CSV,
    its Tables/*.md, or the FULL_TEXT/Text context for its chunk. Quote-check
    yourself before claiming from_source.
  - Do NOT fabricate titles, footnotes, source notes, page numbers, or quotes.
    If a value is not visibly present, either give a defensible "agent_inferred"
    value (e.g. an English translation of a visible original title) OR send the
    field as null with basis "not_captured". When in doubt, not_captured.
  - "from_source" must never be claimed for a value you reasoned to. Translations,
    transliterations, and guessed units are "agent_inferred".

PER-LINE RECORD (see PATCH_CONTRACT.md):
  - REQUIRED: table_uid, doc_id (== <DOC_ID>), run_id (== <RUN_ID>).
  - Include ONLY metadata fields you actually assessed. null = assessed and not
    recoverable (=> basis not_captured); absent = not assessed (untouched).
  - REQUIRED field_basis object covering EVERY metadata field you included, each
    value in {mechanical, from_source, agent_inferred, not_captured, unknown}.
  - Optional but encouraged two-axis quality:
      obs_status: SDMX CL_OBS_STATUS dominant code for the table
                  (A normal | B break | D dubious | E estimated | P provisional |
                   U low-reliability | I imputed | O missing | M not-applicable |
                   L missing-data-exists). Assess the SOURCE data.
      transcription_status: your READ fidelity vs the source. Emit ONE of
                  H high-confidence | L low-confidence | R reconstructed | X illegible.
                  ASSESS per table — do NOT default everything to L: H = the digitized
                  cells confidently match the page; L = uncertain (faint scan, ambiguous
                  digits); R = reconstructed from a damaged region; X = illegible. Do NOT
                  emit V — "V" (verified) is reserved for the formal audit spot-check.
  - Optional: columns[], taxonomy_suggestions[], concordance_hint,
      quality_observations[], enrichment_confidence (high|medium|low).

SIZE: one line per table; keep footnotes <= ~2000 chars (truncate with
"…[truncated]"). Do NOT open robertdb.sqlite. Emit the JSONL file and stop.
```

## Outputs

- `enrichment/patches/<run_id>/<doc_id>.jsonl` (agent output) + `_rejected.jsonl`.
- Updated `xtables` metadata + `field_provenance` (honoring no-downgrade) +
  `obs_status`/`transcription_status`; `enrichment_status='enriched'`.
- An `enrich_merge` run-ledger row; views regenerated.

## Failure handling

- **Rejected lines** → read `_rejected.jsonl`, fix the offending field/basis, re-emit
  that document's JSONL, re-merge (idempotent). Never edit the DB by hand to "force"
  a rejected field.
- **Table-dense doc blows the 32K cap** → split the assignment by chunk-range across
  multiple subagents; do not trim fields or summarize tables to fit.
- **Agent claimed `from_source` for a reasoned value** → reject in review, correct to
  `agent_inferred` or `not_captured`. This is the single most important check.
- **Transient agent/API errors** (overload, socket close) → retry the subagent;
  these are not data problems and never justify fabricated content.
