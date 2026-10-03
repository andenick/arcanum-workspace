---
description: "State-aware, idempotent PDF preparation for HDARP v6.3 (native-enrichment flag, discovery, diagnosis, remediation, preparation, verification)"
allowed-tools: Bash, Read, Write, Glob, Grep, Agent
argument-hint: "[N | --wave WAVE | --batch BATCH | --folder FOLDER | --diagnose-only | --remediate-only | --force | --resume]"
---

**HDARP Framework v6.3** — see `VERSION_REGISTRY.md`

> **Native RDB metadata capture (v6.3) — enabling step:** preparation writes
> `rdb_native_enrichment: true` into the v6.0 manifest so downstream DARP processors and the
> validator know to emit/expect the per-table `RDB_METADATA*.jsonl` sidecar (doctrine:
> `hdarp-processing.md`; spec: `NATIVE_ENRICHMENT_CONTRACT.md`).
> Preparation itself extracts nothing — it only sets the flag.

<!--
MASTER COMMAND FILE
===================
Command: /prepareHDARP
Version: v6.3
Created: November 30, 2025
Last Updated: April 27, 2026

VERSION HISTORY:
- v1.0 (2025-11-30): Initial creation for HDARP preparation workflow
- v2.0 (2025-12-01): Added full automation implementation
- v4.0 (2025-12-22): Density-aware chunking, Sraffa 4.0 OCR, enhanced manifests
- v4.5 (2026-02-09): Catalog initialization, BATCH_STATE.json integration
- v5.0 (2026-04-03): Sonnet Mandatory, flat 10-page, 10-chunk batching
- v5.1 (2026-04-03): Preflight content-filter routing
- v6.0 (2026-04-27): Complete rewrite. Four-phase state machine (discover → diagnose → remediate → prepare). State-aware, idempotent, handles 15 known failure modes. Python engine delegation.
- v6.3 (2026-06-13): Native-enrichment manifest flag added; Sraffa engine remains 4.0.
-->

# Prepare HDARP Command v6.3

**Command**: `/prepareHDARP`
**Purpose**: State-aware, idempotent PDF preparation for HDARP processing
**Engine**: `preparehdarp_v6_engine.py`

## HDARP Lifecycle Position

This command is step **2** (PREPARE) in the HDARP lifecycle:
1. `/hdarp-campaign` — campaign setup
2. **`/preparehdarp`** — chunk and prepare PDFs
3. `/sphdarp` (or `/phdarp`, `/spdarp`, `/pdarp`) — extract all 4 content types
4. `/hdarp-wrapup` — validate, remediate, document, close out
5. `/hdarp-integrate-pipeline` — catalog, classify, crossref, Robert sync

**After preparation, the next step is ALWAYS agent extraction via a DARP command — never direct PyMuPDF text dumping.** Preparation creates chunk PDFs and manifests; it does NOT extract content.

## Scope Syntax

```
/prepareHDARP                              # Smart default: scan for work
/prepareHDARP 10                           # Next 10 unprepared PDFs (v5.1 compat)
/prepareHDARP --wave Wave_05               # All docs in a specific wave
/prepareHDARP --wave Wave_05,Wave_06       # Multiple waves
/prepareHDARP --batch BATCH_858-BATCH_900  # Batch range
/prepareHDARP --folder "Inputs/2026.04.08 Books"
/prepareHDARP --diagnose-only              # Phase 0+1 only, no writes
/prepareHDARP --remediate-only             # Phase 0+1+2, no new chunking
/prepareHDARP --force                      # Re-verify even PREPARED batches
/prepareHDARP --resume                     # Resume from last checkpoint
```

## Execution Overview

Four phases, each independently resumable via checkpoint files:

```
PHASE 0: CANONICAL RESOLUTION (always, ~30s)
    ↓
PHASE 1: DISCOVERY + DIAGNOSIS (read-only reconnaissance)
    ↓  user sees summary, confirms
PHASE 2: REMEDIATION (fix known issues)
    ↓
PHASE 3: PREPARATION (chunk new PDFs, assign batches)
    ↓
PHASE 4: VERIFICATION + REPORT
```

---

## PHASE 0: CANONICAL RESOLUTION

Execute these checks before any other work.

### 0a. Resolve BATCH_STATE.json canonical location

Check for `BATCH_STATE_CANONICAL.txt` at project root. If it exists, use the path it contains.

If it does NOT exist, check both locations:
- `{project_root}/BATCH_STATE.json`
- `{project_root}/Technical/HDARP_Processing/BATCH_STATE.json`

