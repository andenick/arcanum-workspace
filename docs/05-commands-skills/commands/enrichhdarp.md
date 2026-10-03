---
description: "HDARP Enrichment Audit & Remediation Protocol: Audit existing output and remediate enrichment failures"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent
argument-hint: "[audit|remediate|full] [BATCH_NNN|CAMPAIGN_X]"
---

**HDARP Framework v6.3** — see `VERSION_REGISTRY.md`

# HDARP Enrichment Audit & Remediation Protocol (ENRICHHDARP)

**Command**: /enrichhdarp [mode] [target]
**Full Name**: HDARP Enrichment Audit & Remediation Protocol
**Version**: 6.3
**Created**: 2026-03-15
**Updated**: 2026-04-30 — Sraffa 4.0 document-adaptive OCR integration for Type D remediation

## HDARP Lifecycle Position

This command runs **between steps 3 and 4** — it remediates quality issues found during processing before the formal wrap-up. It is called BY `/hdarp-wrapup` as part of Phase 2 (Remediation), or can be run standalone.

**Quality Baseline**: enrichhdarp audits against the full 4-content-type HDARP standard (tables, equations, figures, body text). Output that is only body text (e.g., PyMuPDF dumps) will be flagged as Type D failure (missing content types).

## Usage

```
/enrichhdarp audit BATCH_021        # Audit a specific batch
/enrichhdarp audit CAMPAIGN_B       # Audit entire campaign
/enrichhdarp remediate BATCH_022    # Fix enrichment failures in a batch
/enrichhdarp full BATCH_022         # Audit + remediate in one pass
```

## Modes

| Mode | Description |
|------|-------------|
| `audit` | Scan existing HDARP output, classify documents, generate audit report |
| `remediate` | Fix identified enrichment failures (requires prior audit or manifest) |
| `full` | Combined audit + remediate in one pass |

## DARP Command Family Context

| Command | Purpose | Gap Filled by /enrichhdarp |
|---------|---------|---------------------------|
| /preparehdarp | Chunk PDFs, create directories | N/A |
| /sphdarp N | Smart parallel extraction (forward) | No audit of existing output |
| /phdarp N | Parallel hybrid extraction (forward) | No audit of existing output |
| /spdarp N | Smart parallel DARP (forward) | No enrichment review |
| /pdarp N | Parallel DARP (forward) | No enrichment review |
| /hdarp-cleanup | Remove chunk artifacts | N/A |
| **/enrichhdarp** | **Audit + remediate existing output** | **THIS COMMAND** |

---

## Phase 1: DISCOVERY

Read BATCH_STATE.json to resolve the target scope.

```
BATCH_STATE path: {Project}/Technical/HDARP_Processing/BATCH_STATE.json
Catalog path: {Project}/Technical/HDARP_MASTER_CATALOG.csv
Knowledge_Base: {Project}/Knowledge_Base/
```

### Target Resolution
- `BATCH_NNN` → Read that batch's documents from BATCH_STATE
- `CAMPAIGN_X` → Filter all batches with `campaign: "X"`, collect all documents

### For each document, resolve:
1. Document ID (from BATCH_STATE)
2. Expected chunk count
3. Knowledge_Base directory path
4. Current status (COMPLETE/VERIFIED/etc.)

---

## Phase 2: AUDIT (parallel agents, max 3 per wave)

Deploy audit agents. Each agent reviews 2-3 documents max (avoid "prompt too long" errors).

### Per-Document Audit Protocol (6 dimensions)

> **Dimension 6 — Native RDB metadata completeness (v6.3):** for each doc, compare the
> `RDB_METADATA.jsonl` (+ `_chunks_*.jsonl`) line count to the number of non-marker table CSVs.
> Report `RDB_META_OK` (≥1 line per non-marker table), `RDB_META_PARTIAL` (some missing), or
> `RDB_META_MISSING` (no sidecar). PARTIAL/MISSING → **Type E** remediation (backfill the missing
> lines from the chunk PDFs while they survive). This is a metadata-completeness signal, **not** a
> 4-type failure — never reclassify a 4-type-complete doc as hollow on this basis alone.

**1. Structural Verification**
```
For each document KB directory:
  Count files in body_text/, equations/, tables/, figures/
  Compare to BATCH_STATE chunk_count
  Report: MATCH or MISMATCH (with details)
```

