---
name: robert-db-audit
version: "1.0"
description: "QA gate for a Robert database: run rdb_audit.py with/without --gate, read the AUDIT_REPORT, run the honesty-contract checks, and perform a stratified agent spot-check that records verification_status via a reviewed patch (or a documented sqlite3 one-liner)."
when-to-use: '"User wants to audit a Robert database, run the QA gate before publish, check provenance/honesty invariants, or spot-check extracted tables against source context."'
search-hints: "robert db audit gate AUDIT_REPORT honesty contract provenance check spot-check verification_status stratified sample exit code"
argument-hint: "[project]"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit, Agent
requires: robert-db-organize
part-of: Robert Database Framework v1.0
---

# robert-db-audit — Stage 5 (AUDIT)

The QA gate that stands between a built database and a published one. It runs the
mechanical audit, surfaces honesty-contract violations, and drives an agent
spot-check of real tables against their source context.

## Purpose

1. Run `rdb_audit.py` to produce an `AUDIT_REPORT` and (with `--gate`) a pass/fail
   exit code.
2. Confirm the honesty invariants hold (provenance coverage, no-downgrade, flags in
   vocabulary, marker-vs-empty sanity).
3. Perform a **stratified spot-check**: an agent compares CSV content against the
   source context and records `verification_status` honestly.

## Preconditions

- `robert-db-organize` has run (taxonomy ratified, concordances committed) so the
  audit sees a fully built DB. Enrichment for Tier A/B should be substantially done.

## Procedure

1. **Report (no gate).** Produce the human-readable report first:
   ```bash
   PYTHONIOENCODING=utf-8 python rdb_audit.py \
       --config <P>/Technical/RobertDB/robertdb_config.json
   ```
   Writes `Technical/RobertDB/audit/AUDIT_REPORT.{md,json}`. Exit code is
   informational here.

2. **Read the AUDIT_REPORT.** It tallies, per check: provenance coverage (every
   published field has a `field_provenance` basis), honesty-contract checks
   (`from_source` never overwritten; no field value present without a basis;
   `not_captured` rendered as such, not as blank-pretending-to-be-data), flag
   vocabulary conformance (no flag outside `config.flag_vocabulary`), parse health
   (`unparseable`/`ragged` counts), count reconciliation vs `DOCUMENT_AUDIT`, tier
   coverage, and `verification_status` distribution. `error`-severity items must be
   zero to pass; `warn`-severity items should be explained.

3. **Gate.** When you intend to publish, run the gate:
   ```bash
   PYTHONIOENCODING=utf-8 python rdb_audit.py \
       --config <P>/Technical/RobertDB/robertdb_config.json --gate
   ```
   Exit `0` = pass (publish may proceed), exit `1` = fail (fix and re-run). The gate
   fails on any `error`-severity check.

4. **Stratified spot-check (agent).** Sample tables stratified across tier, quality
   grade, convention, and language. For each sampled `table_uid` an agent:
   - reads the table's CSV and its source context (`Tables/*.md`, `FULL_TEXT`/`Text`
     chunk);
   - compares the parsed/enriched record against what the source actually shows
     (title, units, page, a few cell values, the `from_source` basis claims);
   - records a verdict: `spot_checked_ok` (matches) or `spot_checked_issue`
     (discrepancy), with a note.
   Recording the verdict — **two permitted methods**:
   - **Preferred: a reviewed patch.** Emit a small JSONL with the sampled tables and
     merge it like any enrichment round (records the verdict + a `quality_observations`
     note through the contract path). This keeps verification on the audited,
     idempotent write path.
   - **Documented sqlite3 one-liner** (permitted in AUDIT ONLY, never elsewhere):
     ```bash
     PYTHONIOENCODING=utf-8 python -c "import sqlite3,sys; c=sqlite3.connect(sys.argv[1]); \
c.execute(\"update xtables set verification_status=? , updated_at=strftime('%Y-%m-%dT%H:%M:%SZ','now') where table_uid=?\", \
(sys.argv[2], sys.argv[3])); c.commit(); print('set',sys.argv[3],sys.argv[2])" \
         <P>/Technical/RobertDB/robertdb.sqlite spot_checked_ok <PJ>-T-000123
     ```
     Log every such update in the audit notes. Direct DB writes are otherwise
     forbidden — this is the single documented exception.

5. **Recompute + re-gate.** If a spot-check surfaced issues, recompute quality
   (`rdb_quality.py --recompute-all`), regenerate views (`rdb_views.py`), and re-run
   the gate.

## A10 view-freshness (WAL-safe; 2026-07-15)

A10 judges whether `views/DB_MANIFEST.json` reflects the current DB. Its **verdict is
timestamp-based**: fresh iff `manifest.generated_at >= ` the last content-mutating run.
Two engine hardenings (from a production A10 reconciliation) make it reliable:

- **`publish` is excluded from "content-mutating"** (alongside `audit`/`views`). A
  `publish` run recorded after a views regen does not change DB content; counting it
  spuriously flipped A10 to WARN. If A10 ever WARNs only because of a later publish,
  the fix is a `rdb_views.py` regen, not a re-harvest.
- **`content_digest_match` is the signal to trust, not `hash_match_live`.** The raw-file
  hash (`hash_match_live`) is **always false on a live WAL DB** (checkpoints reorder
  pages with zero logical change) and is now labelled advisory in the report. A10
  instead reports a stable, WAL-invariant `content_digest` (ordered hash over every
  table's `(table_uid, content_sha256)` + row counts, written into the manifest by
  `rdb_views.py`); `content_digest_match=true` means the views were generated from the
  same harvested content that is live now. Manifests predating this field show
  `manifest_content_digest=null` until the next `rdb_views.py` run repopulates it.

## Outputs

- `audit/AUDIT_REPORT.md` + `.json`.
- Updated `verification_status` on sampled tables (via reviewed patch or the
  documented one-liner).
- An `audit` run-ledger row; gate exit code.

## Failure handling

- **Gate exit 1** → open AUDIT_REPORT, fix the `error`-severity items (most are
  provenance-coverage or flag-vocabulary problems), re-run. Do not pass the gate by
  editing the report.
- **Spot-check finds fabricated `from_source`** → mark `spot_checked_issue`, send the
  table back to `robert-db-enrich` to correct the basis to `agent_inferred`/
  `not_captured`; this is a stop-ship until fixed.
- **`warn` items** → explain each in the audit notes; unexplained warns block publish
  unless `rdb_publish.py --allow-warn` is justified by the user.
