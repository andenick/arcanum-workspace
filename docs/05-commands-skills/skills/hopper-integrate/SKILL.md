---
name: hopper-integrate
description: "Land + tag + integrate a finished Hopper Line v2 KB into the project KB + Robert ledgers + robert-db, method-tagged. Use when a Hopper extraction is done (on the floor at {hopper}/<doc> or already at <P>/Knowledge_Base) and you need to LAND it, TAG it read_method=Hopper, CATALOG it (KBIP v1.0), wire the unified ledgers, build a dated safe-package, and hand off to /kb-integrate-pipeline + robert-db-build. Does NOT extract or use the GPU."
when-to-use: '"User finished a Hopper Line v2 extraction and wants it integrated into the project Knowledge_Base, the unified Robert ledgers, and robert-db — method-tagged as Hopper, not HDARP."'
search-hints: "hopper integrate on-ramp land tag catalog ledger safe-package KBIP read_method Hopper Line v2 robert-db Jane"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[project] [--dry-run] [--apply] [--kb-root DIR] [--zip]"
version: 1.0
part-of: KB Integration Pipeline (KBIP) v1.0
requires: hopper
---

# /hopper-integrate — Hopper KB on-ramp v1.0

> **Sibling of `/kb-integrate-pipeline`** (the W4-renamed, method-agnostic pipeline). This is the
> **Hopper on-ramp** that prepares a finished Hopper Line v2 (HL2) KB so the shared pipeline can run.
> It is **lighter** than `/hdarp-integrate-pipeline` because Hopper already emits the 4 KB artifacts
> (body text + tables + equations + figures) at extraction time — this on-ramp just **LANDs, TAGs,
> CATALOGs, wires LEDGERs, builds a SAFE-PACKAGE, and hands off**. It never re-extracts.
> Canonical standard: `KB_INTEGRATION_PIPELINE_BUILD_PLAN.md` (§4 W3).

## When to use

You have a finished **Hopper Line v2** extraction (4-artifact + Anu-ready output, produced by `/hopper`)
and you want it folded into the project's `Knowledge_Base`, the unified Robert ledgers, and the per-project
Robert database — **method-tagged as `Hopper`**, joinable by `source_md5` alongside HDARP docs in the same
KB. The proof run is **Jane** (305 docs).

```bash
/hopper-integrate Jane --dry-run     # ACCEPTANCE: list every doc + planned artifacts, write NOTHING
/hopper-integrate Jane               # same as --dry-run (DRY-RUN IS THE DEFAULT)
/hopper-integrate Jane --apply       # actually TAG + LEDGER-write + SAFE-PACKAGE, then hand off
/hopper-integrate Jane --apply --zip # ...and write the dated KB_HOPPER_Jane_{date}.zip backup
```

**`--dry-run` is the default.** Nothing outside the project KB + ledgers is ever touched, and in dry-run
mode nothing is written at all — it only lists each doc and the artifacts it *would* produce. You must pass
`--apply` to write.

## Shared KBIP v1.0 contracts (same as `/kb-integrate-pipeline`)

This on-ramp and the pipeline share one read-method spine — **one key, three names, same value, joined by
`source_md5`** (build plan §2):

- `READ_METHOD.json` (schema v1.0, §3.3) — per-doc self-declaration: `read_method:"Hopper"`,
  `read_engine:"Hopper Line v2"`, `read_version:"v1.0"`, `models_used`, `source_md5`, `native_enrichment`.
- `manifest.json.read_method` (top-level string) + a `<!-- read_method: Hopper Line v2 v1.0 -->` header in
  each `FULL_TEXT_chunks_*.md`.
- `RDB_METADATA.jsonl` — native-enrichment sidecar, one line per table CSV (NATIVE_ENRICHMENT_CONTRACT;
  `transcription_status` derived mechanically from Hopper confidence, `obs_status:null`, every present field
  carries a `field_basis`).
- KBIP catalog schema v1.0 (§3.4) — `DOCUMENT_AUDIT.csv` (23-col superset) + TABLE/EQUATION/FIGURE/
  CHART_DATA/CLASSIFICATION catalogs + `CROSS_REFERENCE_INDEX.json`, all `read_method`-tagged.
- Ledger fields (§3.5) — `PROCESSING_LOG.process_type=Hopper`, `KB_CATALOG`, `PROVENANCE_LEDGER`, keyed by
  `source_md5`.

The `read_method` enum (frozen v1.0): `HDARP | SPHDARP | PHDARP | SPDARP | PDARP | Sraffa_OCR | Hopper |
manual`. An agent must be able to answer "how was this doc read?" from the folder alone
(`READ_METHOD.json`) OR from `source_md5 → PROCESSING_LOG`.

## The proven pieces (this skill ORCHESTRATES them — do not reinvent)

All live in `Hopper`, are **stdlib-only**, and were proven on Jane (305 docs).
Run with the eval venv and `PYTHONUTF8=1`:

```
PY="python.exe"
HOP="Hopper"
KB="<P>/Knowledge_Base"
OUT="<P>/KBIP_Integration"
```

| Step | Script | What it does | Writes where |
|------|--------|--------------|--------------|
| TAG | `kbip_backfill.py <KB> [--only ID,..] [--limit N]` | Per doc: `RDB_METADATA.jsonl` (via `hopper_to_rdb.py`), `READ_METHOD.json`, `manifest.read_method`, FULL_TEXT `<!-- read_method -->` header. Idempotent, additive. | inside each `KB/<doc>/` |
| CATALOG | `kbip_catalog.py <KB> <OUT> --batch <ID>` | KBIP v1.0 `DOCUMENT_AUDIT.csv` (§3.4) + TABLE/EQUATION/FIGURE/CHART_DATA/CLASSIFICATION catalogs + `CROSS_REFERENCE_INDEX.json`, all `read_method`-tagged. **Read-only on the KB.** | `OUT/` |
| LEDGERS | `kbip_ledgers.py <OUT>/DOCUMENT_AUDIT.csv --project <P> --batch <ID> [--apply]` | Idempotent backed-up APPEND of `PROCESSING_LOG`/`KB_CATALOG`/`PROVENANCE_LEDGER` rows keyed by `source_md5` (`process_type=Hopper`). **Default DRY-RUN.** | unified Robert ledgers |
| SAFE-PACKAGE | `kbip_safepackage.py <KB> --project <P> [--zip] [--dest DIR]` | `INTEGRATION_CHECK_{date}.json` gate + `BACKUP_MANIFEST.json`; COMBINE only on gate PASS; `--zip` → `KB_HOPPER_{P}_{date}.zip`. | `OUT/` (+ `--dest` for zip) |

A thin `hopper_integrate.py` orchestrator (same dir) chains all four in order with `--dry-run`/`--apply`,
`--project`, `--kb-root`, `--zip`, `--limit`, `--only` — it is the single entry point this skill runs.

## Steps (in this exact order)

### 0. LAND — ensure the project KB holds the Hopper doc folders

A finished Hopper KB may live on the **floor** (`<doc>`) or already be at
`Knowledge_Base`. LAND = make the project KB hold the doc folders in the existing Hopper KB
layout (`<doc>/{manifest.json, confidence.json, hdarp/{FULL_TEXT_chunks_*.md, CSV_Tables/, equations/,
figures/}}`). Rules:

- **If the docs are already in `Knowledge_Base` (the Jane case): LAND is a no-op verify.**
  Confirm the `DOC*` folders exist with their `manifest.json` + `hdarp/` artifacts; do nothing else.
- If the docs are on the floor: **COPY** (never move) `<doc>` → `<doc>`,
  preserving the layout. Archive-don't-delete: leave the floor copy in place.
- **NEVER re-extract.** Hopper already produced the 4 artifacts. LAND only relocates/verifies folders.

Verify with: `manifest.json` and a spot-check that each has an
`hdarp/` subdir.

### 1. TAG — `kbip_backfill.py` (idempotent, additive; writes only inside the KB)

```
PYTHONUTF8=1 "$PY" "$HOP/kbip_backfill.py" "$KB"
```
Emits per doc: `RDB_METADATA.jsonl`, `READ_METHOD.json`, `manifest.read_method`, FULL_TEXT header. Re-runnable
(adds the header once, rewrites tags identically). Use `--only DOC0001,DOC0002` / `--limit N` to scope.

### 2. CATALOG — `kbip_catalog.py` (read-only on the KB)

```
PYTHONUTF8=1 "$PY" "$HOP/kbip_catalog.py" "$KB" "$OUT" --batch <P>_HL2
```
Writes the KBIP v1.0 catalog set to `OUT/`. Confirm: `DOCUMENT_AUDIT.csv` rows all carry `read_method=Hopper`,
`CHART_DATA_CATALOG.csv` exists, `CROSS_REFERENCE_INDEX.json` has `engines_present:["Hopper"]`.

### 3. LEDGERS — `kbip_ledgers.py` (DRY-RUN first, then `--apply`)

```
# 3a. DRY-RUN (default — reports new rows, writes nothing):
PYTHONUTF8=1 "$PY" "$HOP/kbip_ledgers.py" "$OUT/DOCUMENT_AUDIT.csv" --project <P> --batch <P>_HL2
# 3b. APPLY (only when not in --dry-run mode):
PYTHONUTF8=1 "$PY" "$HOP/kbip_ledgers.py" "$OUT/DOCUMENT_AUDIT.csv" --project <P> --batch <P>_HL2 --apply
```
Idempotent (keyed by `(source_md5, project)`; replay adds nothing), backs up each ledger to
`Robert/_kbip_ledger_backups/<date>/` before appending.

### 4. SAFE-PACKAGE — `kbip_safepackage.py` (gate; COMBINE only on PASS)