If both exist and differ in size by >5%:
1. Run the consolidator to compare:
   ```bash
   python "batch_state_consolidator.py" \
     "{root}/BATCH_STATE.json" "{root}/Technical/HDARP_Processing/BATCH_STATE.json" \
     --report "{root}/Technical/BATCH_STATE_CONSOLIDATION_REPORT.md" --dry-run
   ```
2. Show the user the comparison summary (batch counts, VERIFIED counts, last_updated)
3. Ask user to confirm which is canonical
4. Write `BATCH_STATE_CANONICAL.txt` with the confirmed path
5. Archive the non-canonical copy to `Technical/archive/`

If only one exists, use it and write `BATCH_STATE_CANONICAL.txt`.

### 0b. Workspace capacity check

```bash
powershell -NoProfile -Command "(Get-PSDrive D).Free / 1GB"
```

If < 50 GB free: show WARNING, recommend `/workspace-upkeep` before continuing.

### 0c. Load content-filter patterns

Read `HDARP_CONTENT_FILTER_PATTERNS.md` to load annotation patterns. These patterns populate the `cf_risk_level` field (LOW / MEDIUM / HIGH) in the v6.0 manifest as a processing hint for downstream agents. Pattern matches do **NOT** route documents to the OCR queue and do **NOT** change batch status — annotation only.

### 0d. Parse scope arguments

Translate the user's command arguments into a scope string for the engine:
- `/prepareHDARP` → `""` (smart default)
- `/prepareHDARP 10` → `"10"` (count)
- `/prepareHDARP --wave Wave_05` → `"wave=Wave_05"`
- `/prepareHDARP --batch BATCH_858-BATCH_900` → `"batch=BATCH_858-BATCH_900"`
- `/prepareHDARP --folder "2026.04.08 Books"` → `"folder=2026.04.08 Books"`

---

## PHASE 1: DISCOVERY + DIAGNOSIS

Run the engine's diagnose command:

```bash
python "preparehdarp_v6_engine.py" diagnose \
  --project-root "{project_root}" \
  --batch-state "{canonical_batch_state_path}" \
  --scope "{scope_string}" \
  --output "{project_root}/Technical/preparehdarp_v6_manifest.json"
```

The engine classifies every document in scope into one of 10 states:

| State | Meaning | What Happens |
|---|---|---|
| `CLEAN_PREPARED` | Chunks + v6 manifest + BATCH_STATE=PREPARED | Verify only |
| `STALE_MANIFEST` | Chunks exist, manifest is v3.3/5.0/5.1 | Phase 2: upgrade |
| `CHUNK_MISMATCH` | manifest.chunk_count ≠ filesystem | Phase 2: reconcile |
| `PHANTOM_PENDING` | BATCH_STATE=PENDING, no chunks, no source | Phase 2: deprecate |
| `NEEDS_PREPARATION` | Source PDF exists, no chunks/entry | Phase 3: full prep |
| `CF_RISK_FLAGGED` | Matches content-filter pattern | Annotate manifest with `cf_risk_level=HIGH`, proceed with normal preparation — no status change, no OCR routing |
| `QUARANTINED` | PDF <10KB, unreadable, or 0 pages | Log and exclude |
| `DUPLICATE` | MD5 hash matches existing KB doc | Log and exclude |
| `IN_PROGRESS_PARTIAL` | BATCH_STATE=IN_PROGRESS/PARTIAL_DEFERRED | Skip |
| `ALREADY_PROCESSED` | BATCH_STATE=COMPLETE/VERIFIED/ARCHIVED | Skip |

The engine also detects these 15 failure modes:

| FM | Issue |
|---|---|
| FM-01 | Manifest version mismatch (3.3/5.0/5.1 instead of 6.0) |
| FM-02 | Dual BATCH_STATE.json at different locations |
| FM-03 | Chunk count mismatch (manifest vs filesystem) |
| FM-04 | Phantom PENDING batches (no documents) |
| FM-05 | Catalog-reality divergence (missing catalog rows) |
| FM-06 | Mislabeled PDFs (filename ≠ content) |
| FM-07 | OVERSIZED_CHUNKS: chunk PDFs >5MB after 10-page split (triggers automatic re-chunking cascade) |
| FM-08 | Stale next_to_process pointer |
| FM-09 | Duplicate documents across waves |
| FM-10 | Content-filter false negatives (PREPARED docs missing `cf_risk_level` annotation) |
| FM-11 | Undrained OCR pending queue |
| FM-12 | Multi-agent collision risk |
| FM-13 | Corrupted/truncated PDFs (<10KB) |
| FM-14 | bisect_chunk orphans not in BATCH_STATE |
| FM-15 | 2-digit chunk numbering on 100+ chunk docs |

