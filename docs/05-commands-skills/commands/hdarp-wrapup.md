---
description: "HDARP Wrap-Up v6.4 (command lineage v1.0): Post-processing validation, remediation, documentation, and wave/batch closeout"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
argument-hint: "[wave|batch|campaign] [target]"
---

**HDARP Framework v6.4** — see `VERSION_REGISTRY.md`

# HDARP Wrap-Up Command v1.0

**Command**: `/hdarp-wrapup [scope] [target]`
**Purpose**: Validate, remediate, document, and close out processed HDARP batches
**Version**: 6.4
**Created**: 2026-05-06

## HDARP Lifecycle Position

This command is step **4** (WRAP-UP) in the HDARP lifecycle. Canonical order and membership — the
same string appears in `CLAUDE.md`, `instructions.md` and `readystart.md`:

1. `/preparehdarp` — chunk and prepare PDFs
2. `/sphdarp` (or `/phdarp`, `/spdarp`, `/pdarp`) — extract all 4 content types via agents
3. `/enrichhdarp` — audit + remediate extraction gaps
4. **`/hdarp-wrapup`** — validate, remediate, document, close out
5. `/sraffa-ocr --augment` — **Hybrid Stage 5: MANDATORY, every document, no exceptions**
6. `/hdarp-cleanup` — remove chunk artifacts with catalog verification
7. `/kb-integrate-pipeline` — catalog, classify, crossref, Robert sync (`/hdarp-integrate-pipeline` alias)

**Run this AFTER processing is complete and BEFORE Stage 5 and integration.**

> **Stage 5 is not conditional.** It was written "(if needed)" until 2026-08-27, which is why it ran
> on 5 of 143 documents in a recent wave while 353 of 384 documents had no verbatim body text
> anywhere. Every document gets a verbatim OCR sibling in `Knowledge_Base/_OCR_Only/<short_id>/`
> beside its agent-read body text — augmentation, never substitution. The two-layer rule is canonical
> in `hdarp-processing.md` ("Hybrid body text = two layers"); the mode is
> `sraffa-ocr.md` Mode 4.
>
> **Owner (one owner — this statement appears identically in `sphdarp.md` Phase 5 and
> `sraffa-ocr.md`):** the **`/sphdarp` orchestrator** owns Hybrid Stage 5 and runs it as its
> **Phase 5**, once per drain scope, by invoking `/sraffa-ocr --augment` after `/hdarp-wrapup` has
> gated the scope. **This command GATES Stage 5 and never RUNS it**; `/sraffa-ocr` never schedules
> itself and is never invoked inline mid-round. "Once per drain scope" is defined precisely in
> `sphdarp.md` Phase 5 — do not re-derive it here.

## Usage

```
/hdarp-wrapup wave Wave_02              # Wrap up entire wave
/hdarp-wrapup batch BATCH_1205          # Wrap up single batch
/hdarp-wrapup campaign Second_HDARP     # Wrap up entire campaign
/hdarp-wrapup --audit-only wave Wave_02 # Phase 1 only (read-only)
```

## Scope

- **wave**: All batches in the specified wave
- **batch**: Single batch
- **campaign**: All waves in the campaign

## Execution (9 Phases)

### Phase 1: Extraction Quality Audit

For every COMPLETE batch in scope:

1. **4-Type Completeness Check** (MANDATORY — see `hdarp-processing.md`):
   - Does `CSV_Tables/` exist with CSVs or `_no_tables.txt`?
   - Does `equations/` exist with `.tex` files or `_no_equations.txt`?
   - Does `figures/` exist with `.md` files or `_no_figures.txt`?
   - Do body text files have HDARP chunk markers (`<!-- chunk_NNN -->`)?

2. **PyMuPDF Dump Detection** (CRITICAL — from 2026-05-06 incident):
   - Files named `PYMUPDF_TEXT_DUMP.md` → batch needs reprocessing
   - `Text/chunk_NNN_body.txt` with `[PAGE NEEDS OCR]` → not HDARP output
   - KB with ONLY `Text/` + `FULL_TEXT.md` and no structured dirs → not HDARP
   - **Any detection → Tier E, batch reverts to PREPARED**

3. **Chunk Coverage Check**:
   - Count body text files vs expected chunks (from BATCH_STATE)
   - Flag batches where chunks_processed < total_chunks

4. **Spot-Check** (2 random body text files per batch):
   - File > 200 chars of substantive content
   - Contains chunk boundary markers (`<!-- chunk_NNN -->`)
   - Is not a stub or placeholder

