---
name: hdarp-extract
version: "6.2"
description: "Core HDARP extraction protocol. Extracts 4 content types (Tables→CSV, Equations→LaTeX, Figures→Markdown, Body Text→Sraffa 4.0 OCR) from PDF chunks with 98%+ accuracy. Includes single-chunk, batch, and full-document workflows."
when-to-use: '"User needs to extract structured content from PDFs via the cloud Claude Read-tool HDARP pipeline — tables, equations, figures, body text. Invoked by all DARP commands (/pdarp, /phdarp, /spdarp, /sphdarp). For OFFLINE local-VLM extraction with the same 4-artifact output, use /hopper instead."'
search-hints: "hdarp darp pdf extract table csv equation latex figure ocr sraffa chunk batch process cloud"
allowed-tools: Read, Write, Bash, Glob, Grep, Edit
argument-hint: "[chunk_number|batch_id|document_path]"
requires: hdarp-chunker
part-of: HDARP Framework v6.3 (cloud Claude Read-tool — distinct from /hopper local VLM engine)
---

> **Scope**: this is the cloud Claude Read-tool HDARP pipeline. `/hopper` (Hopper Line v2, local GPU) produces the same 4-artifact KB shape OFFLINE without API calls. Never call them by each other's names.

# HDARP Extract — Core Extraction Protocol v6.2

Extract structured content from PDF chunks using the Hybrid Direct Agent Reading Protocol. Produces 4 content types per chunk with 98%+ accuracy for tables and 95-100% for body text via Sraffa 4.0 document-adaptive OCR.

This is the consolidated extraction skill — invoked by all DARP commands and usable standalone for single-chunk processing.

---

## Mandatory Output (ALL 4 types per chunk)

Every chunk MUST produce evidence of all 4 content types:

| Type | Output Format | Output Path |
|------|--------------|-------------|
| **Tables** | CSV (one sheet per file) | `CSV_Tables/chunk_NN_table_MM.csv` or `_no_tables.txt` |
| **Equations** | LaTeX | `equations/equations_chunks_NNN_NNN.tex` or `_no_equations.txt` |
| **Figures** | Markdown (200+ word descriptions) | `figures/figures_chunks_NNN_NNN.md` or `_no_figures.txt` |
| **Body Text** | Sraffa 4.0 OCR | `FULL_TEXT_chunks_NNN_NNN.md` with chunk boundary markers |

**A chunk processed for body text only is NOT complete HDARP.** PyMuPDF `page.get_text()` is a triage tool, not an extraction method.

**Never give up on a chunk.** The only acceptable non-extraction outcomes are:
- **DUPLICATE**: Confirmed identical content in another batch (MD5 verified)
- **CF_EXHAUSTED**: Content filter after full L1/L2/L3 retry ladder on every page
- **QUARANTINED**: Physically unreadable (corrupt file, zero pixels)

See `sphdarp-scrounger.md` for the complete retry behavior tree.

---

## Single Chunk Processing

```
/hdarp {chunk_number}
/hdarp              # Show status and next chunk
```

### Step 1: Locate Chunk & Detect Version

Read `manifest.json` to detect HDARP version and chunk metadata:
- All processing uses 27-point scoring
- Verify chunk size limits (<=10 pages, <=1 MB)

### Step 2: Extract All 4 Content Types

#### A. Tables → CSV (DARP Agent Vision — 98%+ accuracy)
- Read chunk PDF with agent vision
- Identify ALL tables in chunk
- Extract complete cell values with structural integrity
- Save each table: `CSV_Tables/chunk_{N}_table_{M}.csv`

#### B. Equations → LaTeX (100% accuracy target)
- Identify all numbered equations in chunk
- Transcribe to LaTeX with exact symbols
- Include variable definitions from surrounding text

#### C. Figures → Markdown (200+ word descriptions)
- Identify all numbered figures (charts, graphs, diagrams)
- Analyze visual content: chart type, axes, scales, trends
- Write 200+ word description

#### D. Body Text → Sraffa 4.0 OCR

**Document-Adaptive Routing**:
- **Digital pages**: PyMuPDF text extraction — instant, 100% accurate
- **Scanned/mixed pages**: EasyOCR GPU → Agent QA → Chandra 2 if escalated
- **Agent QA**: Mandatory on every scanned page. Read output + view page image. Mark qa_pass / qa_fail_escalate / qa_pass_with_notes.

### Step 3: Quality Validation (27-point scoring)

| Component | Points | Criteria |
|-----------|--------|----------|
| Tables Complete | 8 | All tables identified and extracted to CSV |
| Table Accuracy | 4 | Cell values match source (spot check) |
| Equations Complete | 3 | All equations transcribed to LaTeX |
| Figures Complete | 3 | All figures described comprehensively |
| Text Extraction | 4 | Body text captured with chunk markers |
| Formatting | 3 | Outputs follow standards |
| OCR / Agent QA | 2 | Sraffa 4.0 QA pass rate >=90% of scanned pages |
| **TOTAL** | **27** | |

