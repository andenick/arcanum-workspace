---
description: "HDARP Cleanup v4.5: Remove processed chunk artifacts with catalog verification and status update"
allowed-tools: Bash, Read, Write, Glob, Grep
argument-hint: "[document_name]"
---

**HDARP Framework v6.3** — see `VERSION_REGISTRY.md`

# HDARP Cleanup Command v6.2

**Command**: `/hdarp-cleanup [document_name]`
**Purpose**: Manual cleanup of processed/validated HDARP chunk artifacts with catalog sync
**Version**: 6.2
**Created**: 2025-12-23
**Updated**: 2026-02-09

## What's New in v4.5

- **Pre-Cleanup Catalog Verification**: Verifies catalog status before cleanup
- **Post-Cleanup Status Update**: Updates catalog status (VERIFIED → ARCHIVED)
- **Unified v4.5 Standard**: Consistent with all DARP commands

---


## BATCH_STATE Integration

**CRITICAL**: Only clean batches with status VERIFIED in BATCH_STATE.json.

**Location**: {Project}/BATCH_STATE.json

**Before Cleanup**:
1. Read BATCH_STATE.json
2. Verify batch status == "VERIFIED"
3. Only then proceed with cleanup

**Safety Check**:
```python
if state["batches"][batch_id]["status"] != "VERIFIED":
    print(f"ERROR: Cannot clean {batch_id} - status is not VERIFIED")
    return
```

See BATCH_STATE_PROTOCOL.md for full details.

---

## Pre-Cleanup Catalog Verification (v4.5)

**MANDATORY**: Before cleaning any document, verify catalog status.

### Verification Steps

1. **Read HDARP_MASTER_CATALOG.csv**
   ```python
   catalog_path = f"{project}/HDARP_MASTER_CATALOG.csv"
   catalog = pd.read_csv(catalog_path)
   ```

2. **Verify Document Status**
   ```python
   doc_entry = catalog[catalog['document_id'] == document_id]
   if doc_entry['hdarp_status'].values[0] not in ['COMPLETE', 'VERIFIED']:
       print(f"ERROR: Cannot clean {document_id} - status is not COMPLETE/VERIFIED")
       return
   ```

3. **Verify Quality Score**
   ```python
   quality_score = doc_entry['quality_score'].values[0]
   if quality_score < 22:  # v4.5 minimum
       print(f"WARNING: Quality score {quality_score}/27 below threshold")
   ```

---

## Usage

```bash
/hdarp-cleanup                  # Cleanup all validated chunks in project
/hdarp-cleanup Volcker          # Cleanup specific document only
/hdarp-cleanup --dry-run        # Preview what would be deleted (no actual deletion)
```

## What Gets Cleaned Up

**DELETED** (after verification):
- `*.pdf` - Chunk PDFs
- `manifest.json` - Processing manifest

**PRESERVED** (never deleted):
- `{original}.pdf` - Original source PDFs
- `{doc}` - All extracted content
- `chunk_*_PROCESSING_SUMMARY.md` - Processing summaries
- `RDB_METADATA.jsonl` and `RDB_METADATA_chunks_*.jsonl` — **native RDB enrichment sidecars (v6.3). CRITICAL: never delete.** Once the chunk PDFs are gone this honest read-time metadata cannot be re-recovered; the sidecar is the only surviving record.

**Pre-cleanup WARN (v6.3):** if `config.kb.native_enrichment="require"` was in effect (or the doc was extracted under v6.3) and no `RDB_METADATA*.jsonl` sidecar is present, emit a WARNING before deleting chunk PDFs — the metadata can still be backfilled by `enrichhdarp` Type E *only while the chunk PDFs survive*.

---

## Workflow

### Step 1: Discover Documents

```bash
# Find all documents with HDARP processing
ls HDARP_Processing
```

### Step 2: Verify Validation Status

For each document, check ALL chunks passed validation:

1. **Check processing summaries exist**:
   ```bash
   ls chunk_*_PROCESSING_SUMMARY.md
   ```

