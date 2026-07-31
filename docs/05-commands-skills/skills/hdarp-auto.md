---
name: hdarp-auto
description: "Automated HDARP pipeline using VLM-based extraction with Sraffa 4.0 document-adaptive OCR."
when-to-use: '"User wants automated PDF extraction, or needs to process PDFs with the VLM OCR pipeline"'
search-hints: "hdarp auto vlm olmocr chandra automated pipeline sraffa"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[pdf_path|batch_id]"
requires: hdarp-chunker
part-of: HDARP Framework v6.3
---

# autohdarp Skill

## Description
Automated HDARP pipeline using **Sraffa 4.0** document-adaptive OCR. Each page is classified independently: digital pages get instant PyMuPDF extraction, scanned pages get EasyOCR GPU with mandatory Agent QA, and only pages that fail QA are escalated to Chandra 2 (NF4 on RTX 3080). Three rounds of benchmarking proved that Chandra Solo outperforms all consensus approaches, but Sraffa 4.0 reserves Chandra for escalation only to optimize speed and GPU usage.

## Trigger
User requests:
- `/autohdarp` command
- VLM-based document extraction
- Automated HDARP processing
- Cloud batch OCR processing

## Components

### Core Scripts (scripts)
- `hdarp_auto_orchestrator.py` - Main pipeline orchestrator
- `content_detector.py` - Content type detection (Cyrillic, tables, equations, figures)
- `model_router.py` - Model selection and fallback routing
- `quality_scorer.py` - Extraction quality scoring

### Configuration
- `pipeline_config.json`

### Command Definition
- `autohdarp.md`

## Pipeline Flow (Sraffa 4.0)

1. **Per-Page Classification**
   - Classify each page: digital / scanned / mixed
   - Algorithm: text_density + word_quality_check via PyMuPDF
   - Cyrillic detection (U+0400-U+04FF) for language routing

2. **Engine Routing**
   - Digital pages → PyMuPDF `get_text()` (100% accurate, instant)
   - Scanned/mixed pages → EasyOCR GPU (mandatory `gpu=True`)
   - Content type detection: tables, equations, figures → DARP extraction

3. **Agent QA (scanned pages only)**
   - Agent reads EasyOCR output + views page image
   - Judges: qa_pass / qa_pass_with_notes / qa_fail_escalate
   - Chandra 2 invoked only for qa_fail_escalate pages

4. **Output Generation**
   - FULL_TEXT.md with provenance headers and page boundaries
   - page_manifest.json with per-page classification, confidence, QA status
   - HDARP-compatible structure: tables CSV, equations LaTeX, figures markdown

## Model Configuration

### Primary (Local): Chandra OCR 2 (5B)
- Checkpoint: `datalab-to/chandra-ocr-2`
- VRAM: ~3.2 GB (NF4) or ~9.7 GB (BF16)
- Environment: `chandra2` (Python 3.11, CUDA 12.8)
- **Consensus v2 benchmark (2026-04-30)**: Lowest overall CER (0.3408), wins 11/17 pages
- Best on: English tables (0.0494 CER), Cyrillic (0.0682 CER, 99.6% preservation), old scans (0.0056 CER)
- Weakness: Slow (318s/page avg at 300 DPI), high CER on large academic scans
- **Role**: Escalation engine in sraffa40_processor.py (Sraffa 4.0 document-adaptive pipeline)

### Cloud Fallback: olmOCR-7B
- Checkpoint: `allenai/olmOCR-7B-0225-preview`
- VRAM: ~16GB (bfloat16) -- requires cloud GPU (RunPod A100/H100)
- Scores: Tables 84.5, Cyrillic 88.3, Equations 70.0
- Use when: batch throughput needed or Chandra unavailable

### Deprecated: Chandra 1 (9B)
- `chandra/chandra-ocr-9b` -- replaced by Chandra 2 (5B), which is smaller, faster, and more accurate

## Usage Examples

```bash
# Single document
/autohdarp path/to/document.pdf

# Batch processing
/autohdarp path/to/directory/ --batch

# Custom threshold
/autohdarp document.pdf --threshold 0.90

# Disable fallback
/autohdarp document.pdf --no-fallback
```

## Deployment Options

### Local (RTX 3080 10GB)
- Chandra 2 with NF4 quantization: ~229s/page
- Suitable for small batches and quality-critical documents
- Setup: `CHANDRA2_LOCAL_SETUP.md`

### Cloud (RunPod A100-80GB or H100-80GB)
- olmOCR-7B or Chandra 2 in BF16: ~30s per chunk
- Cost: ~$1.60-3.25 per 100 chunks
- Better for large batch processing

## Related Skills
- `/preparehdarp` - Density-aware PDF chunking
- `/phdarp` - Parallel HDARP with agents
- `/sphdarp` - Smart parallel with batch state
- `/hdarp-cleanup` - Clean processed artifacts

## Implementation Status

Current implementation:
- ✅ **Sraffa 4.0 Processor**: `sraffa40_processor.py` — document-adaptive routing
- ✅ Per-page classification: digital / scanned / mixed
- ✅ PyMuPDF digital extraction (instant, 100%)
- ✅ EasyOCR GPU mandatory for scanned pages
- ✅ Chandra 2 local inference (NF4 on RTX 3080) — escalation only
- ✅ Agent QA protocol on every scanned page
- ✅ FULL_TEXT.md assembly with provenance
- ✅ page_manifest.json with per-page QA metadata
- ✅ Consensus engine retired (3 rounds of benchmarks proved single-engine superior)
- ✅ Thunderdome v2 + Consensus v2 benchmarks validated
- ✅ `/fullread` standalone text extraction command
- ⏳ Cloud vLLM integration (placeholder)

## Consensus v2 Final Rankings (2026-04-30)

| Rank | Candidate | Avg CER | Page Wins (of 17) |
|------|-----------|---------|-------------------|
| 1 | **Chandra Solo** | **0.3408** | **11** |
| 2 | EasyOCR Raw | 0.3711 | 1 |
| 3 | Word Consensus (v2-B) | 0.3862 | 1 |
| 4 | PaddleOCR Raw | 0.4520 | 1 |
| 5 | Old Line Consensus (v1) | 0.4911 | 0 |
| 6 | Tesseract Raw | 0.5967 | 4 |
| 7 | Paragraph Consensus (v2-A) | 0.6334 | 3 |

All consensus approaches perform worse than Chandra Solo. Consensus is retired.

## Benchmark Reference
- **Consensus v2**: `consensus_v2_results.md`
- Thunderdome v2: `CHANDRA2_VS_SRAFFA_THUNDERDOME_V2.md`
- v1 report: `CHANDRA2_VS_SRAFFA_THUNDERDOME.md`
- Ground truth: `ground_truth`
- Raw scores: `all_scores.json`
- Setup: `CHANDRA2_LOCAL_SETUP.md`

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
<!-- HDARP Framework v6.3 (2026-06-13): native RDB enrichment capture added. -->
