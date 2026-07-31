---
description: "Sraffa 4.0 Document-Adaptive OCR: Extract text from PDFs using PyMuPDF (digital), EasyOCR GPU (scans), Agent QA, Chandra 2 escalation"
allowed-tools: Bash, Read, Write, Edit, Glob, Grep
argument-hint: "<pdf_path> [output_path] | --batch <dir> | --chunks <doc_id>"
---

**HDARP Framework v6.3** — see `VERSION_REGISTRY.md`

> **Native RDB metadata capture (v6.3):** Sraffa is body-text OCR and normally emits **no** sidecar.
> **Only if** an OCR pass reconstructs a table into a CSV, emit one `RDB_METADATA*.jsonl` line for it
> with `transcription_status="R"` (reconstructed) and honest `field_basis` (OCR'd values are
> `agent_inferred` unless verbatim-legible). Spec:
> `NATIVE_ENRICHMENT_CONTRACT.md`.

# Sraffa 4.0 OCR Command

**Command**: /sraffa-ocr
**Version**: 6.2
**Created**: 2026-03-18
**Updated**: 2026-04-30

## What This Does

Invokes the **Sraffa 4.0** document-adaptive OCR system. Classifies each page as digital/scanned/mixed and routes to the appropriate engine. Digital pages get instant PyMuPDF extraction. Scanned pages get EasyOCR GPU with mandatory Agent QA. Only pages that fail QA get escalated to Chandra 2.

Replaces Sraffa 3.0 multi-engine consensus (retired after three rounds of benchmarks).

## Usage

```
/sraffa-ocr document.pdf                    # OCR single PDF, save to _OCR_Only/
/sraffa-ocr document.pdf output/            # OCR single PDF, save to custom dir
/sraffa-ocr --batch dir/                    # OCR all PDFs in a directory (to _OCR_Only/)
/sraffa-ocr --chunks DOC_ID                 # OCR hollow HDARP chunks (to Knowledge_Base)
```

## Modes

### Mode 1: Single PDF

Process one PDF file with Sraffa 4.0. Default output: `{doc_id}`.

**Implementation**:
```bash
python "sraffa40_processor.py" "INPUT.pdf" --output "OUTPUT_DIR" --mode fullread
```

Display results: page classification summary, confidence, method per page, total chars extracted.

### Mode 2: Batch Directory (`--batch dir/`)

Process all PDFs in a directory. Default output: `_OCR_Only` (each PDF gets its own subdirectory).

**Implementation**:
```bash
python "sraffa40_processor.py" "INPUT_DIR/" --output "OUTPUT_DIR" --mode fullread
```

Each PDF gets its own output subdirectory with FULL_TEXT.md + page_manifest.json.

### Mode 3: HDARP Chunks (`--chunks DOC_ID`)

Process hollow chunk PDFs for an HDARP document using Sraffa 4.0. Output goes to the **HDARP Knowledge Base** (not `_OCR_Only/`) since this is remediating HDARP output.

1. Find all chunk PDFs in `HDARP_Processing/{DOC_ID}/chunks/`
2. Skip chunks that already have real content (>500 chars body text)
3. Run Sraffa 4.0 on each hollow chunk (classify → extract → Agent QA → escalate if needed)
4. Write output to `chunk_NNN_body.txt`
5. Write `page_manifest_chunk_NNN.json` for each chunk
6. After all chunks: assemble FULL_TEXT.md and unified page_manifest.json

**Path convention**: Modes 1-2 (standalone OCR) default to `_OCR_Only/`. Mode 3 (HDARP remediation) writes to the main `Knowledge_Base` since it's fixing HDARP output.

---

## Architecture: Sraffa 4.0 Pipeline

```
PDF Input
    │
    ▼
[Phase 1: Per-Page Classification]
    │ For each page: text_density + word_quality_check
    │
    ├── Digital (text_density > 0.80) ──▶ PyMuPDF get_text() → confidence: 1.0
    │
    ├── Scanned (text_density < 0.10) ──▶ EasyOCR GPU (300 DPI)
    │                                         │
    │                                         ▼
    │                                    [Agent QA]
    │                                    ├── qa_pass → Accept
    │                                    ├── qa_pass_with_notes → Accept + notes
    │                                    └── qa_fail_escalate → Chandra 2 (page-only)
    │
    └── Mixed (0.10-0.80) ──▶ Same as Scanned
    │
    ▼
[Phase 5: Assembly]
    FULL_TEXT.md + page_manifest.json + processing_summary.md
```

## Expected Accuracy

| Document Type | Engine | Accuracy | Confidence |
|--------------|--------|----------|------------|
| Digital PDF (text layer) | PyMuPDF | 100% | 1.0 |
| Clean scanned document | EasyOCR GPU | 90-95% | 0.85-0.95 |
| Degraded/aged scan | EasyOCR GPU → Chandra 2 | 95-98% | 0.90-0.95 |
| Heavily damaged scan | Chandra 2 (escalated) | 85-95% | 0.80-0.95 |

## Engine Requirements

| Engine | Package | Status | Notes |
|--------|---------|--------|-------|
| PyMuPDF | `PyMuPDF` (fitz) | Required | Page classification + digital extraction |
| EasyOCR | `easyocr` | Required | Primary OCR for scans (GPU mandatory) |
| Chandra 2 | `chandra-ocr[hf]` + `bitsandbytes` | Optional | Escalation engine (lazy-loaded) |
| PIL/Pillow | `Pillow` | Required | Image processing |
| NumPy | `numpy` | Required | Array operations |
| PyTorch | `torch` | Required | GPU/CUDA support |

Check availability:
```bash
python "sraffa40_processor.py" --help
```

## Output Formats

### FULL_TEXT.md
YAML frontmatter with provenance + per-page text with boundary comments.

### page_manifest.json
Per-page: pdf_type, extraction_method, confidence metrics, qa_status, escalation info.

### processing_summary.md
Extraction statistics, timing, page type distribution.

## Canonical Paths

| Resource | Path |
|----------|------|
| Sraffa 4.0 Processor | `sraffa40_processor.py` |
| OCR Engines | `sraffa30_ocr_engines.py` |
| Protocol Reference | `SRAFFA_4_PROTOCOL.md` |

## Related Commands

| Command | Relationship |
|---------|-------------|
| `/enrichhdarp` | Uses Sraffa 4.0 for Type D (OCR recovery) remediation |
| `/sphdarp` | Forward processing pipeline (Sraffa 4.0 is the OCR backend) |
| `/preparehdarp` | Classifies pages during preparation, pre-extracts digital text |
| `/fullread` | Standalone text extraction using Sraffa 4.0 for entire folders |

---

**Command Version**: 6.1
**Status**: PRODUCTION READY
**Created**: 2026-03-18
**Updated**: 2026-04-30
**Sraffa Protocol**: 4.0 (Document-adaptive: PyMuPDF + EasyOCR GPU + Agent QA + Chandra 2)
**Replaces**: Sraffa 3.0 multi-engine consensus (retired)

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
