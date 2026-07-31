---
name: hdarp-chunker
description: "Density-aware PDF chunking for HDARP using pdf_splitter_orchestrator.py. Flat 10-page max chunks with density metrics."
when-to-use: '"User needs to split or chunk a PDF for HDARP processing, especially large PDFs with mixed content density"'
search-hints: "pdf split chunk density hdarp prepare large document chunker"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[pdf_path] [options]"
requires: none
part-of: HDARP Framework v6.3
---

# PDF Density Splitter - Intelligent Density-Aware PDF Chunking for HDARP v6.2

## Description

This skill wraps the pdf_splitter_orchestrator.py canonical tool to provide intelligent, density-aware PDF chunking. It analyzes each PDF's MB/page ratio to select the optimal chunking strategy, achieving 100% compliance with HDARP size limits. In v4.5, also initializes catalog entries and BATCH_STATE.

## When to Use

- Preparing PDFs with complex layouts for HDARP processing
- PDFs with mixed content density (text-heavy and image-heavy pages)
- Large PDFs that need chunking before /phdarp or /phdarpsmart
- When legacy split_pdf.py fails to produce compliant chunks

## Prerequisites

- PDFs in project's Inputs folder
- Project initialized with /readystart
- Python with PyPDF2 installed

## Workflow

### Step 1: Analyze PDF Density

```python
import sys
sys.path.insert(0, 'scripts')
from pdf_splitter_orchestrator import PDFSplitterOrchestrator

orchestrator = PDFSplitterOrchestrator()

# Analyze single PDF
analysis = orchestrator.analyze_pdf("path/to/document.pdf")
print(f"Size: {analysis['size_mb']:.2f} MB")
print(f"Pages: {analysis['pages']}")
print(f"Density: {analysis['density_mb_per_page']:.4f} MB/page")
print(f"Category: {analysis['density_category']}")
```

### Step 2: Understand Density Categories

| Category | MB/Page | Strategy | Max Pages | Max Size |
|----------|---------|----------|-----------|----------|
| LOW | <0.05 | PAGE-FIRST | 10 | 1MB |
| MEDIUM | 0.05-0.10 | SIZE-FIRST | 10 | 0.75MB |
| HIGH | >0.10 | SIZE-FIRST | 10 | 0.5MB |

### Step 3: Run Chunking

```python
result = orchestrator.split_pdf(
    input_pdf="path/to/document.pdf",
    output_dir="chunks"
)

print(f"Chunks created: {len(result['chunks'])}")
print(f"Strategy used: {result['strategy']}")
print(f"Warnings: {result['warnings']}")
```

### Step 4: Handle Oversized Single Pages

HDARP v4.0 accepts single pages >1MB with warnings:

```python
if result['warnings']:
    for warning in result['warnings']:
        print(f"WARNING: {warning}")
    # Warnings are informational - chunks are still valid
```

## Expected Outputs

- `chunks/chunk_001_pages_001-010.pdf` (or similar)
- Per-chunk files ≤1MB (or flagged if single-page exception)
- Ready for HDARP processing

## Integration with Other Skills

- **Before**: Run after /readystart sets project context
- **After**: Run /preparehdarp which uses this skill, then /phdarp

## Critical Protocols (HDARP v4.5)

1. **Density-First Strategy**: Always analyze before chunking
2. **Retry Logic**: 10→5→3→2→1 pages until compliant
3. **Single-Page Exception**: Accept >1MB single pages with WARNING
4. **Manifest Fields**: Include density metrics in v4.5 manifests
5. **Catalog Init (v4.5)**: Initialize HDARP_MASTER_CATALOG.csv entry
6. **BATCH_STATE (v4.5)**: Update BATCH_STATE.json with document info

## Common Issues and Solutions

### Issue: Chunk exceeds 1MB
**Solution**: Orchestrator automatically retries with smaller page counts. Single pages >1MB are accepted with warnings.

### Issue: Import error for orchestrator
**Solution**: Ensure path is correct: `pdf_splitter_orchestrator.py`

## Canonical Tools

- **PDF Splitter Orchestrator**
  - Path: `pdf_splitter_orchestrator.py`
  - Purpose: Density-aware adaptive chunking with 100% compliance

## Skill Metadata

**Created**: 2025-12-22
**Updated**: 2026-02-09
**Version**: 6.2
**HDARP Protocol**: v6.3
**Status**: PRODUCTION
**New in v4.5**: Catalog initialization, BATCH_STATE integration

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