**After Phase 1**: Display the summary to the user. Show state distribution, failure mode counts, and total documents. If `--diagnose-only` was specified, STOP HERE.

Otherwise, ask: "Phase 1 complete. Proceed with remediation and preparation?"

---

## PHASE 2: REMEDIATION

Run the engine's remediate command (dry-run first if user wants preview):

```bash
# Optional dry-run preview:
python "preparehdarp_v6_engine.py" remediate \
  --manifest "{project_root}/Technical/preparehdarp_v6_manifest.json" --dry-run

# Actual remediation:
python "preparehdarp_v6_engine.py" remediate \
  --manifest "{project_root}/Technical/preparehdarp_v6_manifest.json"
```

The engine applies these remediations:

| FM | Remediation |
|---|---|
| FM-01 | Upgrade manifest in-place (preserve all fields, add v6.0 fields) |
| FM-03 | If actual > manifest: update count. If actual < manifest: demote to PENDING |
| FM-04 | No source PDF → DEPRECATED. Source exists → leave as PENDING for Phase 3 |
| FM-05 | Bulk backfill missing catalog rows from manifest/filesystem evidence |
| FM-07 | Re-chunking cascade: if chunk PDFs >5MB after 10-page split, re-split at 5-page ranges; if still >5MB, try 3-page ranges; quarantine only if a single page exceeds 5MB |
| FM-08 | Recompute next_to_process from current scope's first PREPARED batch |
| FM-10 | Retroactive CF annotation: write `cf_risk_level=HIGH` to manifest for PREPARED docs matching content-filter — no status change, no OCR routing |

All BATCH_STATE.json writes use lock file + backup-before-write protocol.

If `--remediate-only` was specified, STOP after Phase 2.

---

## PHASE 3: PREPARATION

Only runs on documents classified as `NEEDS_PREPARATION`. The engine handles:

```bash
python "preparehdarp_v6_engine.py" prepare \
  --manifest "{project_root}/Technical/preparehdarp_v6_manifest.json" \
  --splitter-path "pdf_splitter_orchestrator.py"
```

For each document:
1. **Content-filter risk check** → annotate manifest with `cf_risk_level` (LOW/MEDIUM/HIGH); proceed with preparation regardless of match — no deferral, no status change
2. **Corrupted PDF check** → QUARANTINED if <10KB or unreadable
3. **Per-page PDF type classification (Sraffa 4.0)**: Classify each page as digital/scanned/mixed using `Sraffa40Processor.classify_page()`. Write `page_classifications` array into manifest. For digital pages, pre-extract text via PyMuPDF and store in `pre_extracted_text/page_{NNN}.txt`.
4. **Chunk via pdf_splitter_orchestrator.py** with `--manifest-version 6.0 --chunk-digits 3`
5. **Write v6.0 manifest** (content_filter_checked, cf_risk_level, preparation_agent, catalog_entry_created, batch_state_updated, **page_classifications**, **pdf_type_summary**)
6. **Batch assignment** (10-chunk / finish-document rule, unchanged from v5.1)
7. **BATCH_STATE.json update** with lock+backup protocol (include `pdf_type_summary` per batch)
8. **Catalog initialization** (new rows in HDARP_MASTER_CATALOG.csv)

### Batch Assignment Rule (unchanged from v5.0)

1. Take documents in preparation order
2. Add each document's chunks to the current batch
3. If the batch reaches ~10 chunks, finish the current document then start a new batch
4. If a single document has >10 chunks, it gets its own exclusive batch(es) of 10 chunks each
5. Always include ALL chunks from a document once you start adding it

### Idempotency

If chunks + valid v6.0 manifest already exist for a document, it is classified as CLEAN_PREPARED and skipped. Re-running `/prepareHDARP` on the same scope produces identical state.

---

## PHASE 4: VERIFICATION + REPORT

```bash
python "preparehdarp_v6_engine.py" verify \
  --manifest "{project_root}/Technical/preparehdarp_v6_manifest.json" \
  --output "{project_root}/Technical/HDARP_v6.0_PREPARATION_REPORT.md"
```

Verification checks:
1. Spot-check 5% of chunks are non-zero-byte files
2. Verify manifest ↔ filesystem concordance for all prepared docs
3. Verify BATCH_STATE integrity (no duplicate IDs, all PREPARED batches have docs)
4. Verify catalog rows exist for all prepared docs