2. **Verify quality scores** in each summary:
   - v4.0: Score >= 22/27
   - v3.3: Score >= 20/25

3. **Confirm content extraction**:
   - `chunk_*_text.md` exists for each chunk
   - Tables, equations, figures extracted as applicable

### Step 3: Safety Check Report

Generate cleanup eligibility report:

```markdown
## HDARP Cleanup Eligibility Report

### Document: {document_name}

| Chunk | Text | Summary | Score | Status |
|-------|------|---------|-------|--------|
| 01 | YES | YES | 25/27 | ELIGIBLE |
| 02 | YES | YES | 24/27 | ELIGIBLE |
| 03 | NO | YES | 23/27 | BLOCKED |

**Eligible for cleanup**: 2/3 chunks
**Blocked**: 1 chunk (missing text file)
```

### Step 4: Execute Cleanup

Only cleanup ELIGIBLE chunks:

```bash
# For each eligible chunk
rm chunk_{N}*.pdf

# After ALL chunks cleaned for a document
rm manifest.json
rmdir chunks
rmdir {doc}
```

### Step 5: Generate Cleanup Report

```markdown
## HDARP Cleanup Report

**Date**: {timestamp}
**Document**: {document_name}

### Actions Taken
- Deleted: 5 chunk PDFs (12.3 MB total)
- Deleted: 1 manifest.json
- Removed: HDARP_Processing/{doc}/ directory

### Preserved
- Original: {document}.pdf
- Knowledge Base: {doc} (all content intact)

### Status: CLEANUP COMPLETE
```

### Step 6: Update Catalog Status (v4.5)

After successful cleanup, update HDARP_MASTER_CATALOG.csv:

```python
# Update catalog entry
catalog.loc[catalog['document_id'] == document_id, 'hdarp_status'] = 'ARCHIVED'
catalog.loc[catalog['document_id'] == document_id, 'archived_date'] = datetime.now().isoformat()
catalog.loc[catalog['document_id'] == document_id, 'artifacts_cleaned'] = True
catalog.to_csv(catalog_path, index=False)
```

**Status Transition**: VERIFIED → ARCHIVED

---

## Safety Protocols

### DO NOT Cleanup If:

1. **Any chunk failed validation** (score below minimum)
2. **Missing text files** in Knowledge_Base
3. **Missing processing summaries**
4. **Original PDF not in **

### Version Detection

```python
def get_min_score(manifest):
    # Check for v4.0 indicator
    if 'density_mb_per_page' in manifest:
        return 22, 27  # v4.0: 22/27 minimum
    return 20, 25  # v3.3: 20/25 minimum
```

---

## Dry Run Mode

Use `--dry-run` to preview cleanup without deleting:

```bash
/hdarp-cleanup --dry-run
```

Output:
```
DRY RUN - No files will be deleted

Would cleanup:
- chunk_01_pages_1-10.pdf (0.8 MB)
- chunk_02_pages_11-20.pdf (0.7 MB)
- manifest.json (2 KB)

Total: 3 files, 1.5 MB

To execute: /hdarp-cleanup Volcker
```

---

## Integration

- **After /phdarp**: Validator agent cleans individual validated chunks
- **After batch completion**: Use /hdarp-cleanup to clean remaining artifacts
- **Before /handoff**: Run /hdarp-cleanup to minimize workspace size

---

## Error Handling

| Error | Resolution |
|-------|------------|
| "No HDARP_Processing found" | No chunks to cleanup |
| "Document not found" | Check document name spelling |
| "Validation incomplete" | Run /phdarp to validate remaining chunks first |
| "Missing content" | Re-process chunks with missing extractions |

---

**Command Version**: 6.1 Catalog Verification + Status Update
**Status**: PRODUCTION READY
**Created**: 2025-12-23
**Updated**: 2026-02-09
**HDARP Protocol**: v6.3
**New in v4.5**: Pre-cleanup catalog verification, post-cleanup ARCHIVED status update

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