**Minimum**: 22/27 (81%). **Target**: 26/27 (96%).

### Step 4: Update Progress

- Mark chunk COMPLETE in progress tracker
- Update HDARP_MASTER_CATALOG.csv (mandatory catalog sync)
- Record quality score

---

## Batch Processing Workflow

When invoked by DARP commands (/sphdarp, /phdarp, etc.):

### 1. Read BATCH_STATE.json

Determine current batch and validation queue. Commands auto-continue across all PREPARED batches unless `--single` flag is set.

### 2. Process All Chunks in Batch

For each chunk in the batch, run the single-chunk extraction workflow above. Track progress per chunk.

### 3. Catalog Sync

After all chunks complete, update HDARP_MASTER_CATALOG.csv with document-level status.

### 4. Advance to Next Batch

If more PREPARED batches exist (and `--single` not set), advance and loop back to step 1.

**Stopping conditions**: No more PREPARED batches, next batch is BLOCKED, or `--single` flag.

---

## Full Document Workflow

For processing a complete PDF end-to-end:

### 1. Analyze PDF Density

```python
from pdf_splitter_orchestrator import PDFSplitterOrchestrator
orchestrator = PDFSplitterOrchestrator()
analysis = orchestrator.analyze_pdf("path/to/document.pdf")
```

### 2. Chunk if Needed

Chunk if >10 pages OR >1MB:

```python
result = orchestrator.split_pdf(
    input_pdf="path/to/document.pdf",
    output_dir="chunks"
)
```

### 3. Process Each Chunk

Run single-chunk extraction on each chunk. Use DARP commands for parallel processing.

### 4. Assemble Final Output

After all chunks complete, the validator consolidates output into the canonical Knowledge Base:
- `FULL_TEXT.md` — assembled from all chunk body text files
- `CSV_Tables/` — all extracted tables
- `equations/` — all extracted equations
- `figures/` — all extracted figure descriptions

---

## Content Filter Retry Protocol (Scrounger L1/L2/L3 Ladder)

If content filter error occurs during chunk processing:

### L1: Bisect Chunk
Split into page-range halves, retry each independently. Recurse down to single pages.

### L2: Scholarly Framing
Retry with academic/analytical extraction framing per page. Focus on structural elements.

### L3: OCR Queue
Only after L1+L2 both fail on a specific page: log to `ocr_pending_queue` for Sraffa 4.0 OCR. Batch status stays COMPLETE — only exhausted pages go to OCR.

---

## Model Assignment (Sonnet Mandatory)

- **Processors**: Sonnet (cost-efficient, high accuracy)
- **Validator**: Opus (quality assurance requires strongest model)
- **Haiku**: BANNED — insufficient accuracy for HDARP extraction

---

## Output Structure

```
{document}
+-- chunks/
|   +-- chunk_*.pdf
+-- output/
|   +-- CSV_Tables/
|   +-- equations/
|   +-- figures/
|   +-- FULL_TEXT_chunks_*.md
+-- manifest.json
```

After validation, consolidated to Knowledge Base:
```
{document}
+-- FULL_TEXT.md
+-- CSV_Tables/
+-- equations/
+-- figures/
```

---

## Integration with DARP Commands

| Command | Smart Batching | Hybrid (OCR) | Invokes This Skill |
|---------|----------------|--------------|-------------------|
| `/pdarp N` | No | No | Yes (tables, equations, figures only) |
| `/phdarp N` | No | Yes | Yes (all 4 content types) |
| `/spdarp N` | Yes | No | Yes (tables, equations, figures only) |
| `/sphdarp N` | Yes | Yes | Yes (all 4 content types) |
| `/sphdarp-asp N` | Yes | Yes + handwriting | Yes (all 4 + handwriting) |

---

## HDARP Framework Context

- **Pipeline Stage**: 3 (EXTRACTION) — after chunking (Stage 2), before enrichment (Stage 4)
- **Upstream**: hdarp-chunker (chunks must exist before extraction)
- **Downstream**: hdarp-integrate, hdarp-integrate-pipeline (organize extracted content into KB)
- **Connection to Anu Framework**: KB output feeds anu-research at Stage 1

---

*Part of the HDARP Framework v6.3 — Core Extraction Protocol*
*Consolidated from: hdarp.md, hdarp_complete_processing.md, hdarp_process_chunk.md (May 2026)*

<!-- HDARP Framework v6.2 (2026-05-29): unified per VERSION_REGISTRY.md and HDARP_v6.2_UPGRADE_PLAN.md. Prior version stamps retained in history above. -->