**2. Body Text Classification**
```
For each body_text file, classify by size:
  EMPTY:       0 bytes
  STUB:        1-199 bytes
  THIN:        200-499 bytes
  ADEQUATE:    500-1,999 bytes
  SUBSTANTIAL: 2,000+ bytes

Sample 3-5 files for content quality:
  - Check for placeholder text ("Content from chunk...", "This chunk appears to be empty")
  - Check for page-marker-only content ("--- Page N ---")
  - Check for garbled OCR (character substitution patterns)
```

**2b. Hollow Root Cause Detection** (v1.1)
```
For HOLLOW documents, determine cause by testing the first chunk PDF:

Run: python -c "import fitz; d=fitz.open('CHUNK_PDF'); print(len(d[0].get_text()))"

  HOLLOW_SCANNED:  PyMuPDF returns <100 chars (scanned images, needs Sraffa 4.0 OCR)
  HOLLOW_DIGITAL:  PyMuPDF returns >100 chars (text layer exists but extraction failed — re-extract)
  HOLLOW_CORRUPT:  PyMuPDF throws error (corrupt PDF — skip, document in report)

This determines whether Type B (re-extract) or Type D (OCR) remediation is needed.
```

**3. Enrichment Classification**
```
For each enrichment file (equations/*.tex, tables/*.csv, figures/*.md):
  REAL:        Contains actual extracted content
  PLACEHOLDER: Contains "No equations found" / "No tables found" / "No figures found"
  CF_BLOCKED:  Contains "Content-filter blocked" / "OCR fallback" message
  MISSING:     File does not exist

Calculate per-document rates:
  eq_real_rate  = real_equations / total_chunks
  tab_real_rate = real_tables / total_chunks
  fig_real_rate = real_figures / total_chunks
```

**Placeholder Detection Patterns** (regex):
```
EQUATIONS:
  /^%?\s*(No equations|Equations not extracted|No mathematical)/i
  /^%?\s*Content-filter blocked/i

TABLES:
  /^(No tables|Tables not extracted|No tabular)/i
  /^Content-filter blocked/i

FIGURES:
  /^(No figures|Figures not extracted|No visual)/i
  /^Content-filter blocked/i
```

**4. Anomaly Detection**
- Extra files outside 4 subdirectories
- Files with non-standard naming (not chunk_NNN_type.ext)
- Misplaced content (wrong directory)
- Ghost/duplicate directories

**5. Cross-Reference**
- BATCH_STATE chunk_count vs filesystem file counts
- BATCH_STATE document status vs actual content quality

### Document Classification

Based on audit results, classify each document:

| Classification | Criteria |
|---------------|----------|
| FULL_HDARP | All 4 types present with real content (>50% enrichment rate) |
| PARTIAL | Body text OK, enrichment <50% real content |
| BODY_ONLY | Body text present, ALL enrichment is placeholder |
| HOLLOW | Body text is page markers / empty, no real content |
| EMPTY | Missing or zero-byte files |

### Verdict Assignment

| Verdict | Criteria |
|---------|---------|
| PASS | FULL_HDARP or PARTIAL with >30% enrichment |
| PASS_WITH_WARNINGS | PARTIAL with <30% enrichment, or BODY_ONLY with legitimate reason (CF-blocked) |
| FAIL | HOLLOW, EMPTY, or BODY_ONLY without CF-blocked justification |

### Audit Report Output

Generate: `{Project}/Technical/HDARP_Processing/{TARGET}_ENRICHMENT_AUDIT.md`

```markdown
# Enrichment Audit Report: {TARGET}

**Date**: {date}
**Scope**: {batch/campaign details}

## Summary
| Classification | Count |
|---------------|-------|
| FULL_HDARP | N |
| PARTIAL | N |
| BODY_ONLY | N |
| HOLLOW | N |
| EMPTY | N |

## Per-Document Results
(table with doc ID, chunks, body classification, eq/tab/fig rates, verdict)

## Remediation Manifest
(JSON listing documents needing remediation, by type)
```

---

## Phase 3: REMEDIATION PLANNING

Classify remediation needs from audit results:

| Type | Condition | Action |
|------|-----------|--------|
| **Type A: Enrichment-only** | Body text OK, eq/tab/fig placeholder | Read chunk PDFs → extract eq/tab/fig only |
| **Type B: Full recovery** | Hollow body text (not CF-blocked) | Full 4-type extraction from chunk PDFs |
| **Type C: Stub replacement** | Individual placeholder body_text chunks | Re-extract specific chunks from PDFs |
| **Type D: OCR Recovery** | Hollow body text (HOLLOW_SCANNED) | Sraffa 4.0 document-adaptive OCR (EasyOCR GPU + Agent QA + Chandra 2 escalation) |
| **Type E: Native RDB metadata backfill (v6.3)** | 4 types OK but `RDB_METADATA*.jsonl` sidecar missing/incomplete (line count < non-marker table CSVs) | Agents read the chunk PDFs + table CSVs and emit the missing sidecar lines per `NATIVE_ENRICHMENT_CONTRACT.md` (same honesty rules: `from_source` only if verbatim; `null`+`not_captured` otherwise; `transcription_status` H/L/R/X, never V). Possible **only while chunk PDFs survive** — run before `hdarp-cleanup`. |

### Generate Remediation Manifest

```json
{
  "target": "BATCH_022",
  "audit_date": "2026-03-15",
  "remediation_items": [
    {
      "document_id": "[B] [2005] Chiang & Wainwright...",
      "type": "D",
      "chunks_affected": 71,
      "description": "Content-filter blocked. All 71 chunks hollow.",
      "action": "Sraffa 4.0 document-adaptive OCR for body text recovery"
    }
  ]
}
```

---

## Phase 4: TARGETED REMEDIATION (parallel agents, max 3 per wave)

### Agent Assignment Rules
- **Max 2-3 docs per agent prompt** (avoid "prompt too long" errors)
- **All agents run in foreground** (no background execution)
- **Sonnet mandatory** for all agents (Haiku BANNED from entire HDARP pipeline)

### Type A Agents: Enrichment Extraction
```
Task: Read chunk PDFs, extract ONLY equations/tables/figures
Input: chunk PDFs in HDARP_Processing/{doc}/chunks/
Output: Overwrite placeholder files in Knowledge_Base/{doc}/{equations,tables,figures}/
Skip: body_text (already has real content)
```

### Type B Agents: Full Recovery
```
Task: Full 4-type HDARP extraction from chunk PDFs
Input: chunk PDFs
Output: All 4 content types in Knowledge_Base/{doc}/
Process: Same as /sphdarp but targeted at specific docs
```

### Type C Agents: Stub Replacement
```
Task: Re-extract specific chunks that have placeholder body_text
Input: specific chunk PDFs (by number)
Output: Replace stub body_text files
```

### Type D Agents: OCR Recovery (Sraffa 4.0) — v2.0
```
Task: Extract body text from scanned chunk PDFs using Sraffa 4.0 document-adaptive OCR
Tool: sraffa40_processor.py
Max docs per agent: 1-2 (GPU-accelerated, ~5-15s per page)
Max chunks per agent: 10 (prevents timeout)

Workflow per chunk:
1. Import sraffa40_processor.Sraffa40Processor
2. Classify chunk pages (digital / scanned / mixed)
3. Digital pages: PyMuPDF extraction (instant)
4. Scanned/mixed pages: EasyOCR GPU → Agent QA → Chandra 2 if qa_fail_escalate
5. Write body_text file to Knowledge_Base/{doc_id}/body_text/
6. Write page_manifest_chunk_N.json with per-page QA results

After all chunks for a doc:
- Assemble FULL_TEXT.md from all chunk text files
- Build unified page_manifest.json
- Calculate avg confidence across chunks
- Classify: OCR_RECOVERED (avg conf > 0.80), OCR_PARTIAL (0.50-0.80), OCR_FAILED (<0.50)
- Attempt enrichment (eq/tab/fig) on successfully OCR'd chunks

Timeout: 120s per chunk. If exceeded, skip chunk and flag.
```

### Content-Filter Handling
When a chunk is content-filter blocked during remediation:
1. Log the chunk number and document
2. Attempt Sraffa 4.0 OCR fallback
3. If OCR succeeds: mark as CF_RECOVERED
4. If OCR fails: mark as CF_BLOCKED, document in report
5. NEVER leave a CF-blocked chunk undocumented

---

## Phase 5: REASSESSMENT