5. **Scrounger Standard Compliance** (per `sphdarp-scrounger.md`):
   - Any chunk with zero extraction (no body text, no tables, no equations, no figures) must have a documented reason
   - Only 3 acceptable reasons: DUPLICATE (confirmed identical, not assumed), CF_EXHAUSTED (after full L1 bisect + L2 scholarly framing on every page), QUARANTINED (physically unreadable)
   - Batches with undocumented zero-extraction chunks → classify Tier D

**Output**: `WAVE_NN_AUDIT_REPORT.json` with per-batch tier classification:

| Tier | Criteria | Action |
|------|----------|--------|
| **A** | All 4 types complete, all chunks covered, spot-checks pass | → Phase 3 (validate) |
| **B** | Body text + structured types present, minor gaps | → Phase 2 (remediate), then Phase 3 |
| **C** | Body text complete but missing structured extraction dirs | → needs partial reprocessing |
| **D** | Empty or stub output | → needs full reprocessing |
| **E** | PyMuPDF dump detected | → MUST revert to PREPARED and reprocess |

If `--audit-only` was passed, STOP here and display results.

### Phase 2: Remediation

**Tier A batches**: Skip — already clean.

**Tier B batches**:
- Run `/enrichhdarp audit` to identify specific gaps (Type A-D failures)
- Run `/enrichhdarp remediate` on actionable failures
- Run `/sraffa-ocr --chunks` on pages flagged for OCR (L3 CF rescue only — **not** Stage 5)
- Re-classify after remediation (B batches may promote to A)

**Tier C batches**:
- Report as needing reprocessing — these have body text but no structured extraction
- Do NOT attempt to fix — they need `/sphdarp` re-runs with agent extraction
- Keep status COMPLETE but flag for user action

**Tier D/E batches**:
- Revert status to PREPARED
- Preserve chunk PDFs (needed for reprocessing)
- Add note: "REVERTED by hdarp-wrapup: {reason}"
- Report to user as requiring reprocessing

### Phase 3: Opus Validation

For every Tier A batch (and promoted Tier B after remediation):

**The validator MUST perform ALL of these checks:**

1. Read BATCH_STATE.json for batch metadata (total_chunks, documents)
2. Verify KB output completeness:
   - `FULL_TEXT_chunks_*.md` or `FULL_TEXT.md` covering all chunks
   - `CSV_Tables/` directory with content or `_no_tables.txt`
   - `equations/` directory with content or `_no_equations.txt`
   - `figures/` directory with content or `_no_figures.txt`
3. Spot-check 2 body text files: >200 chars substantive content with chunk markers
4. Score: Content (9), Accuracy (9), Structure (9) = 27 max, 22 minimum
5. **4-Type Completeness is a HARD GATE**: missing content type dir without marker → FAIL regardless of score
6. Write substantive validation_notes (50+ chars describing what was physically checked)
7. **Scrounger Validator Check** (per `sphdarp-scrounger.md`): body text files MUST contain chunk boundary markers (`<!-- chunk_NNN -->` or `[CHUNK_NNN_START]`). PyMuPDF text dumps do not have these markers. If no chunk markers → FAIL regardless of score, revert to PREPARED.
8. **Digest-mode anti-fabrication check (v6.2)**: when `extraction_method == analytical_digest` (in-copyright/trade books), spot-check that body text is genuine paraphrase/digest in a neutral encyclopedic tone — NOT fabricated verbatim quotes, letters, poems, or anecdotes. Long block quotes or invented attributed text → FAIL. Short genuine quotes (<~15 words) are acceptable.
9. **ROLLING_OVERLAP legitimacy (v6.2)**: a chunk carrying an explicit overlap marker that names the canonical chunk it duplicates (a splitter artifact) is LEGITIMATE, not degradation — treat as extracted (≤1-pt deduction max), do NOT FAIL the batch for it.
10. **Native RDB metadata check (v6.3 — WARN-only)**: confirm `RDB_METADATA.jsonl` (or `_chunks_*.jsonl` shards) exists with ≥1 line per non-marker table CSV; lines parse, `source_relpath` resolves, `field_basis` covers present fields (valid vocab), `transcription_status` ∈ {H,L,R,X} (never V); honesty spot-check ≥2 lines (`from_source` must be verbatim). **WARN deduction (≈ −2/27) in validation_notes, NOT a hard fail** — the 4-type gate (check 5) is unchanged; gaps are backfillable by `enrichhdarp` Type E. See `hdarp-processing.md` "Native RDB Enrichment Capture".

**ANTI-AUTO-VALIDATION**: Every check must be physical file verification. No rubber-stamping. See sphdarp.md for the 2026-04-28 abdication audit and 2026-05-06 silent degradation incidents.