The report has 4 sections: Discovery Summary, Actions Taken, Verification, Next Steps.

Display the Next Steps section to the user with the command to proceed (e.g., `/sphdarp 5 --wave Wave_05`).

---

## Canonical Tools

| Tool | Location |
|---|---|
| **v6.0 Engine** | `preparehdarp_v6_engine.py` |
| **PDF Splitter** | `pdf_splitter_orchestrator.py` (v2.0) |
| **Sraffa 4.0 Processor** | `sraffa40_processor.py` (page classification) |
| **State Consolidator** | `batch_state_consolidator.py` |
| **Content Filter Patterns** | `HDARP_CONTENT_FILTER_PATTERNS.md` |
| **Batch State Protocol** | `BATCH_STATE_PROTOCOL.md` |
| **Catalog Protocol** | `CATALOG_PROTOCOL.md` |
| **Sraffa 4.0 Protocol** | `SRAFFA_4_PROTOCOL.md` |

## Related Commands

- `/sphdarp [N]` — Process PREPARED batches with N parallel agents (smart batching + OCR)
- `/spdarp [N]` — Process PREPARED batches (tables/equations/figures only, no OCR)
- `/enrichhdarp [audit|remediate|full]` — Quality audit and remediation
- `/sraffa-ocr [pdf|--chunks DOC_ID]` — OCR for deferred documents
- `/hdarp-cleanup [doc]` — Remove chunk artifacts after verification
- `/readystart [project]` — Initialize project context

## v6.0 Manifest Schema

```json
{
  "hdarp_version": "6.0",
  "document_name": "...",
  "source_pdf": "...",
  "source_size_mb": 12.5,
  "total_pages": 156,
  "preparation_date": "2026-04-27T...",
  "chunks_created": 16,
  "max_chunk_size_mb": 1.0,
  "max_chunk_pages": 10,
  "chunk_numbering": "3-digit",
  "processing_status": "READY",
  "ready_for_processing": true,
  "content_filter_checked": true,
  "preparation_agent": "preparehdarp_v6.0",
  "catalog_entry_created": true,
  "batch_state_updated": true,
  "avg_density_mb_per_page": 0.08,
  "density_category": "MEDIUM",
  "strategy_used": "SIZE_FIRST",
  "chunks": [...],
  "warnings": [],
  "bisection_history": []
}
```

## Status Vocabulary

| Status | Meaning | Current? |
|---|---|---|
| `CLEAN_PREPARED` | Chunks + v6 manifest + BATCH_STATE=PREPARED | Active |
| `STALE_MANIFEST` | Manifest is pre-v6.0 format | Active |
| `CHUNK_MISMATCH` | manifest.chunk_count ≠ filesystem | Active |
| `PHANTOM_PENDING` | BATCH_STATE=PENDING, no chunks, no source | Active |
| `NEEDS_PREPARATION` | Source PDF exists, no chunks/entry | Active |
| `CF_RISK_FLAGGED` | Matches content-filter pattern; proceeds with preparation, manifest annotated `cf_risk_level=HIGH` | Active |
| `QUARANTINED` | PDF <10KB, unreadable, or single page >5MB | Active |
| `DUPLICATE` | MD5 hash matches existing KB doc | Active |
| `IN_PROGRESS_PARTIAL` | BATCH_STATE=IN_PROGRESS/PARTIAL_DEFERRED | Active |
| `ALREADY_PROCESSED` | BATCH_STATE=COMPLETE/VERIFIED/ARCHIVED | Active |
| `DEFERRED_TO_OCR` | **LEGACY (v5.1)** — formerly used to route CF-matching docs to OCR queue. Replaced by `CF_RISK_FLAGGED` + manifest annotation. Do not emit this state in v6.0. | Legacy |
| `PREFLIGHT_DEFERRED` | **LEGACY (v5.1)** — formerly used during discovery to hold CF-matching docs out of preparation. Replaced by `CF_RISK_FLAGGED`. Do not emit this state in v6.0. | Legacy |

---

**Command Version**: 6.3 Native-Enrichment Enablement + State-Aware + Idempotent + 15 FM Remediation
**Status**: PRODUCTION READY
**Created**: 2025-11-30
**Updated**: 2026-06-13
**HDARP Protocol**: v6.3 (native-enrichment flag, four-phase state machine, Python engine delegation)
**Backward Compatible**: Reads and upgrades v3.3/4.5/5.0/5.1 manifests and BATCH_STATE entries

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native-enrichment enabling flag added; Sraffa engine remains 4.0. -->