```
PYTHONUTF8=1 "$PY" "$HOP/kbip_safepackage.py" "$KB" --project <P>            # gate + manifest
PYTHONUTF8=1 "$PY" "$HOP/kbip_safepackage.py" "$KB" --project <P> --zip      # + dated zip backup
```
Writes `INTEGRATION_CHECK_{date}.json` (per-doc: 4-artifact + chunk markers + RDB_METADATA schema-valid +
READ_METHOD present + manifest provenance + PENDING accounted + ledger row). **COMBINE happens only if the
gate PASSES**; a broken doc blocks COMBINE. Prior packages are archived, never deleted.

### 5. HANDOFF

```
/kb-integrate-pipeline <P> --engine hopper     # the shared 7-phase method-agnostic pipeline
robert-db-build <P>                            # robert-db is READ_METHOD.json-aware -> process_type=Hopper
```
`robert-db-build` auto-detects `process_type=Hopper` from each doc's `READ_METHOD.json`; the
`transcription_status` axis is pre-populated from the Hopper confidence sidecar (no schema change vs an HDARP
project).

## Dry-run mode (the acceptance)

`/hopper-integrate <P> --dry-run` (the default) **lists every doc and its planned artifacts and writes
nothing outside the project KB + ledgers** — in dry-run it writes nothing at all. For each doc it reports:
the planned `RDB_METADATA.jsonl` (line count = number of table CSVs), `READ_METHOD.json`,
`manifest.read_method` + FULL_TEXT header, the catalog rows it would contribute (audit/table/equation/figure/
chart), the `PROCESSING_LOG`/`KB_CATALOG`/`PROVENANCE_LEDGER` rows it would append (keyed by `source_md5`),
and whether the safe-package gate would PASS. CATALOG and LEDGER-dry-run are themselves read-only, so the
dry-run executes them (they write only to `OUT/` for catalogs, which is the project's own Technical tree) and
reports the would-be ledger/tag/package deltas without applying any tag, ledger append, or package combine.

### Dry-run for Jane — exact command

```bash
PY="python.exe"
PYTHONUTF8=1 "$PY" "hopper_integrate.py" \
  --project Jane \
  --kb-root "Knowledge_Base" \
  --dry-run
```

What it outputs for Jane: LAND = no-op verify (305 `DOC*` folders already in
`Knowledge_Base`); then, per doc, the planned `RDB_METADATA.jsonl` / `READ_METHOD.json` /
`manifest.read_method` / FULL_TEXT-header tags; a KBIP v1.0 catalog summary (305 docs, all
`read_method=Hopper`, `engines_present:["Hopper"]`, table/equation/figure/chart counts); the
`PROCESSING_LOG`/`KB_CATALOG`/`PROVENANCE_LEDGER` rows that *would* be appended for the 305 Jane docs
(none applied); and the safe-package gate result — all with **zero** writes to the ledgers and zero tag/
package mutations.

## Non-negotiables

- **Hopper ≠ HDARP (naming).** The extraction engine is **Hopper Line v2 (HL2)**. NEVER call it "HDARP",
  "Local-HDARP", or "Sraffa N". Every artifact this skill writes is tagged `read_method=Hopper` /
  `process_type=Hopper` — never `HDARP`. (See `feedback_hdarp_naming.md`.)
- **Archive-don't-delete.** LAND copies (never moves); ledgers are backed up before any append; prior
  safe-packages are archived, not overwritten. Nothing is deleted.
- **Idempotent / re-runnable.** Every step is keyed (FULL_TEXT header added once; tags rewritten identically;
  ledger rows keyed by `(source_md5, project)`; catalogs regenerated). Replay produces no duplicates.
- **Read-only on extraction artifacts.** This on-ramp only ADDS sidecars (`RDB_METADATA.jsonl`,
  `READ_METHOD.json`), tags (`manifest.read_method`, FULL_TEXT header), and catalogs. It never edits the
  body text, tables, equations, or figures Hopper produced.
- **GPU-free.** This is post-extraction integration — no models load, no GPU is used. (Extraction was
  `/hopper`; this is the on-ramp after.)
- **Dry-run is the default.** Writing requires `--apply`.

## Lifecycle position

Hopper lane: `/hopper` (extract) → **`/hopper-integrate`** (LAND + TAG + CATALOG + LEDGER + SAFE-PACKAGE) →
`/kb-integrate-pipeline <P> --engine hopper` → `robert-db-build <P>`.

This is the Hopper sibling of the HDARP lane's `/kb-integrate-pipeline` (alias
`/hdarp-integrate-pipeline`); both share the KBIP v1.0 contracts.

---

**Skill Version**: 1.0
**Command**: `/hopper-integrate`
**Standard**: KB Integration Pipeline (KBIP) v1.0 (`KB_INTEGRATION_PIPELINE_BUILD_PLAN.md` §4 W3)
**Created**: 2026-06-22
**Engine pieces**: `{hopper_integrate,kbip_backfill,kbip_catalog,kbip_ledgers,kbip_safepackage}.py` (stdlib-only; eval venv)
**Proven on**: Jane (305 docs)