If PASS (≥22/27):
- Set status → VERIFIED
- Set verified_date, validation_score, validation_notes
- Remove from validation_queue

If FAIL:
- Keep status as-is, add failure notes
- Flag for remediation or reprocessing

### Phase 4: FULL_TEXT.md Assembly

For verified batches that have chunk-range files but no consolidated FULL_TEXT.md:

1. Concatenate all `FULL_TEXT_chunks_NNN_NNN.md` files in chunk order
2. Preserve chunk boundary markers (`<!-- chunk_NNN -->`) between sections
3. Write consolidated `FULL_TEXT.md`
4. Verify assembled size is reasonable (within 5% of sum of parts)
5. **Native RDB metadata consolidation (v6.3; corrected v6.4)**: consolidate the `RDB_METADATA_chunks_*.jsonl` shards into a single `RDB_METADATA.jsonl`.
   🔴 **If you consolidate, you MUST retire the shards** — move them out of the `Knowledge_Base/` (to `Technical/HDARP_Processing/_repair_backups/`), never delete them. `robert-db-harvest` globs `RDB_METADATA*.jsonl` and reads the consolidated file **and** the shards, so leaving both in place **double-counts every absorbed table**. The earlier wording here — *"or leave shards in place — robert-db-harvest reads both"* — was wrong and produced the defect: measured 2026-08-27, 31 documents carried 56 fully-absorbed shards worth 241 duplicate metadata lines.
   **Retire a shard ONLY when its lines are a strict subset of the consolidated file.** If a shard holds even one line the consolidated file lacks, retiring it destroys metadata — leave that document alone and report it (20 documents were in this state).
   **Do NOT resolve conflicting lines by "last writer wins."** Two lines for the same `source_relpath` with different `title_raw`/`page`/`transcription_status` are two readings that disagree; one may be a correction of the other. Picking the last silently discards corrections at scale. Record them for a human (see `SIDECAR_CONFLICTS.md` for the pattern) and leave both in place. Do NOT fabricate lines for tables no shard covered.

### Phase 5: Chunk PDF Cleanup

For every VERIFIED batch:

1. Verify KB content is intact: FULL_TEXT.md exists, structured dirs exist
2. Delete chunk PDFs from `Technical/HDARP_Processing/{doc_id}/chunks/`
3. Delete manifest.json
4. Remove empty directories
5. Log space freed per batch and cumulative total

**NEVER delete chunk PDFs for non-VERIFIED batches.** This is the rule that was violated in the 2026-05-06 incident.

### Phase 6: METADATA.json Creation

For verified batches missing METADATA.json in their KB directory:

1. Generate from available KB content and BATCH_STATE:
   ```json
   {
     "title": "...",
     "author": "...",
     "year": "...",
     "page_count": N,
     "chunk_count": N,
     "extraction_date": "YYYY-MM-DD",
     "hdarp_version": "6.0",
     "quality_score": "NN/27",
     "tables_count": N,
     "equations_count": N,
     "figures_count": N,
     "source_batch": "BATCH_NNNN",
     "wave": "Wave_NN"
   }
   ```
2. Write to `Knowledge_Base/{doc_id}/METADATA.json`

### Phase 7: Catalog Sync

Update HDARP_MASTER_CATALOG.csv for every batch in scope:
- `hdarp_status`: VERIFIED, PREPARED (if reverted), DEPRECATED, SKIPPED
- `quality_score`: from validation
- `completion_date`: today
- `chunk_count`, `tables_extracted`, `figures_extracted`, `equations_extracted`
- `wave`, `batch_id`

If row exists, update. If not, append.

### Phase 8: CURRENT_STATUS.md Update

Update `Handoffs/CURRENT_STATUS.md`:
- Wave progress matrix row with accurate counts
- Batch state totals (recalculate from BATCH_STATE.json)
- Latest activity section with wrapup summary
- Campaign totals and "waves at 100%" count
- Update `last_updated` header

### Phase 9: Wave Handoff Report

Generate `Technical/HDARP_Processing/WAVE_NN_WRAPUP_REPORT.md`:

```markdown
# Wave NN Wrap-Up Report
**Date**: YYYY-MM-DD
**Agent**: [model]
**Scope**: [wave/batch/campaign]

## Final Status
| Status | Count |
|--------|------:|
| VERIFIED | N |
| PREPARED (needs reprocessing) | M |
| DEPRECATED | K |
| SKIPPED | J |

## Extraction Quality
- Tier A (full HDARP, all 4 types): N batches
- Tier B (remediated to full): M batches
- Tier C (needs partial reprocessing): K batches
- Tier D/E (needs full reprocessing): J batches

## Content Extracted
- Total documents: N
- Total pages: N
- Body text: N files (total KB)
- CSV tables: N files
- Equation files: N files
- Figure descriptions: N files

## Disk Space
- Chunk PDFs freed: X GB
- KB output retained: Y MB

## OCR Backlog (L3 CF rescue)
- Pages deferred to /sraffa-ocr --chunks: N

## Hybrid Stage 5 readiness (MANDATORY — gate)
- Documents in scope: N
- Documents with a verbatim sibling at `Knowledge_Base/_OCR_Only/<short_id>/`: N
- Bounded exception rows in `_OCR_Only/STAGE5_EXCEPTIONS.csv` (DUPLICATE / QUARANTINED / GPU_DEFERRED only): N
- Of those, open `GPU_DEFERRED`: N — **non-terminal; while >0 the scope is Stage 5 INCOMPLETE**
- **Identity check: documents_in_scope == siblings + exception_rows — must hold exactly, no document in both halves**
- **Documents in NEITHER half: N — any non-zero value FAILs the wrap-up gate below**

## Reprocessing Queue
- Batches needing /sphdarp re-run: [list with reasons]

## Ready for Integration?
- YES: all batches VERIFIED or terminal → proceed to /hdarp-integrate-pipeline
- NO: M batches still need work → [what remains]
```

## Stopping Conditions

Wrap-up is complete when:
- All batches in scope are in a terminal state (VERIFIED, DEPRECATED, SKIPPED) OR documented as needing reprocessing
- **Hybrid Stage 5 gate — bounded, per-document, artifact-backed.** Every document in scope must
  either **(a)** have a verbatim sibling at `Knowledge_Base/_OCR_Only/<short_id>/` (non-empty
  `FULL_TEXT.md` plus a manifest covering every source page), or **(b)** have its own row in the
  named artifact `Knowledge_Base/_OCR_Only/STAGE5_EXCEPTIONS.csv` whose `reason` is one of exactly
  three — `DUPLICATE`, `QUARANTINED`, `GPU_DEFERRED` — each with its bar and required `detail`
  defined in `sphdarp.md` Phase 5, "The closed exception list". **That list is closed and the
  artifact is required: a reason given in prose, in this report, or in a commit message does not
  count, and no reason outside the three is acceptable however carefully recorded.** `GPU_DEFERRED`
  is **non-terminal** — a scope with any open `GPU_DEFERRED` row is Stage 5 INCOMPLETE, not
  gated-through. Report the identity `documents_in_scope == siblings + exception_rows` with all
  three numbers; a document in neither half, or in both, FAILs. A missing sibling is never grounds to
  accept a text dump in the document's KB directory instead — that is the substitution the
  anti-silent-degradation rule forbids
- HDARP_MASTER_CATALOG.csv is synced
- CURRENT_STATUS.md is updated
- Wrapup report is written

## Error Handling

- BATCH_STATE.json encoding: use `utf-8-sig` (may have BOM)
- Missing chunk PDFs: note in report, proceed with validation on existing KB content
- Missing KB directory: flag batch for reprocessing, do not attempt to validate
- PyMuPDF dumps detected: revert to PREPARED immediately, do NOT validate

## Related Commands

| Command | Relationship |
|---------|-------------|
| `/enrichhdarp` | Called by Phase 2 for Type A-D remediation |
| `/sraffa-ocr --chunks` | Called by Phase 2 for L3 CF rescue on OCR gap pages |
| `/sraffa-ocr --augment` | **Step 5 — Hybrid Stage 5, mandatory over every document. Owned and run by `/sphdarp` Phase 5; this command GATES it and never RUNS it (see Stopping Conditions)** |
| `/hdarp-cleanup` | Phase 5 overlaps; wrapup handles cleanup inline |
| `/kb-integrate-pipeline` | Step 7 (`/hdarp-integrate-pipeline` is a backward-compatible alias) |
| `robert-db-build` | Downstream of integration: builds the research-grade table database (Robert Database Framework v1.0 — `ROBERT_DATABASE_FRAMEWORK_OVERVIEW.md`) |
| `/sphdarp` | Prior step (step 3); wrapup validates its output |

---

**Command Version**: 6.4
**Status**: PRODUCTION READY
**Created**: 2026-05-06
**HDARP Protocol**: v6.4
**Key**: Post-processing validation + remediation + documentation + closeout

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): added WARN-only native RDB metadata validation and sidecar consolidation. -->
