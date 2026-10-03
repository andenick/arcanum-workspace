---
name: hdarp-sraffa-ocr
description: "Sraffa 4.0 document-adaptive OCR: PyMuPDF (digital), EasyOCR GPU (scans), Agent QA, Chandra 2 escalation."
when-to-use: '"Body text extraction during HDARP, standalone /fullread, any PDF text extraction"'
search-hints: "ocr sraffa easyocr chandra digital scan pdf text extraction consensus"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[pdf_path|--chunks DOC_ID]"
requires: none
part-of: HDARP Framework v6.3
---

# Sraffa 4.0 — Document-Adaptive OCR Protocol

## Overview

Sraffa 4.0 is Arcanum's production OCR system. It classifies each page independently and routes it to the appropriate extraction engine. Multi-engine consensus (Sraffa 3.0) is **retired** — three rounds of benchmarks proved single-engine extraction with agent QA is superior.

**Expected Accuracy**: 100% for digital pages, 90-98% for scanned pages (depending on source quality and engine)

---

## THE FOUR MANDATES (NON-NEGOTIABLE)

### Mandate 1: CLASSIFY EVERY PAGE

- Per-page classification: `digital` / `scanned` / `mixed`
- Algorithm: `text_density` + `word_quality_check` via PyMuPDF
- **Never assume an entire PDF is one type** — mixed PDFs are common
- A page with embedded text that is garbled gets classified as `mixed` (treated as scanned)

### Mandate 2: USE THE RIGHT ENGINE FOR EACH PAGE

- **Digital pages**: PyMuPDF `get_text()` — instant, 100% accurate, no OCR needed
- **Scanned/mixed pages**: EasyOCR with `gpu=True` (GPU is **MANDATORY**)
- **Escalated pages only**: Chandra 2 (NF4, lazy-loaded, page-only re-OCR)
- **NEVER** run all engines on every page — Sraffa 3.0 consensus is RETIRED

### Mandate 3: AGENT QA ON EVERY SCANNED PAGE

- After EasyOCR runs, the agent MUST read the output and view the page image
- The agent judges quality and marks each scanned page:
  - `qa_pass` — text is usable
  - `qa_pass_with_notes` — acceptable with minor issues noted
  - `qa_fail_escalate` — significant quality problems → Chandra 2
- This costs tokens — **the user has explicitly approved this expenditure**
- **NO SKIPPING QA** for any reason (context limits, token budgets, time pressure)

### Mandate 4: EXTRACT EVERYTHING — NO EXCUSES

- Every page must be extracted — no skipping "boilerplate"
- No truncation, no "representative samples"
- If context is tight, compact prior output first, then continue
- Record failures honestly in `page_manifest.json` with `extraction_method: "failed"`
- **Never fabricate text** — a gap is always preferable to hallucinated content

---

## Technology Stack

| Engine | When Used | Accuracy | Notes |
|--------|-----------|----------|-------|
| PyMuPDF | Digital pages (text_density > 0.80) | 100% | Embedded text extraction, instant |
| EasyOCR (GPU) | Scanned/mixed pages | 90-95% | Primary OCR, GPU mandatory, `['en','ru']` |
| Chandra 2 (NF4) | Escalated pages only | 95-98% | 5B VLM, lazy-loaded, page-only |

---

## Agent QA Protocol

After EasyOCR processes a scanned/mixed page, the agent evaluates:

### Coherence Check
- Does the text read as natural language?
- Are there garbled words, nonsense character sequences, or missing paragraphs?

### Numeric Accuracy Check
- Are numbers, decimal points, and percentages preserved?
- Do column alignments and row structures look plausible?

### Completeness Check
- Is all visible text from the page captured?
- Compare OCR output length to what the agent can see on the page image

### Known-Hard Pattern Check
- Faded ink, handwriting, unusual fonts?
- Stamps, marginalia, microfilm artifacts?
- Multi-column layouts that may have been read out of order?

### Judgment
Record in `page_manifest.json`: `qa_status`, `qa_notes` (free text), `escalation_reason` (if escalating).

---

## Chandra 2 Escalation

**Trigger**: Agent QA marks page as `qa_fail_escalate`

- **Model**: `datalab-to/chandra-ocr-2` (~4B params)
- **Quantization**: NF4 via BitsAndBytes
- **VRAM**: ~3.2 GB model + ~4.7-6 GB peak (a 10 GB-class consumer GPU)
- **Environment**: `chandra2`
- **Engine**: `chandra_engine.py` (consolidated, v1.1)
- **Inference**: `prompt_type="ocr_layout"`, `max_output_tokens=8192`
- **DPI**: 150 (reduced from 300 to shrink token count ~4x)
- **Image size cap**: 6,291,456 pixels max (prevents KV cache explosion)
- **Language**: Hardcoded to `en` (prevents hallucinations on numeric grids)
- **Lazy loading**: Model is NOT loaded until the first escalation is triggered
- **Scope**: Only the failing page is re-OCR'd — all other pages keep EasyOCR results

---

## Workflow

### Phase 1: Classification

```python
import sys
sys.path.insert(0, 'scripts')
from sraffa40_processor import Sraffa40Processor

processor = Sraffa40Processor()
classification = processor.classify_document("path/to/document.pdf")
# Returns per-page pdf_type, text_density, word_quality
```