After remediation, re-scan all remediated documents:

1. Re-count files in all 4 subdirectories
2. Re-classify body text quality
3. Re-calculate enrichment rates (before/after)
4. Re-assign verdicts

### Before/After Metrics (MANDATORY)

```markdown
## Remediation Results

| Document | Metric | Before | After | Delta |
|----------|--------|--------|-------|-------|
| Chiang | Body Text | HOLLOW | SUBSTANTIAL | +100% |
| Chiang | Eq Rate | 0% | N/A (CF) | — |
| Setterfield | Body Text | HOLLOW | SUBSTANTIAL | +100% |
```

### OCR Recovery Metrics (Type D documents — v1.1)

For documents remediated via Sraffa 4.0 OCR, include additional metrics:

```markdown
### OCR Recovery Results
| Document | Chunks OCR'd | Avg Confidence | Method | Total Chars | Classification |
|----------|-------------|----------------|--------|-------------|----------------|
| Berndt   | 61          | 0.957          | sraffa40_ocr | 1,245,000 | OCR_RECOVERED |
| Fernandes| 37          | 0.823          | sraffa40_ocr | 456,000   | OCR_RECOVERED |

Classification: OCR_RECOVERED (conf > 0.80), OCR_PARTIAL (0.50-0.80), OCR_FAILED (<0.50)
```

### Verdict Upgrades
- FAIL → PASS: Document was remediated and now passes
- FAIL → PASS_WITH_WARNINGS: Partially remediated (e.g., CF-blocked enrichment, OCR_PARTIAL)
- HOLLOW → SUBSTANTIAL: OCR recovered body text successfully
- No downgrade should occur from remediation

---

## Phase 6: CATALOG SYNC

### BATCH_STATE.json Updates
```python
for batch_id in remediated_batches:
    state["batches"][batch_id]["status"] = "VERIFIED"
    state["batches"][batch_id]["validation_date"] = now()
    state["batches"][batch_id]["validation_notes"] = "Enrichment audit + remediation. {summary}"
```

### HDARP_MASTER_CATALOG.csv Updates
For each remediated document:
- Update `hdarp_status` to reflect new state
- Update `quality_score` if applicable
- Add enrichment rate columns if not present

### Campaign Tracker Update
Add session log entry to HDARP_CAMPAIGN_TRACKER.md with:
- Target scope
- Documents audited and remediated
- Before/after metrics summary
- Final verdict counts

---

## Key Design Principles

1. **Foreground only**: All agents run in foreground (blocking mode). Never use `run_in_background: true`.
2. **Max 2-3 docs per agent**: Prevents "prompt too long" errors.
3. **Sonnet Mandatory**: All agents use Sonnet. Haiku BANNED from entire HDARP pipeline.
4. **Zero tolerance for undocumented placeholders**: Every placeholder must be classified (legitimate vs failure).
5. **Content-filter blocks get documented**: Even if OCR succeeds, the CF-blocked status is recorded.
6. **Before/after metrics mandatory**: Every remediation report must show delta metrics.
7. **Non-destructive**: Never delete existing real content. Only overwrite placeholders.

---

## Canonical Paths

| Resource | Path |
|----------|------|
| BATCH_STATE | {Project}/Technical/HDARP_Processing/BATCH_STATE.json |
| Master Catalog | {Project}/Technical/HDARP_MASTER_CATALOG.csv |
| Campaign Tracker | {Project}/Technical/HDARP_CAMPAIGN_TRACKER.md |
| Knowledge_Base | {Project}/Knowledge_Base/ |
| Sraffa 4.0 Processor | sraffa40_processor.py |
| Sraffa 4.0 Protocol | SRAFFA_4_PROTOCOL.md |
| PDF Splitter | pdf_splitter_orchestrator.py |

---

**Command Version**: 6.3
**Status**: PRODUCTION READY
**Created**: 2026-03-15
**Updated**: 2026-06-13
**HDARP Protocol**: v6.3
**v2.0**: Sraffa 4.0 document-adaptive OCR for Type D remediation (EasyOCR GPU + Agent QA + Chandra 2 escalation), replaces Sraffa 3.0 multi-engine consensus
**Key**: Audit existing output, classify failures, targeted remediation (including OCR), before/after metrics

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): added Type E native RDB metadata backfill. -->
