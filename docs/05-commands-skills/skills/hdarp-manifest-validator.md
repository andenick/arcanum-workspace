---
name: hdarp-manifest-validator
description: "Validate HDARP manifest files for density metrics, chunk compliance, and processing readiness."
when-to-use: '"User wants to verify HDARP preparation is correct before processing, or troubleshoot processing failures"'
search-hints: "hdarp manifest validate verify compliance chunk ready"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[batch_id]"
requires: hdarp-chunker
part-of: HDARP Framework v6.3
---

# HDARP Manifest Validator - Verify v6.2 Manifest Compliance

## Description

This skill validates HDARP v4.5 manifest files to ensure chunks are ready for processing. It checks density metrics, chunk compliance, schema completeness, and BATCH_STATE integration before running /phdarp or /sphdarp.

## When to Use

- After /preparehdarp to verify preparation succeeded
- Before starting a processing campaign with /phdarp or /sphdarp
- When troubleshooting chunk processing failures
- To audit existing projects for v4.5 compliance
- To verify BATCH_STATE.json is properly initialized (v4.5)

## Prerequisites

- Project prepared with /preparehdarp
- manifest.json files in each document's HDARP_Processing directory

## Workflow

### Step 1: Locate Manifests

```python
from pathlib import Path
import json

# Find all manifests
manifests = list(Path("HDARP_Processing").glob("*/manifest.json"))
print(f"Found {len(manifests)} manifest files")
```

### Step 2: Validate Each Manifest

```python
def validate_manifest(manifest_path):
    with open(manifest_path) as f:
        manifest = json.load(f)
    
    issues = []
    version = manifest.get('hdarp_version', '3.3')
    
    # Check v4.0 required fields
    if version == '4.0':
        required_v4 = [
            'density_mb_per_page',
            'density_category', 
            'strategy_used',
            'warnings'
        ]
        for field in required_v4:
            if field not in manifest:
                issues.append(f"Missing v4.0 field: {field}")
    
    # Check chunk compliance
    chunks_created = manifest.get('chunks_created', 0)
    if chunks_created == 0:
        issues.append("No chunks created")
    
    # Check processing status
    status = manifest.get('processing_status')
    if status not in ['CHUNKED', 'COPIED']:
        issues.append(f"Invalid status: {status}")
    
    # Check readiness
    if not manifest.get('ready_for_processing'):
        issues.append("Not marked ready for processing")
    
    return {
        'path': str(manifest_path),
        'version': version,
        'chunks': chunks_created,
        'issues': issues,
        'valid': len(issues) == 0
    }
```

### Step 3: Generate Validation Report

```python
results = []
for manifest_path in manifests:
    result = validate_manifest(manifest_path)
    results.append(result)
    
    status = "VALID" if result['valid'] else "ISSUES"
    print(f"[{status}] {result['path']}: v{result['version']}, {result['chunks']} chunks")
    
    if result['issues']:
        for issue in result['issues']:
            print(f"  - {issue}")

# Summary
valid_count = sum(1 for r in results if r['valid'])
print(f"\nSummary: {valid_count}/{len(results)} manifests valid")
```

## Validation Checklist

### v4.5 Required Fields

| Field | Type | Description |
|-------|------|-------------|
| `hdarp_version` | string | Must be "4.5" |
| `density_mb_per_page` | float | MB per page ratio |
| `density_category` | string | LOW, MEDIUM, or HIGH |
| `strategy_used` | string | PAGE_FIRST, SIZE_FIRST, or COPY |
| `oversized_single_pages` | int | Count of >1MB single pages |
| `warnings` | array | List of warning messages |
| `catalog_entry_created` | boolean | (v4.5) Catalog entry initialized |
| `batch_state_updated` | boolean | (v4.5) BATCH_STATE.json updated |

### Chunk Compliance

| Check | Requirement |
|-------|-------------|
| Size | Each chunk ≤1MB (or flagged as OVERSIZED_SINGLE_PAGE) |
| Count | chunks_created > 0 |
| Status | processing_status = CHUNKED or COPIED |
| Ready | ready_for_processing = true |

### Version Detection

```python
def get_version(manifest):
    # v4.0 manifests have density metrics
    if 'density_mb_per_page' in manifest:
        return '4.0'
    return '3.3'
```

## Expected Outputs

- Validation report (console or file)
- List of valid manifests
- List of manifests with issues

## Integration with Other Skills

- **Before**: /preparehdarp creates manifests
- **After**: /phdarp uses validated manifests

## Critical Protocols (HDARP v4.5)

1. **All v4.5 fields must be present** for v4.5 processing
2. **Warnings are informational** - don't fail validation
3. **Version detection** based on density_mb_per_page field
4. **Quality thresholds** differ by version (27 vs 25 points)
5. **BATCH_STATE validation** - verify batch entry exists (v4.5)
6. **Catalog validation** - verify HDARP_MASTER_CATALOG.csv entry (v4.5)

## BATCH_STATE Validation (v4.5)

```python
# Verify BATCH_STATE.json is properly initialized
batch_state_path = f"{project}/BATCH_STATE.json"
if Path(batch_state_path).exists():
    with open(batch_state_path) as f:
        state = json.load(f)
    if document_id in state.get('batches', {}).get(current_batch, {}).get('documents', []):
        print(f"[VALID] Document in BATCH_STATE: {document_id}")
    else:
        issues.append("Document not in BATCH_STATE")
else:
    issues.append("BATCH_STATE.json not found")
```

## Common Issues and Solutions

### Issue: Missing density_mb_per_page
**Solution**: Re-run /preparehdarp with v4.0 canonical tools

### Issue: processing_status = ERROR
**Solution**: Check preparation logs, may need manual chunking

### Issue: chunks_created = 0
**Solution**: Verify PDF exists and is readable

## Post-Processing Validation (added 2026-05-06)

After SPHDARP processing completes, this validator can also verify extraction integrity:

### 4-Type Completeness Check

For each document in the KB:
1. `CSV_Tables/` exists with content or `_no_tables.txt`
2. `equations/` exists with content or `_no_equations.txt`
3. `figures/` exists with content or `_no_figures.txt`
4. Body text has HDARP chunk markers (`<!-- chunk_NNN -->`)

### Scrounger Standard Compliance (per `sphdarp-scrounger.md`)

Any chunk with zero extraction (no body text, no tables, no equations, no figures) must have a documented acceptable reason:
- **DUPLICATE**: confirmed identical content already in KB (MD5 match)
- **CF_EXHAUSTED**: content filter hit after full L1 bisect + L2 scholarly framing on every page
- **QUARANTINED**: file physically unreadable (0 bytes, crashes every tool)

Flag any zero-extraction chunks without one of these documented reasons as INVALID → needs reprocessing.

### PyMuPDF Dump Detection

Flag as INVALID if:
- KB has only `Text/` + `FULL_TEXT.md` with no structured directories
- Body text files are `chunk_NNN_body.txt` with raw text and `[PAGE NEEDS OCR]` markers
- FULL_TEXT.md named `PYMUPDF_TEXT_DUMP.md` (explicitly relabeled)

## Skill Metadata

**Created**: 2025-12-22
**Updated**: 2026-05-06
**Version**: 6.2
**HDARP Protocol**: v6.3
**Status**: PRODUCTION
**New in v4.6**: Post-processing 4-type completeness check, PyMuPDF dump detection
**v4.5**: BATCH_STATE validation, catalog entry verification

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