### Phase 2: Extraction

```python
result = processor.process_pdf("path/to/document.pdf", mode="fullread")
# Processes each page based on classification
# Digital → PyMuPDF, Scanned/Mixed → EasyOCR GPU
# Returns page_results with qa_status='pending' for scanned pages
```

### Phase 3: Agent QA (scanned pages only)

The agent reads each scanned page's OCR output and the page image. For each:
- Set `qa_status` to `qa_pass`, `qa_pass_with_notes`, or `qa_fail_escalate`
- Write `qa_notes` explaining the judgment

### Phase 4: Chandra 2 Escalation (if qa_fail_escalate)

```python
# Only for pages where agent set qa_fail_escalate
escalated_result = processor.chandra2_escalate(doc, page_num)
# Replace EasyOCR result for that page
```

### Phase 5: Assembly

```python
full_text = processor.assemble_full_text(page_results, mode="fullread", document_meta=result)
manifest = processor.build_page_manifest(page_results, pdf_path, mode="fullread")
processor.write_output("doc_id", full_text, manifest, summary)
```

---

## Output Format

### FULL_TEXT.md
- YAML frontmatter with provenance (document_id, source_pdf, sraffa_version, engines_used)
- Page boundary comments: `<!-- Page N | type: scanned | method: easyocr | confidence: 0.92 -->`
- Body text in reading order
- Inline DARP artifacts in HDARP mode (tables as markdown, equations as LaTeX, figure references)

### page_manifest.json
- Per-page: pdf_type, extraction_method, confidence metrics, qa_status, qa_notes, escalation info
- Summary: page type counts, escalation count, average confidence, total chars

### processing_summary.md
- Extraction statistics, timing, page type distribution

---

## Integration with HDARP

Sraffa 4.0 is the body text extraction component of HDARP:

- **Before Sraffa 4.0**: PDF is chunked using `pdf_splitter_orchestrator.py`; pages classified during `/preparehdarp`
- **Sraffa 4.0 Role**: Extracts body text from each chunk
- **After Sraffa 4.0**: Text is combined with DARP extraction (tables CSV, equations LaTeX, figures MD)
- **Assembly**: `FULL_TEXT.md` with inline DARP references placed in `{doc_id}` (HDARP KB, not `_OCR_Only/`)
- **Catalog Sync (v4.5)**: OCR confidence + QA pass rate recorded in `HDARP_MASTER_CATALOG.csv`

### HDARP Quality Points

In HDARP scoring (27 points max):
- Body Text extraction: 4 points
- OCR / Agent QA: 2 points (Sraffa 4.0 QA pass rate >= 90% of scanned pages)

---

## Canonical Tools

| Tool | Path | Purpose |
|------|------|---------|
| Sraffa 4.0 Processor | `sraffa40_processor.py` | Main OCR orchestrator |
| Chandra 2 Engine | `chandra_engine.py` | Consolidated Chandra 2 VLM OCR (v1.1) |
| Chandra Batch Pipeline | `chandra_batch_pipeline.py` | Standalone bulk OCR |
| OCR Engines (legacy) | `sraffa30_ocr_engines.py` | EasyOCR/Paddle/Tesseract wrappers |
| Protocol Reference | `SRAFFA_4_PROTOCOL.md` | Canonical protocol documentation |

---

## Quick Reference Card

```
┌──────────────────────────────────────────────────────────────┐
│                   SRAFFA 4.0 MANDATES                        │
├──────────────────────────────────────────────────────────────┤
│ 1. CLASSIFY EVERY PAGE — digital / scanned / mixed           │
│ 2. RIGHT ENGINE PER PAGE — PyMuPDF / EasyOCR GPU / Chandra 2│
│ 3. AGENT QA ON EVERY SCANNED PAGE — no exceptions            │
│ 4. EXTRACT EVERYTHING — NO EXCUSES                           │
├──────────────────────────────────────────────────────────────┤
│                   ENGINE ROUTING                             │
├──────────────────────────────────────────────────────────────┤
│ Digital (text_density > 0.80) → PyMuPDF (instant, 100%)      │
│ Scanned (text_density < 0.10) → EasyOCR GPU (90-95%)        │
│ Mixed (0.10-0.80) → EasyOCR GPU (treat as scanned)          │
│ Escalated (qa_fail) → Chandra 2 NF4 (95-98%, page-only)     │
├──────────────────────────────────────────────────────────────┤
│                   RETIRED                                    │
├──────────────────────────────────────────────────────────────┤
│ ✗ PaddleOCR + EasyOCR + Tesseract consensus (Sraffa 3.0)    │
│ ✗ 6-rule line-based adjudication                             │
│ ✗ Paragraph/word-level consensus (v2)                        │
└──────────────────────────────────────────────────────────────┘
```

---

## Skill Metadata

**Created**: 2026-01-01  
**Updated**: 2026-04-30  
**Version**: 4.0 (Sraffa 4.0 — Document-Adaptive OCR)  
**Previous Version**: 4.5 (Sraffa 3.0 — Three Mandates + Consensus — RETIRED)  
**Status**: PRODUCTION  
**Protocol**: Sraffa 4.0  

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
